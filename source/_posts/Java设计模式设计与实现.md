---
title: Java设计模式总结与感想
date: 2018-09-23 14:45:06
tags:
- Design Patterns
categories: 
- Java
---

# 前言

从学习设计模式到现在也有一个月了，这段时间一直在看书和自己的实践来学习设计模式。通过学习设计模式，对面向对象程序的设计有不一样的感觉，可以说用了设计模式，才会感觉到面向对象设计的精髓所在，所有设计模式的代码都可以在[我的github](https://github.com/heshangbuxitou/mytest/tree/master/src/com/zy/designs)中能找到。

设计模式是什么？我们首先来看这样一个问题。官方的回答是：设计模式是一套被反复使用、多数人知晓的、经过分类编目的、代码设计经验的总结。使用设计模式是为了可重用代码、让代码更容易被他人理解、保证代码可靠性。非官方的回答则是：设计模式是对已有问题的解决方案，通过对生活中的例子通过抽象和分解，可以发现许多相似的地方，然后对相似的地方进行处理的总结，总结出一些最佳实践以及核心解决方案，这些最佳实践通过沉淀就成为了现在的设计模式。

# 设计模式和面向对象六大原则

## 简单工厂模式与开闭原则

通过对设计模式的实践和观察，可以发现设计模式主要遵循的还是面向对象代码的六大原则，可以这么说，每个设计模式都有对六大原则的最佳实践，首先我们来看看开闭原则，开闭原则可以算是软件开发中第一个被反复强调的原则了，它强调的是代码写完后的拓展，代码编写应该尽量考虑后面可能变化的需求，如果新需求有增加，我们不应该对前面写的代码进行大幅度的修改，而是应该尽量不对写好的代码进行修改，而是添加新的需求类来处理新的需求。在面向对象中存在接口和抽象类这样的语法可以很方便实践这个原则。下面通过简单工厂模式来演示开闭原则的使用。

假设我们需要写一个实现加减法的程序，如果直接通过代码写的话是这个样子：

``` java
public static void main(String[] args) {
        String numberA = "5";
        String numberB = "6";
        String operator = "-";
        double result = 0;
        if (operator.equals("+")){
            result = Double.parseDouble(numberA) + Double.parseDouble(numberB) 
        }else if(operator.equals("-")){
            result = Double.parseDouble(numberA) - Double.parseDouble(numberB)
        }
        System.out.println("结果是：" + result);
    }
```

这样的代码粗看上去好像没有什么毛病，功能也是可以实现的，但是有几个问题需要考虑下，这儿数字和操作都是独立的东西，是客户端输入的，而加法和减法的操作是程序的后台业务逻辑，我们把显示和业务逻辑分开是不是更好？第二个是现在已经有了加减法的处理，那现在我们还需要加一个乘除的操作应该怎么加？在程序中直接添加新的 `if` 分支吗？那要是还有其他的操作过来呢，又去修改原来的代码？毫无疑问这是很糟糕的操作。

我们使用简单工厂的设计模式来解决这个问题，首先需要写一个抽象 `Opertor` 类，它是所有计算类的父类，代码如下:

``` java
public abstract class Operator {
    double numberA;
    double numberB;
    //省略了属性的get和set代码，方便排版
    abstract double getResult();
}
```

定义了计算类，我们来定义它的实现类-加法类，代码如下:（减法类，乘法类，除法类与其类似，可以自行实现）

```java
public class OperatorAdd extends Operator {

    @Override
    double getResult() {
        return numberA + numberB;
    }
}
```

然后在我们的客户端程序中来使用这些类，前面提到过最好是将程序的显示和业务逻辑分离，所以 `Client` 代码中只有输入和输出的逻辑，它不负责产生计算类，那么我们使用一个工厂类来实例化计算类，把产生计算类的任务交给工厂类，工厂类的代码如下：

```java
public class Factory {
    private Factory(){}

    public static Operator createOperation(String operate){
        Operator operator;
        switch (operate){
            case "+":
                operator = new OperatorAdd();
                break;
            case "-":
                operator = new OperatorSub();
                break;
            case "*":
                operator = new OperatorMul();
                break;
            case "/":
                operator = new OperatorDiv();
                break;
            default:
                throw new RuntimeException("请输入正确的操作符号..");
        }
        return operator;
    }
}
```

然后我们在客户端可以很方便的进行对应的计算了。

```java
public class Main {
    public static void main(String[] args) {
        String numberA = "5"; //假设输入
        String numberB = "6"; //假设输入
        String operator = "*"; //假设输入

        Operator operation = Factory.createOperation(operator);
        operation.numberA = Double.parseDouble(numberA);
        operation.numberB = Double.parseDouble(numberB);
        System.out.println(operation.getResult());
    }
}
```

这样子写有什么好处呢？首先我们把每个计算的操作放到了计算类的内部，每个计算类之间不需要交互，客户端也不会知道计算类的内部是怎么计算的，这样很好的分开了显示也业务逻辑。第二呢，假设我们需要增加新的计算类，只要添加新的计算类的子类实现就行了，然后在工厂类里添加分支（假如工厂通过反射建立对象的话，这一步可以省略）就可以实现新的功能，这样的话实现的话，遵循了开闭原则，让代码的拓展看起来非常流畅。


## 里氏替换原则

再来看看里氏替换原则，里氏替换原则强调的是面向对象中的继承，它规定面向对象类的设计应该符合这样的原则，即任何父类出现的地方，子类也一定可以出现。下面用鸟会飞的例子来演示这个原则。

假设有一个鸟的类如下：

```java
public abstract class Bird
    {
        protected abstract double GetFlightSpeed();

        //飞行一段距离所需时间
        public double Fly(double distance)
        {
            return distance / GetFlightSpeed();
        }
    }
```

燕子是鸟，它的飞行速度是 100：

``` java
class Swallow extend Bird
    {
        protected double GetFlightSpeed()
        {
            return 100.0;
        }
    }
```

现在还有一个鸵鸟，它是鸟但是不会飞，所以它的速度是 0：

```java
class Ostrich extend Bird
    {
        protected double GetFlightSpeed()
        {
            return 0.0;
        }
    }
```

然后客户端调用如下:

```java
public class Main {
    public static void main(String[] args) {
            Bird bird = new Swallow();
            Double distance = 100;
            string time = bird.Fly(distance).ToString();
            System.out.println(time);
    }
}
```

上面的程序运行是没有问题的，但是如果把燕子换成鸵鸟的话就会出错，这样的类设计是违反了里氏替换原则的，正确的做法是鸟类应该分为会飞的和不会飞的鸟，然后燕子继承会飞的鸟，鸵鸟继承不会飞的鸟。这儿需要多说几句，继承滥用的情况是很常见的，因为很容易使用继承来实现人们所需要的功能，但其实继承会让类的关系变得臃肿，有些子类不需要的父类方法也会被强行加入到继承的子类中，这样子一层层继承下去，这个类势必会变得越来越难以维护，所以推荐使用继承前确认下他们是否是完全的继承关系，有句老话说的好，“优先使用组合而不是继承”。

## 单一职责原则

单一职责原则说起来是很简单的事情，即所谓引起一个类变化的原因只有一个，这个原则说起来容易，但其实由于现实中的种种因素，人们通常不会遵循这个模式。下面用一个动物行走的例子说明一下。

假设有模拟动物行走的程序如下：

``` java
public class Run{
    public void run(String animal){
        System.out.println(animal + "行走");
    }
}
public class Main {
    public static void main(String[] args) {
        Run run = new Run();
        run.run("人"); 
        run.run("狗"); 
        run.run("鱼"); 
    }
}
```

程序运行后出问题了，原来鱼是游的而不是行走的，现在需要添加动物游的需求，是直接加个类吗？显然如果加类的话修改会特别的多，不仅要增加新的类，还要修改客户端的调用代码，如果还需要增加需求的话，这可能会成为一场灾难，在方法上进行需求的添加？比如下面的代码：

``` java
public class Run{
    public void run(String animal){
        System.out.println(animal + "行走");
    }
    public void run2(String animal){
        System.out.println(animal + "游走");
    }
}
public class Main {
    public static void main(String[] args) {
        Run run = new Run();
        run.run("人"); 
        run.run("狗"); 
        run.run2("鱼"); 
    }
}
```

这样确实可以实现功能，而且做到了方法的单一职责，修改的地方也比较少，但其实这两种都是实用的，在实际使用中必须要看代码的逻辑是否足够简单，类中的方法是否足够的简单才能决定。当然，我们还是选择遵循软件的单一职责比较好。因为它带来的好处是显而易见的。
1. >可以降低类的复杂度，一个类只负责一项职责，其逻辑肯定要比负责多项职责简单的多；
2. >提高类的可读性，提高系统的可维护性；
3. >变更引起的风险降低，变更是必然的，如果单一职责原则遵守的好，当修改一个功能时，可以显著降低对其他功能的影响。

## 依赖倒置原则与抽象工厂模式

经常电脑的人可能知道，电脑硬件的组装是非常简单的，我们不需要知道主板和配件内部的细节，只需要把主板和接口对应好进行插入，电脑便能正常的工作。这是为啥呢？因为电脑的组装遵循依赖倒置原则，也就是无论主板，CPU，内存，硬盘都是针对接口设计的，不需要关注主板或者内存的实现。其映射到软件上来就是抽象不应该依赖细节，细节应该依赖于抽象，说白了，就是要针对接口编程，不要对实现编程。下面用抽象工厂模式来演示一下。

假设我们要对 `MYSQL` 数据库进行一系列操作，比如根据 `id` 获取用户,或者插入用户等等,如下面代码所示：

```java
public class AccessUserDao{
    public User getUserById(int id) {
        System.out.println("AccessUserDao 的 getUserById 方法");
        return null;
    }

    @Override
    public void insertUser(User user) {
        System.out.println("AccessUserDao 的 insertUser 方法");
    }
}

```

这样的代码是完全只针对 `MYSQL` 操作来看的，假如我们需要换 `SQL Server` 数据库是不是很难换，因为他们的 `SQL` 有相似之处，但是又有不同，修改起来非常的麻烦。此时该怎么办呢，关于这个，其实现在软件中早就已经有解决方法了，答案就是依赖倒置原则的巧妙运行，只定义业务逻辑的方法，至于底层实现，有对应的数据操作子类去实现，其代码如下：

``` java
public interface UserDao {
    User getUserById(int id);
    void insertUser(User user);
}
public class SSUserDaoImpl implements UserDao {
    @Override
    public User getUserById(int id) {
        System.out.println("SSUserDaoImpl 的 getUserById 方法");
        return null;
    }

    @Override
    public void insertUser(User user) {
        System.out.println("SSUserDaoImpl 的 insertUser 方法");
    }
}
public class AccessUserDaoImpl implements UserDao{
    @Override
    public User getUserById(int id) {
        System.out.println("AccessUserDaoImpl 的 getUserById 方法");
        return null;
    }

    @Override
    public void insertUser(User user) {
        System.out.println("AccessUserDaoImpl 的 insertUser 方法");
    }
}
```

这样就很方便的实现了了两个数据库的获取用户和查询用户的操作，而且在使用的时候我们可以使用抽象工厂来实例化对应的数据库操作类，抽象工厂如下：

``` java
public class DataAccess {
    public static Properties prop;
    static {
        prop = new Properties();
        String resousePath = DataAccess.class.getClassLoader().getResource("com/zy/designs/abstract_factory/properties.prop").getPath();
        try {
            prop.load(new FileReader(resousePath));
        } catch (IOException e) {
            e.printStackTrace();
            throw new RuntimeException(e);
        }
    }

    public static UserDao createUserDao() throws ClassNotFoundException, IllegalAccessException, InstantiationException {
        String className = prop.getProperty(UserDao.class.getSimpleName());
        Class<?> clazz = Class.forName(className);
        return (UserDao) clazz.newInstance();
    }

    public static DepartmentDao createDepartmentDao() throws ClassNotFoundException, IllegalAccessException, InstantiationException {
        String className = prop.getProperty(DepartmentDao.class.getSimpleName());
        Class<?> clazz = Class.forName(className);
        return (DepartmentDao) clazz.newInstance();
    }
}
```

使用了抽象工厂来生产数据库操作类后，我们在客户端可以很方便的调用,可以看到客户端的调用是非常简单的，而且也方便的实现了多数据库的操作，并且可以进行很方便的数据库切换。0：

``` java
public class Main {
    /**
     * 抽象工厂模式是对工厂方法模式的扩展，它在原有工厂方法模式的基础上通过一个工厂
     * 建立多个产品，这样子产品和工厂虽然比较界限分明，但是加入要增加一个新的产品，
     * 改动会非常大，所以最好采用IOC工厂进行依赖注入的方法来生产产品，下面展示一个
     * 数据库访问的例子。
     */
    public static void main(String[] args) throws IllegalAccessException, InstantiationException, ClassNotFoundException {
        DepartmentDao departmentDao = DataAccess.createDepartmentDao();
        UserDao userDao = DataAccess.createUserDao();

        departmentDao.getDepartmentById(123);
        departmentDao.insertDepartment(new Department());

        userDao.getUserById(123);
        userDao.insertUser(new User());
    }
}
```

## 接口隔离原则

接口隔离原则指的接口中的方法应该尽量的少，不要定义接口不需要的方法，一定是接口这儿必须要这儿方法才去定义方法，否则就不去定义这个方法，假设有一个接口有五个方法，一个类只需要实现其中的2个，但是由于它实现了这个方法，所有它必须对其他三个方法进行实现，这无疑造成了类的冗余，应该考虑的是把接口中的方法在进行分割，以达到符合接口隔离原则的标准。我们通过分散定义多个接口，可以预防外来变更的扩散，提高系统的灵活性和可维护性。

## 迪米特原则和外观模式

迪米特法则注重的是客户端知道的越少越好，即所谓一个对象应该对其他对象保持最少的了解。 问题由来：类与类之间的关系越密切，耦合度越大，当一个类发生改变时，对另一个类的影响也越大。越少的依赖能最大限度的保持程序内高内聚，低耦合的特性，下面通过外观模式来说明迪米特原则。

假设有三个系统类如下，我们需要先后调用系统一和系统二的方法，我们应该在客户端里面直接实例化一个系统一和系统二吗？如果这样的话假设内部调用改了，需要增加系统三的调用，那么所有的调用客户端都需要改，这个工作量是巨大的，而且这里有个问题，把这些小系统都交给客户端来调用是非常糟糕的，它会让这些系统代码和客户端耦合起来，正确的做法是应该使用外观类来调用，客户端只需要知道外观类的存在，而不需要知道其他类的存在，这样的结构非常符合迪米特法则，而且也最大限度的减少了系统类与客户端的耦合，外观模式代码实现如下：

``` java
public class SystemOne {
    public void methodOne(){
        System.out.println("SystemOne:methodOne");
    }
}
public class SystemTwo {
    public void methodTwo(){
        System.out.println("SystemTwo:methodTwo");
    }
}
public class SystemThree {
    public void methodThree(){
        System.out.println("SystemThree:methodThree");
    }
}
public class Facade {
    SystemOne one;
    SystemTwo two;
    SystemThree three;

    public Facade() {
        one = new SystemOne();
        two = new SystemTwo();
        three = new SystemThree();
    }

    public void methodA() {
        one.methodOne();
        three.methodThree();
    }

    public void methodB() {
        two.methodTwo();
    }
}
public class Main {
    public static void main(String[] args) {
        /**
         * 外观模式 是将子类系统的模块随着不断的重构演化变得
         * 越来越复杂，会有很多小的类，如果此时直接去调用的话比较
         * 复杂，而且与子系统比较的耦合，比较好的做法建立一个外观类来
         * 执行子类操作，这样子非常符合类设计原则中的迪米特法则。
         * 下面使用几个类演示外观模式。
         */
        Facade facade = new Facade();
        facade.methodA();
        facade.methodB();
    }
}
```

# 总结

在文中尽管我只介绍了六大设计原则和少数几种设计模式，但是在写代码的过程中我们会经常碰到相似的问题，如何合理的运用设计模式解决对应的问题是我们需要思考的。每学习一种设计模式都会让我感到面向对象代码的精妙之处，虽然学习完了所有的设计模式，但其实写代码的路才刚刚开始，把设计模式的思想融入到平时代码中才算是小小的入门。以上的代码和其他的设计模式实践，都可以在[我的github](https://github.com/heshangbuxitou/mytest/tree/master/src/com/zy/designs)中能找到。题外话，写完这篇文章已经是深夜一点半了，恰好是中秋节，在这这个时候，我真心祝愿我们每一个人能够通过自己的努力，成功的由菜鸟进行转变为优秀的软件工程师！