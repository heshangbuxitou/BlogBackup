---
title: Spring Boot自动持续化集成实践
date: 2018-11-30 11:15:55
tags:
- Spring Boot
- Docker
- Jenkins
- 持续化集成
categories: 
- Java
---

# 前言

通常在开发项目后，我们会运行单元测试检验项目可用性，检查完之后打包交付给测试进行集成测试，可能中间还有`QA` 等质量检测阶段。在这些阶段中，我们会遇到很多情况，比如，项目中出现的`Bug`需要修复，有些遗漏的条件判断需要补充，产品经理临时变更需求，以及客户的需求变更等等。碰到上述这些情况，无疑我们需要再次进行开发，开发完成后又是验证，单元测试，打包交付等等一系列操作，可以看到，这些流程是非常长而且又繁琐的，手工进行这些流程难保不会错误，我们可以使用自动持续化集成打包来避免这些重复劳动又容易出错的事情。

# 项目发布的方式

通常自动化项目部署的流程是这样：

![newjicheng.png](https://i.loli.net/2018/11/30/5c00f03020571.png)

通过搭建`Jenkins`构建环境，然后在`Github`上面注册`Jenkins`的`Hook`,每次代码的提交后，`Jenkins` 会自动拉取最新的代码进行自动化的打包和发布。

# 集成构建环境

首先要做的是搭建一个自动化集成的环境，笔者用的是`Ubuntu16`  的`Linux` 系统，其他的系统也可以，建议买一个云上的环境，这样不需要自己维护，还可以`24`小时不间断进行构建。

## JDK 和Maven的安装

### JDK的安装

1. 更新软件包列表：

    ```shell
    sudo apt-get update
    ```

2. 安装openjdk-8-jdk：

    ```shell
    sudo apt-get install openjdk-8-jdk
    ```

3. 查看`Java`版本，看看是否安装成功：

    ```shell
    java -version
    ```

### Maven的安装

1. 进入下载列表：[http://www-eu.apache.org/dist/maven/maven-3/](http://www-eu.apache.org/dist/maven/maven-3/)，根据需要下载指定版本。

2. 可以下载`3.5`以上的版本，例如 [http://www-eu.apache.org/dist/maven/maven-3/3.5.4/binaries/apache-maven-3.5.4-bin.tar.gz](http://www-eu.apache.org/dist/maven/maven-3/3.5.4/binaries/apache-maven-3.5.4-bin.tar.gz) ，直接下载即可

3. 解压

    ```shell
    tar -zxvf apache-maven-3.5.4-bin.tar.gz
    ```

4. 移动

    ```shell
    mv apache-maven-3.5.4 /usr/local/maven3
    ```

5. 添加环境变量

    ```shell
    MAVEN_HOME=/usr/local/maven3
    export MAVEN_HOME
    export PATH=${PATH}:${MAVEN_HOME}/bin
    ```

6. 查看`Maven`版本，看看是否安装成功：

    ```shell
    root@VM-0-8-ubuntu:~# mvn -version
    Apache Maven 3.5.4 (1edded0938998edf8bf061f1ceb3cfdeccf443fe; 2018-06-18T02:33:14+08:00)
    Maven home: /usr/local/maven3
    Java version: 1.8.0_181, vendor: Oracle Corporation, runtime: /usr/lib/jvm/java-8-openjdk-amd64/jre
    Default locale: en_US, platform encoding: UTF-8
    OS name: "linux", version: "4.4.0-91-generic", arch: "amd64", family: "unix"
    ```

## Docker的安装

1. 安装`Docker`

    ```shell
    apt-get install docker
    ```

2. 安装`Docker-Compose`

    ```shell
    apt-get install docker-compose
    ```

3. 使用国内的镜像源

    ```shell
    Ubuntu替换国内源：打开/etc/default/docker文件（需要sudo权限），在文件的底部加上一行。

    DOCKER_OPTS="--registry-mirror=https://registry.docker-cn.com`
    然后重启Docker
    ```

4. 重启Docker，查看运行状态

    ```shell
    service restart docker
    service status docker
    ```

## Spring Boot项目集成Docker(以下示例需要替换成对应的名字)

1. 在`pom.xml`文件中添加 `Docker` 镜像名称

    ```xml
    <properties>
        <docker.image.prefix>springboot</docker.image.prefix>
    </properties>
    ```

2. 在插件中添加 `Docker` 构建插件。

    ```xml
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <!-- Docker maven plugin -->
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>1.0.0</version>
                <configuration>
                    <imageName>${docker.image.prefix}/${project.artifactId}</imageName>
                    <dockerDirectory>src/main/docker</dockerDirectory>
                    <resources>
                        <resource>
                            <targetPath>/</targetPath>
                            <directory>${project.build.directory}</directory>
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                </configuration>
            </plugin>
            <!-- Docker maven plugin -->
        </plugins>
    </build>
    ```

3. 在目录`src/main/docker`下创建 `Dockerfile` 文件，`Docker`会依靠文件构建镜像。

    ```shell
    FROM openjdk:8-jdk-alpine
    VOLUME /tmp
    ADD xxxx-0.0.1-SNAPSHOT.jar app.jar
    ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
    ```

    - `FROM` ，表示使用 `JDK1.8` 环境 为基础镜像，`Docker` 会自行从仓库拉取基础镜像。

    - `VOLUME` ，`VOLUME` 指向了一个`/tmp`的目录，由于 `Spring Boot` 使用内置的`Tomcat`容器，`Tomcat` 默认使用`/tmp`作为工作目录。这个命令的效果是：在宿主机的`/var/lib/docker`目录下创建一个临时文件并把它链接到容器中的`/tmp`目录

    - `ADD` ，拷贝文件并且重命名 。**注意，这里项目的名字需要和你的项目打包完后的名字一样**

    - `ENTRYPOINT` ，镜像运行后执行的`shell`命令。为了缩短 Tomcat 的启动时间，添加`java.security.egd`的系统属性指向`/dev/urandom`。

4. 将项目代码上传到服务器中。

    ```shell
    #打包
    mvn package
    #启动
    java -jar target/player-0.0.1-SNAPSHOT.jar
    ```

5. 启动没问题的话，接下来把项目打包成`Docker`镜像。

    ```shell
    mvn package docker:build
    # 构建成功的话会输出如下日志
    Downloaded from central: https://repo.maven.apache.org/maven2/org/eclipse/jgit/org.eclipse.jgit/3.2.0.201312181205-r/org.eclipse.jgit-3.2.0.201312181205-r.jar (1.9 MB at 29 kB/s)
    Downloaded from central: https://repo.maven.apache.org/maven2/com/spotify/docker-client/8.7.1/docker-client-8.7.1-shaded.jar (7.4 MB at 83 kB/s)
    [INFO] Using authentication suppliers: [ConfigFileRegistryAuthSupplier]
    [INFO] Copying /work/SiYuYongPlayerAPI/target/player-0.0.1-SNAPSHOT.jar -> /work/SiYuYongPlayerAPI/target/docker/player-0.0.1-SNAPSHOT.jar
    [INFO] Copying src/main/docker/Dockerfile -> /work/SiYuYongPlayerAPI/target/docker/Dockerfile
    [INFO] Building image springboot/player
    Step 1/4 : FROM openjdk:8-jdk-alpine
    Pulling from library/openjdk
    4fe2ade4980c: Pull complete
    6fc58a8d4ae4: Pull complete
    ef87ded15917: Pull complete
    Digest: sha256:b18e45570b6f59bf80c15c78d7f0daff1e18e9c19069c323613297057095fda6
    Status: Downloaded newer image for openjdk:8-jdk-alpine
    ---> 97bc1352afde
    Step 2/4 : VOLUME /tmp
    ---> Running in 492d1a502cf7
    ---> 1e029257761d.
    Removing intermediate container 492d1a502cf7
    Step 3/4 : ADD player-0.0.1-SNAPSHOT.jar app.jar
    ---> 4cea2399d2e8
    Removing intermediate container 4f3149fdc16b
    Step 4/4 : ENTRYPOINT java -Djava.security.egd=file:/dev/./urandom -jar /app.jar
    ---> Running in bb57fa700508
    ---> 058028c73f7a
    ...
    [INFO] ------------------------------------------------------------------------
    [INFO] BUILD SUCCESS
    [INFO] ------------------------------------------------------------------------
    [INFO] Total time: 54.346 s
    [INFO] Finished at: 2018-10-26T16:20:15+08:00
    [INFO] Final Memory: 42M/182M
    [INFO] ------------------------------------------------------------------------

    # 可以使用docker images 查看镜像
    root@VM-0-8-ubuntu:/work# docker images
    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
    springboot/xxxx   latest              058028c73f7a        2 weeks ago         121 MB
    ```

6. 使用 `Docker-Compose` 运行 `Docker` ，个人比较喜欢用 `Compose` 的方式运行 `Docker` ，这里可以随意。

    ```shell
    root@VM-0-8-ubuntu:/work# vi docker-compose.yml
    version: "3"
    services:
    player:
    image: springboot/player:latest
    restart: always
    ports:
    - 9080:8080
    container_name: player-container
    ```

7. 运行`Docker`服务。

    ```shell
    # 运行
    docker-compose -f docker-compose.yml up -d
    # 查看docker运行状态
    root@VM-0-8-ubuntu:/work# docker ps
    CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS              PORTS                    NAMES
    2420556842f1        springboot/xxxx:latest   "java -Djava.secur..."   2 weeks ago         Up 24 hours         0.0.0.0:9080->8080/tcp   xxxx-container
    # 可以看到docker项目已经正确的运行了
    ```

## Jenkins 安装

说实话上面的几步都是很正常的`Spring Boot`部署以及运行的操作，没有什么难度，下面`Jenkins` 的安装与整个项目的集成才是重头戏，`Jenkins`的配置和安装是会坑人的，为什么这么说呢，因为涉及到许多方面的连接，可能有一个地方配置不正确就跑不起来。读者操作一遍就知道了，不要害怕猜坑，猜坑会让你变得更强大哟。我会尽量把整个过程描述的足够细致，减少各位读者踩坑的机率。

1. 首先下载`Jenkins`的`war`包。

    ``` shell
    # jenkins版本尽量下载最新稳定的版本，不一定需要下载这个版本，因为有的插件可能会需要更高版本的Jenkins来运行
    wget http://ftp-nyc.osuosl.org/pub/jenkins/war-stable/2.138.3/jenkins.war
    ```

2. 使用Java运行Jenkins。

    ```shell
    java -jar jenkins.war
    ```

3. Jenkins第一次启动需输入管理员帐号`admin`密码，密码保存位置会有对应提示，也可以通过Jenkins运行时的输出查看。

    ```shell
    ...
    
    please use the following password to proceed to installation:
    
    xxxxxxxxxxxxxxxxxxx //密码
    
    ...
    ```

    ![](https://upload-images.jianshu.io/upload_images/2518611-d93724fadf9ab855.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

4. 输入对应的密码后点击`Continue` ，如下：

   ![](https://upload-images.jianshu.io/upload_images/2518611-47a485249b5741b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

5. 直接安装建议的插件，也可以稍后自行安装。

   ![](https://upload-images.jianshu.io/upload_images/2518611-72114d3f8f7a42c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/925/format/webp)

6. 设置对应初始账户和密码

   ![](https://upload-images.jianshu.io/upload_images/2518611-7e1e1d4a0317292e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/955/format/webp)

7. 设置完成后进入界面

   ![](https://upload-images.jianshu.io/upload_images/2518611-6a2a1d6ab190eca4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

## Github与Jenkins集成

## sercret text
> 注：此处需要一个对项目有写权限的账户

> 进入github --> setting --> Developer settingsPersonal -->Personal access tokens --> Generate new token

![](https://upload-images.jianshu.io/upload_images/2518611-6c844d8a6bb58800.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/883/format/webp)

![](https://upload-images.jianshu.io/upload_images/436630-943711ff2a74919d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/734/format/webp)

保存好这个 `token`，如果丢失，之后再也无法找到这个 `token` 。

## GitHub webhooks 设置
> 进入GitHub上指定的项目 --> setting --> WebHooks&Services --> add webhook --> 输入刚刚部署jenkins的服务器的IP

![](https://upload-images.jianshu.io/upload_images/436630-1dbb649d8ae063b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/932/format/webp)

## jenkins的github配置

* 安装GitHub Plugin

> 系统管理-->插件管理-->可选插件

直接安装`Github Plugin`, `jenkins`会自动帮你解决其他插件的依赖，直接安装该插件`Jenkins`会自动帮你安装`plain-credentials` 、`git` 、 `credentials` 、 `github-api`。

![](https://upload-images.jianshu.io/upload_images/436630-ff8c8744ed7ade0d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

### 配置GitHub Plugin

> 系统管理 --> 系统设置 --> GitHub --> Add GitHub Sever

如下图所示

![](https://upload-images.jianshu.io/upload_images/2518611-df2c88b65c841fa6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/973/format/webp)

API URL 输入 `https://api.github.com`，`Credentials`点击Add添加，`Kind`选择`Secret Text`,具体如下图所示。

![](https://upload-images.jianshu.io/upload_images/2518611-547c6e295e263296.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

设置完成后，点击`TestConnection`,提示`Credentials verified for user UUserName, rate limit: xxx`,则表明有效。

### 创建一个自由任务

- General 设置
  填写GitHub project URL, 也就是你的项目主页
  eg. `https://github.com/your_name/your_repo_name`

  ![](https://upload-images.jianshu.io/upload_images/2518611-7c250beb46759edf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

- 配置源码管理

  ![](https://upload-images.jianshu.io/upload_images/2518611-9d7836236cbf989b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

1. 填写项目的git地址, eg. `https://github.com/your_name/your_repo_name.git` 。

2. 添加`github`用户和密码。

3. 选择`githubweb`源码库浏览器，并填上你的项目URL，这样每次构建都会生成对应的`changes`，可直接链到`github`上看变更详情。

**构建触发器，构建环境** 

![](https://upload-images.jianshu.io/upload_images/2518611-9906f0e72e95a468.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

**构建**

![](https://upload-images.jianshu.io/upload_images/2518611-a84115bff915a637.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/953/format/webp)

在构建的`shell` 中可以填写你构建所需要的操作，例如：

```shell

```

**构建后操作** 

![](https://upload-images.jianshu.io/upload_images/2518611-e8678da6c93bc25c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

最后点击保存即可。现在你可以尝试`push`代码到`Github`了。

`push` 成功后自动触发构建。

![goujian.png](https://i.loli.net/2018/11/30/5c01019a7525f.png)

# 错误邮件配置

在项目构建错误后，添加构建错误邮件提醒 **(笔者添加的是`QQ`邮箱的提醒，需要提前进入邮箱中开启`POP3`和`SMTP`协议，生成第三方登录的密码)** 。先进入`Jenkins`中的系统配置中，添加如下配置。

![qqmail.png](https://i.loli.net/2018/12/01/5c022322ea87d.png)

然后在项目的配置中添加`E-mail Notification` 构建后操作，如图所示：

![projectmail.png](https://i.loli.net/2018/12/01/5c022322ea62a.png)

失败后对应的邮箱会收到失败信息的提醒，具体可以到邮箱中查看。到此，关于自动持续化配置方面的东西就结束了。

# 总结

通过自动化集成构建的方式，把开发和构建以及测试中重复且容易出错的环节统一起来，它可以让开发不用一直关注代码修改是否影响了现有测试，测试错误的话会通过邮件提醒的方式提醒开发，开发看到错误后可以立即查看是哪个地方出了问题，非常利于检查问题和修复问题，在错误一开始出现的时候就将其解决，减少项目错误排查的成本，有利于项目团队的开发与后期维护。

# 参考

* [简易的微服务持续集成方案](https://juejin.im/post/5abaf987f265da239f076a08)
* [Jenkins+Github持续集成](https://www.jianshu.com/p/b2ed4d23a3a9)
* [手把手教你搭建Jenkins+Github持续集成环境](https://www.jianshu.com/p/22b7860b4e81)
* [[自动化测试概念总结](https://www.cnblogs.com/snifferhu/p/3690001.html)](https://www.cnblogs.com/snifferhu/p/3690001.html)