---
title: Spring Boot CORS实践
date: 2018-11-18 02:19:07
tags:
- Spring Boot
- CORS
categories:
- Java
---

# 前言

现代浏览器通过`Http`请求的方式获取服务器的响应，也可以称之为获取服务器上的资源。在浏览器的请求方式中有一个常用请求方式就是 `AJAX`，`AJAX`通过发送`XMLHttpRequest` 请求来获取服务器资源。`AJAX` 访问服务器资源非常方便，但是也存在一些限制，其中很常见的就是浏览器的同源限制。什么是同源限制呢？同源是指源网页和服务器必须要是有三个相同，即协议、域名和端口都要相同，不同源的访问下浏览器会阻止读取服务器返回的资源。



# 常用的跨域方式-CORS

CORS是现在大多数浏览器都支持的一种跨域方式，浏览器会自动使用这个方式来跨域请求资源，当服务器返回对应的CORS控制头时，浏览器会将服务器的资源`正确`的返回给客户端。

浏览器在检测到访问的资源不同源时会添加`CORS` 对应的请求头，比如`Origin` 请求头，用来表示本次请求来自哪个请求源。服务器支持`CORS` 跨域请求的话，他会对返回资源做相应的处理，比如增加一些资源访问控制的响应头。

```http
Access-Control-Allow-Origin: http://api.bob.com
Access-Control-Allow-Credentials: true
Access-Control-Expose-Headers: FooBar
Content-Type: text/html; charset=utf-8
```



* **Access-Control-Allow-Origin**  请求头表示接收的域名的请求，也可以用`*` 来表示任意域名。

* **Access-Control-Allow-Credentials** 表示浏览器是否可以传递当前网页的`Cookie` ，注意，服务器设置了可以发送`Cookie` ，客户端发送请求的时候需要设置通信对象`XHR` 的`withCredentials`  属性为`true` 。

* **Access-Control-Expose-Headers** 表示`AJAX` 请求可以获取的额外响应头，默认`AJAX` 只能获取到`Cache-Control`、`Content-Language`、`Content-Type`、`Expires`、`Last-Modified`、`Pragma` 等响应值字段。

# Spring Boot实现跨域请求

##	局部的支持

在`Spring Boot` `2.0.1`版本，`Spring MVC` 4.2 版本下，可以使用[`@CrossOrigin`](https://docs.spring.io/spring/docs/5.1.2.RELEASE/javadoc-api/org/springframework/web/bind/annotation/CrossOrigin.html) 很方便的进行局部的跨域请求配置，例如：

    @Controller
    // @CrossOrigin(origins = "http://127.0.0.1:8080", maxAge = 3600) Controller 跨域配置
    public class HelloController {
        @RequestMapping("/hello")
        @ResponseBody
        @CrossOrigin("http://127.0.0.1:8080")  方法跨域配置
        public String hello( ){
            return "Hello World";
        }
    }
## 全局的支持

在`Spring Boot` `2.0.1`版本，`Spring MVC` 4.2 版本下，可以使用可以通过使用自定义的`addCorsMappings`（`CorsRegistry`）注册`WebMvcConfigurer bean`来定义，例如:

```java
@Configuration
public class WebConfig {

    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            // cors 配置
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/**")
                        .allowedOrigins("*")
                        .allowCredentials(true)
                        .allowedMethods("GET", "POST", "DELETE", "PUT", "OPTIONS")
                        .maxAge(3600);
            }
         };
    }
}

```

`addMapping` 用来添加映射的路径，`allowedOrigins` 用来标注放行的请求域，`allowCredentials` 表示时候返回允许携带凭证访问，`allowedMethods` 表示支持的跨域方式，`maxAge` 表示预检请求的最大缓存时间。

# 实际配置中遇到的问题

在我配置如上的全局支持的`CORS` 跨域支持后，前台`AJAX`访问依旧报同源请求限制的错误，所以我直接写了一个`html` 访问项目的资源，在浏览器看到是可以正常访问服务器资源的，为什么放上服务器的前端代码无法访问项目资源呢？我一度以为自己的设置是失效的，但我把请求访问从服务器访问换到本地访问后发现是可以正常跨域的，我不禁怀疑中间漏掉了什么细节？



终于，在我四处寻找之下找到了忽略的细节，就是我的请求都是通过`Nginx` 来进行代理转发的，有没有可能是`Nginx` 把某个头改写了，导致现有`Spring MVC` 没有识别当前请求为跨域请求？下面是我的`Nginx` 的代理配置：

```latex
server
    {
       listen 9999;
       server_name _; #公网地址
    location /
    {
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://localhost:8001;
    }
    }
```

可以看到是一个很简单的`Nginx` 代理转发配置，表面上来看没有什么特别的，设置的头似乎没有什么问题，我们可以答应服务器接收到的请求头如下：

> ------- header -------
> headerKey=host;value=193.xxx.xxx.197
> headerKey=x-real-ip;value=222.xxx.xxx.144
> headerKey=x-forwarded-for;value=222.xxx.xxx.144
> headerKey=connection;value=close
> headerKey=accept;value=application/json, text/plain, */*
> headerKey=origin;value=http://193.xxx.xxx.197
> headerKey=user-agent;value=Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36
> headerKey=referer;value=http://193.xxx.xxx.197/
> headerKey=accept-encoding;value=gzip, deflate
> headerKey=accept-language;value=zh-CN,zh;q=0.9

看到这儿可能有读者已经发现问题了，这个是`Nginx` 为了隐藏自己而做的一般性操作，把 `Host`请求头改写成对应请求 去掉 **端口**的`Host` ，这样子会使的正常的请求也能拿到正确的响应，即`Nginx` 做了同源性适配，否则正常的访问都无法访问了。

了解到原因后，解决的方法就显而易见了，设置`Nginx`请求是带上`Host`的端口，新的配置如下：

```latex
server
    {
       listen 9999;
       server_name _; #公网地址
    location /
    {
        proxy_pass_header Server;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;

        proxy_pass http://127.0.0.1:9080;
    }
    }
```

# 总结

现代`Web`访问中跨源是很常见的操作，其中一个常用的跨源方式就是`CORS` 。`CORS`通过发送的请求头和返回特殊的响应头来控制客户端访问的资源权限，服务器可以自由的选择是否支持`CORS`访问以及是否允许客户端发送凭证以及设置允许获取的网页。通过`CORS`来配置跨域请求在现代服务器框架通常都有支持，在`Spring MVC`可以很方便的使用`@CrossOrigin` 配置局部跨域以及使用`WebMvcConfigurer` 配置全局的跨域。