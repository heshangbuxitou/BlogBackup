---
title: Spring注解事务底层实践
date: 2018-04-30 14:41:49
tags:
- Servlet
- Spring
- AOP
- annotation
categories: 
- Java
---
# 了解AOP

AOP （ Aspect Orient Programming ） 我们一般称其为面向切面编程，它能够帮助我们模块化横切关注点，试想一下我们有个购买物品的 Service 类，它有自己购买物品的业务逻辑，假如我们要在购买物品前后记录一些日志应该怎么做，在业务中编写添加日志的代码吗？那么每个地方都要有记录日志的代码，这样的代码显得很臃肿，也许你说可以把日志的管理统一由一个类或者一个管理器来进行管理，这样还是免不了要在业务代码中调用管理器的方法，会造成业务逻辑代码与日志管理器的耦合。怎么解决这种耦合呢？答案就是使用切面。

在软件中重用通用功能的话，最常见的功能是继承或者委托，如果在整个应用中都使用相同的基类，继承往往会导致一个脆弱的对象体系，使用委托可能对委托对象进行过于复杂的调用。切面在这里提供了另外一种解决方案，它可以让业务只关系自己的业务逻辑，不需要关注那些横跨整个应用的服务，甚至都不知道他们的存在，因为切面只需要声明这个功能要以何种方法在何处调用，而无需修改受影响的类。

![AspectInService.jpg](https://i.loli.net/2018/04/30/5ae6e12d7040a.jpg)

# Spring AOP动态代理的两种方法

Spring 的 AOP 的底层实现主要是动态代理，动态代理底层会动态生成一个类，它包含了目标接口的所有方法，并且可以根据开发人员的意愿，对特定的切点进行增强，并且回调原有对象的方法。

Spring AOP动主要用到的动态代理方式有 JDK 动态代理何 CGLIB 动态代理，当代理的类有实现接口的话就会使用 JDK 动态代理，如果目标类没有实现接口，那么 Spring AOP 会选择使用 CGLIB 来动态代理目标类。这里还有一点需要注意的是，被代理的类不能是 final 类，因为 CGLIB 是通过继承的方式做的动态代理，而 final 类不可以被继承，因此 CGLIB 会抛出错误。

# 使用注解开启事务

Spring 支持声明式事务，它用注解来选择需要使用事务的方法，@Transactional注解在方法上表明该方法需要事务支持。我们来仿照 Spring 实现一个功能相同的注解，假设这个注解的名字为 Tran，就像下面这样。

``` java
@Tran
    void registUser(User user) throws MsgException;
```

## 使用代理Service

在 Spring 中，Service 的生产与代理都是由 Spring 负责的，这次实践没有使用 Spring，只是实现 Spring 声明式事务的功能，因此我们使用一个工厂类来创建 Service 实例。

``` java
public <T> T getService(Class<T> clazz) {

        try {
            String simpleName = clazz.getSimpleName();
            String className = prop.getProperty(simpleName);
            final T service = (T) Class.forName(className).newInstance();

            T proxyService = (T) Proxy.newProxyInstance(service.getClass().getClassLoader(), service.getClass().getInterfaces(), new InvocationHandler() {
                @Override
                public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                    if (method.isAnnotationPresent(Tran.class)) {
                        try {
                            // 开启事务
                            TransactionManager.startTran();
                            Object obj = method.invoke(service, args);
                            // 提交事务
                            TransactionManager.commit();
                            return obj;
                        } catch (InvocationTargetException e) {
                            // 事务回滚
                            TransactionManager.rollback();
                            throw new RuntimeException(e.getTargetException());
                        } catch (Exception e) {
                            TransactionManager.rollback();
                            throw new RuntimeException(e);
                        } finally {
                            // 释放资源
                            TransactionManager.release();
                        }
                    }
                    return method.invoke(service, args);
                }

            });

            return proxyService;
        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException(e);
        }

    }
```

如你所见，当我们发现 @Tran 存在于方法上的时候，将会开启事务，当方法正常的执行完的时候，它与其他的方法显示的结果是一样的，但是如果它没有正确的完成，比如抛出了空指针异常的话，它前面所有执行的操作都会回滚，从而保证数据操作不会出错。

## TransactionManager实现

我们来看 TransactionManager 对事务的管理，可以看到它使用 `isTranLocal` 来标注事务是否开启，`realConn`来保存真实连接，`proxyConn`来保存代理连接。

``` java
    private static ThreadLocal<Boolean> isTranLocal = new ThreadLocal<Boolean>() {
        @Override
        protected Boolean initialValue() {
            return false;
        }
    };
    private static ThreadLocal<Connection> realConn = new ThreadLocal<Connection>();
    private static ThreadLocal<Connection> proxyConn = new ThreadLocal<Connection>();
```

在来看看事务的开启是什么样的：

``` java
    public static void startTran() throws SQLException {
        isTranLocal.set(true);
        final Connection conn = dataSource.getConnection();
        conn.setAutoCommit(false);
        realConn.set(conn);

        final Connection proxyC = (Connection) Proxy.newProxyInstance(conn.getClass().getClassLoader(), conn.getClass().getInterfaces(), new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                if (method.getName().equals("close")) {
                    return null;
                } else {
                    return method.invoke(conn, args);
                }
            }
        });
        proxyConn.set(proxyC);

    }
```

代码里面已经很清楚了，它把真实的连接隐藏了，并且修改了代理类的 close 方法，让连接无法真正的关闭，当然我们还需要保证对操作数据库的是同一个连接，因此，当开启事务时，应该返回的是一个 **数据源** 的代理：

``` java
    public static DataSource getDataSource() {
        if (isTranLocal.get()) {
            return (DataSource) Proxy.newProxyInstance(dataSource.getClass().getClassLoader(), dataSource.getClass().getInterfaces(), new InvocationHandler() {
                @Override
                public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                    if (method.getName().equals("getConnection")) {
                        return proxyConn.get();
                    }else {
                        return method.invoke(dataSource, args);
                    }
                }
            });
        }else {
            return dataSource;
        }
    }
```

# 测试

我们在 `UserService` 的 `registUser` 添加 @Tran 注解，并且故意让它抛出空指针异常，然后观察数据库的变化。

``` java
    public void registUser(User user) throws MsgException {
        if(dao.findUserByUserName(user.getUsername()) != null){
            throw new MsgException("用户名已经存在！");
        }
        dao.addUser(user);
        dao.addUser(null);
    }
```

为了简便，我们编写一个测试类对其进行测试：

``` java
    public void testDoPostHttpServletRequestHttpServletResponse() throws ServletException, IOException {

        expect(request.getParameter("valistr")).andReturn(paramMap.get("valistr"));
        expect(request.getParameterMap()).andReturn(paramMap);
        request.setCharacterEncoding("utf-8");
        expectLastCall();
        expect(request.getContextPath()).andReturn("http://127.0.0.1");
        expect(request.getSession()).andReturn(session).times(2);

        response.setContentType("text/html;charset=utf-8");
        expectLastCall();
        expect(response.getWriter()).andReturn((PrintWriter) writer);
        response.setHeader("refresh", "3;url=http://127.0.0.1/index.jsp");

        expect(session.getAttribute("valistr")).andReturn("valistr");
        session.setAttribute(anyObject(String.class), anyObject(User.class));
        expectLastCall();


        replay(request, response, session);                    //回放

        servlet.doPost(request, response);  //调用
    }
```

可以看到事务的确是被开启了，然后再去数据库进行验证，发现确实没有新插入的记录，因此可以看到我们前面提交的操作确实被回滚了， @Tran
确实实现了自动开启事务的功能。

``` java
信息: Initializing c3p0-0.9.1.2 [built 21-May-2007 15:04:56; debug? true; trace: 10]
开始事务。。。
四月 30, 2018 5:08:03 下午 com.mchange.v2.c3p0.impl.AbstractPoolBackedDataSource getPoolManager
信息: Initializing c3p0 pool... com.mchange.v2.c3p0.ComboPooledDataSource [ acquireIncrement -> 3, acq...
java.lang.NullPointerException
	at com.zy.dao.UserDaoImpl.addUser(UserDaoImpl.java:28)
    at com.zy.service.UserServiceImpl.registUser(UserServiceImpl.java:18)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.refn.junit.JUnitStarter.main(JUnitStarter.java:70)...
```

# 小结

我们实现了使用注解来声明事务，当系统中的服务需要在各个地方使用时，我们使用了 AOP 而不是修改原有代码的方法来实现，这样很好的保证了业务逻辑不与其他的服务相耦合，可以把分散在应用各处的行为放入可重用的模块中，我们只需要申明在何处如何应用该行为，这有效的减少了代码的冗余，并且让业务类只需要关注自身的主要功能。

# 参考资料

1. [Spring in Action](https://book.douban.com/subject/26767354/)
2. [Spring AOP的实现原理](http://www.importnew.com/24305.html)