- [廖雪峰的java教程](#廖雪峰的java教程)
  - [java 介绍](#java-介绍)
    - [版本](#版本)
    - [JDK JRE](#jdk-jre)
  - [java 安装目录 运行](#java-安装目录-运行)
  - [java 语法规定](#java-语法规定)
    - [this变量](#this变量)
    - [重载与覆写不同](#重载与覆写不同)
    - [classpath](#classpath)
    - [模块](#模块)
    - [反射](#反射)
  - [web开发](#web开发)
    - [servlet](#servlet)
    - [JSP 基本没用](#jsp-基本没用)
    - [MVC](#mvc)
    - [Spring](#spring)
    - [Hibernate](#hibernate)
    - [MyBatis](#mybatis)
    - [Spring MVC](#spring-mvc)
    - [Spring Boot](#spring-boot)
  - [maven](#maven)
    - [lifecycle phase goal](#lifecycle-phase-goal)
    - [插件](#插件)
    - [pom.xml](#pomxml)
    - [mvnw](#mvnw)
    - [发布](#发布)

# 廖雪峰的java教程

[source](https://www.liaoxuefeng.com/wiki/1252599548343744)

## java 介绍

Java介于编译型语言和解释型语言之间。编译型语言如C、C++，代码是直接编译成机器码执行，但是不同的平台（x86、ARM等）CPU的指令集不同，因此，需要编译出每一种平台的对应机器码。解释型语言如Python、Ruby没有这个问题，可以由解释器直接加载源码然后运行，代价是运行效率太低。

而Java是将代码编译成一种“字节码”，它类似于抽象的CPU指令，然后，针对不同平台编写虚拟机，不同平台的虚拟机负责加载字节码并执行，这样就实现了“一次编写，到处运行”的效果。当然，这是针对Java开发者而言。对于虚拟机，需要为每个平台分别开发。

为了保证不同平台、不同公司开发的虚拟机都能正确执行Java字节码，SUN公司制定了一系列的Java虚拟机规范。从实践的角度看，JVM的兼容性做得非常好，低版本的Java字节码完全可以正常运行在高版本的JVM上。

### 版本

随着Java的发展，SUN给Java又分出了三个不同版本：

Java SE：Standard Edition

Java EE：Enterprise Edition

Java ME：Micro Edition

简单来说，Java SE就是标准版，包含标准的JVM和标准库，而Java EE是企业版，它只是在Java SE的基础上加上了大量的API和库，以便方便开发Web应用、数据库、消息服务等，Java EE的应用使用的虚拟机和Java SE完全相同。

Java ME就和Java SE不同，它是一个针对嵌入式设备的“瘦身版”，Java SE的标准库无法在Java ME上使用，Java ME的虚拟机也是“瘦身版”。

毫无疑问，Java SE是整个Java平台的核心，而Java EE是进一步学习Web应用所必须的。我们熟悉的Spring等框架都是Java EE开源生态系统的一部分。不幸的是，Java ME从来没有真正流行起来，反而是Android开发成为了移动平台的标准之一，因此，没有特殊需求，不建议学习Java ME。

### JDK JRE
初学者学Java，经常听到JDK、JRE这些名词，它们到底是啥？

JDK：Java Development Kit
JRE：Java Runtime Environment
简单地说，JRE就是运行Java字节码的虚拟机。但是，如果只有Java源码，要编译成Java字节码，就需要JDK，因为JDK除了包含JRE，还提供了编译器、调试器等开发工具。

二者关系如下：
```
  ┌─    ┌──────────────────────────────────┐
  │     │     Compiler, debugger, etc.     │
  │     └──────────────────────────────────┘
 JDK ┌─ ┌──────────────────────────────────┐
  │  │  │                                  │
  │ JRE │      JVM + Runtime Library       │
  │  │  │                                  │
  └─ └─ └──────────────────────────────────┘
        ┌───────┐┌───────┐┌───────┐┌───────┐
        │Windows││ Linux ││ macOS ││others │
        └───────┘└───────┘└───────┘└───────┘
```
那JSR、JCP……又是啥？

JSR规范：Java Specification Request
JCP组织：Java Community Process
为了保证Java语言的规范性，SUN公司搞了一个JSR规范，凡是想给Java平台加一个功能，比如说访问数据库的功能，大家要先创建一个JSR规范，定义好接口，这样，各个数据库厂商都按照规范写出Java驱动程序，开发者就不用担心自己写的数据库代码在MySQL上能跑，却不能跑在PostgreSQL上。

所以JSR是一系列的规范，从JVM的内存模型到Web程序接口，全部都标准化了。而负责审核JSR的组织就是JCP。

## java 安装目录 运行
细心的童鞋还可以在JAVA_HOME的bin目录下找到很多可执行文件：

java：这个可执行程序其实就是JVM，运行Java程序，就是启动JVM，然后让JVM执行指定的编译后的代码；
javac：这是Java的编译器，它用于把Java源码文件（以.java后缀结尾）编译为Java字节码文件（以.class后缀结尾）；
jar：用于把一组.class文件打包成一个.jar文件，便于发布；
javadoc：用于从Java源码中自动提取注释并生成文档；
jdb：Java调试器，用于开发阶段的运行调试。

Java源码本质上是一个文本文件，我们需要先用javac把Hello.java编译成字节码文件Hello.class，然后，用java命令执行这个字节码文件：
```
┌──────────────────┐
│    Hello.java    │◀── source code
└──────────────────┘
          │ compile
          ▼
┌──────────────────┐
│   Hello.class    │◀── byte code
└──────────────────┘
          │ execute
          ▼
┌──────────────────┐
│    Run on JVM    │
└──────────────────┘
```
因此，可执行文件javac是编译器，而可执行文件java就是虚拟机。

## java 语法规定
Java规定，某个类定义的public static void main(String[] args)是Java程序的固定入口方法，因此，Java程序总是从main方法开始执行。
最后，当我们把代码保存为文件时，文件名必须是Hello.java，而且文件名也要注意大小写，因为要和我们定义的类名Hello完全保持一致。

### this变量
在方法内部，可以使用一个隐含的变量this，它始终指向当前实例。因此，通过this.field就可以访问当前实例的字段。

如果没有命名冲突，可以省略this。

### 重载与覆写不同

Override和Overload不同的是，如果方法签名不同，就是Overload，Overload方法是一个新方法；如果方法签名相同，并且返回值也相同，就是Override。

### classpath

classpath是JVM用到的一个环境变量，**它用来指示JVM如何搜索class**。

因为Java是编译型语言，源码文件是.java，而编译后的.class文件才是真正可以被`JVM执行的字节码`。因此，JVM需要知道，如果要加载一个abc.xyz.Hello的类，应该去哪搜索对应的Hello.class文件。

所以，classpath就是一组目录的集合，它设置的搜索路径与操作系统相关。

classpath的设定方法有两种：

在系统环境变量中设置classpath环境变量，不推荐；

在启动JVM时设置classpath变量，推荐。

我们强烈不推荐在系统环境变量中设置classpath，那样会污染整个系统环境。在启动JVM时设置classpath才是推荐的做法。实际上就是给java命令传入-classpath或-cp参数

>不要把任何Java核心库添加到classpath中！JVM根本不依赖classpath加载核心库！

jar包
如果有很多.class文件，散落在各层目录中，肯定不便于管理。如果能把目录打一个包，变成一个文件，就方便多了。

jar包就是用来干这个事的，它可以把package组织的目录层级，以及各个目录下的所有文件（包括.class文件和其他文件）都打成一个jar文件，这样一来，无论是备份，还是发给客户，就简单多了。

jar包还可以包含一个特殊的/META-INF/MANIFEST.MF文件，MANIFEST.MF是纯文本，可以指定Main-Class和其它信息。JVM会自动读取这个MANIFEST.MF文件，如果存在Main-Class，我们就不必在命令行指定启动的类名，而是用更方便的命令：

java -jar hello.jar
在大型项目中，不可能手动编写MANIFEST.MF文件，再手动创建zip包。Java社区提供了大量的开源构建工具，例如Maven，可以非常方便地创建jar包。

类似的，对应到JVM的class文件，我们也可以用Java 17编译一个Java程序，指定输出的class版本要兼容Java 11（即class版本55），这样编译生成的class文件就可以在Java >=11的环境中运行。

指定编译输出有两种方式，一种是在javac命令行中用参数--release设置：

 javac --release 11 Main.java
参数--release 11表示源码兼容Java 11，编译的class输出版本为Java 11兼容，即class版本55。

第二种方式是用参数--source指定源码版本，用参数--target指定输出class版本：

 javac --source 9 --target 11 Main.java

### 模块

在Java 9之前，一个大型Java程序会生成自己的jar文件，同时引用依赖的第三方jar文件，而JVM自带的Java标准库，实际上也是以jar文件形式存放的，这个文件叫rt.jar，一共有60多M。

如果是自己开发的程序，除了一个自己的app.jar以外，还需要一堆第三方的jar包，运行一个Java程序，一般来说，命令行写这样：

java -cp app.jar:a.jar:b.jar:c.jar com.liaoxuefeng.sample.Main

从Java 9开始引入的模块，主要是为了解决“依赖”这个问题。如果a.jar必须依赖另一个b.jar才能运行，那我们应该给a.jar加点说明啥的，让程序在编译和运行的时候能自动定位到b.jar，这种自带“依赖关系”的class容器就是模块。

那么，我们应该如何编写模块呢？还是以具体的例子来说。首先，创建模块和原有的创建Java项目是完全一样的，以oop-module工程为例，它的目录结构如下：
```
oop-module
├── bin
├── build.sh
└── src
    ├── com
    │   └── itranswarp
    │       └── sample
    │           ├── Greeting.java
    │           └── Main.java
    └── module-info.java
```
其中，bin目录存放编译后的class文件，src目录存放源码，按包名的目录结构存放，仅仅在src目录下多了一个module-info.java这个文件，这就是模块的描述文件。

### 反射

反射就是Reflection，Java的反射是指程序在运行期可以拿到一个对象的所有信息。

由于JVM为每个加载的class创建了对应的Class实例，并在实例中保存了该class的所有信息，包括类名、包名、父类、实现的接口、所有方法、字段等，因此，如果获取了某个Class实例，我们就可以通过这个Class实例获取到该实例对应的class的所有信息。

这种通过Class实例获取class信息的方法称为反射（Reflection）。

如何获取一个class的Class实例？有三个方法：

方法一：直接通过一个class的静态变量class获取：

Class cls = String.class;
方法二：如果我们有一个实例变量，可以通过该实例变量提供的getClass()方法获取：

String s = "Hello";
Class cls = s.getClass();
方法三：如果知道一个class的完整类名，可以通过静态方法Class.forName()获取：

Class cls = Class.forName("java.lang.String");

注意到数组（例如String[]）也是一种类，而且不同于String.class，它的类名是Ljava.lang.String;。此外，JVM为每一种基本类型如int也创建了Class实例，通过int.class访问。

## web开发

### servlet

因此，在JavaEE平台上，处理TCP连接，解析HTTP协议这些底层工作统统扔给现成的`Web服务器`去做，我们只需要把自己的应用程序跑在Web服务器上。为了实现这一目的，JavaEE提供了Servlet API，我们使用Servlet API编写自己的Servlet来处理HTTP请求，Web服务器实现Servlet API接口，实现底层功能：
```
                 ┌───────────┐
                 │My Servlet │
                 ├───────────┤
                 │Servlet API│
┌───────┐  HTTP  ├───────────┤
│Browser│<──────>│Web Server │
└───────┘        └───────────┘
```
普通的Java程序是通过启动JVM，然后执行main()方法开始运行。但是Web应用程序有所不同，我们无法直接运行`war文件`，必须先启动Web服务器，再由Web服务器加载我们编写的HelloServlet，这样就可以让HelloServlet处理浏览器发送的请求。

因此，我们首先要找一个支持`Servlet API`的Web服务器。常用的服务器有：

Tomcat：由Apache开发的开源免费服务器；
Jetty：由Eclipse开发的开源免费服务器；
GlassFish：一个开源的全功能JavaEE服务器。

因为我们编写的Servlet并不是直接运行，而是由Web服务器加载后创建实例运行，所以，类似Tomcat这样的Web服务器也称为`Servlet容器`。

### JSP 基本没用

用PrintWriter输出HTML比较痛苦，因为不但要正确编写HTML，还需要插入各种变量。如果想在Servlet中输出一个类似新浪首页的HTML，写对HTML基本上不太可能。

那有没有更简单的输出HTML的办法？

有！
我们可以使用`JSP`。

JSP是`Java Server Pages`的缩写，它的文件必须放到/src/main/webapp下，文件名必须以.jsp结尾，整个文件与HTML并无太大区别，但需要插入变量，或者动态输出的地方，使用特殊指令`<% ... %>`。

### MVC

Servlet适合编写Java代码，实现各种复杂的业务逻辑，但不适合输出复杂的HTML；
JSP适合编写HTML，并在其中插入动态内容，但不适合编写复杂的Java代码。

两者结合

### Spring

随着Spring越来越受欢迎，在Spring Framework基础上，又诞生了Spring Boot、Spring Cloud、Spring Data、Spring Security等一系列基于Spring Framework的项目

### Hibernate

这种把关系数据库的表记录映射为Java对象的过程就是`ORM`：Object-Relational Mapping。ORM既可以把记录转换成Java对象，也可以把Java对象转换为行记录。

使用JdbcTemplate配合RowMapper可以看作是最原始的ORM。如果要实现更自动化的ORM，可以选择成熟的`ORM框架`，例如Hibernate。

### MyBatis

使用MyBatis最大的问题是所有SQL都需要`全部手写`，优点是执行的SQL就是我们自己写的SQL，对SQL进行优化非常简单，也可以编写任意复杂的SQL，或者使用数据库的特定语法，但切换数据库可能就不太容易。好消息是大部分项目并没有切换数据库的需求，完全可以针对某个数据库编写尽可能优化的SQL。

MyBatis是一个半自动化的`ORM框架`，需要手写SQL语句，没有自动加载一对多或多对一关系的功能。

### Spring MVC

直接使用Servlet进行Web开发好比直接在JDBC上操作数据库，比较繁琐，更好的方法是在Servlet基础上封装MVC框架，基于MVC开发Web应用，大部分时候，不需要接触Servlet API，开发省时省力。

我们在MVC开发和MVC高级开发已经由浅入深地介绍了如何编写MVC框架。当然，自己写的MVC主要是理解原理，要实现一个功能全面的MVC需要大量的工作以及广泛的测试。

因此，开发Web应用，首先要选择一个优秀的MVC框架。常用的MVC框架有：

Struts：最古老的一个MVC框架，目前版本是2，和1.x有很大的区别；
WebWork：一个比Struts设计更优秀的MVC框架，但不知道出于什么原因，从2.0开始把自己的代码全部塞给Struts 2了；
Turbine：一个重度使用Velocity，强调布局的MVC框架；
其他100+MVC框架……（略）
Spring虽然都可以集成任何Web框架，但是，Spring本身也开发了一个MVC框架，就叫Spring MVC。这个MVC框架设计得足够优秀以至于我们已经不想再费劲去集成类似Struts这样的框架了。

我们在前面介绍Web开发时已经讲过了`Java Web`的基础：Servlet容器，以及标准的Servlet组件：

Servlet：能处理HTTP请求并将HTTP响应返回；
JSP：一种嵌套Java代码的HTML，将被编译为Servlet；
Filter：能过滤指定的URL以实现拦截功能；
Listener：监听指定的事件，如ServletContext、HttpSession的创建和销毁。

### Spring Boot

我们已经在前面详细介绍了Spring框架，它的主要功能包括IoC容器、AOP支持、事务支持、MVC开发以及强大的第三方集成功能等。

那么，Spring Boot又是什么？它和Spring是什么关系？

Spring Boot是一个基于Spring的套件，它帮我们`预组装`了Spring的一系列组件，以便以尽可能少的代码和配置来开发基于Spring的Java应用程序。

## maven

Maven是一个Java项目的管理和构建工具：

Maven使用pom.xml定义项目内容，并使用预设的目录结构；
在Maven中声明一个依赖项可以自动下载并导入classpath；
Maven使用groupId，artifactId和version唯一定位一个依赖。

Maven通过解析依赖关系确定项目所需的jar包，常用的4种scope有：compile（默认），test，runtime和provided；

Maven从中央仓库下载所需的jar包并缓存在本地；

可以通过镜像仓库加速下载。

### lifecycle phase goal

在实际开发过程中，经常使用的命令有：

mvn clean：清理所有生成的class和jar；

mvn clean compile：先清理，再执行到compile；

mvn clean test：先清理，再执行到test，因为执行test前必须执行compile，所以这里不必指定compile；

mvn clean package：先清理，再执行到package。

大多数phase在执行过程中，因为我们通常没有在pom.xml中配置相关的设置，所以这些phase什么事情都不做。

经常用到的phase其实只有几个：

clean：清理
compile：编译
test：运行测试
package：打包

lifecycle相当于Java的package，它包含一个或多个phase；

phase相当于Java的class，它包含一个或多个goal；

goal相当于class的method，它其实才是真正干活的。

### 插件

我们以compile这个phase为例，如果执行：

mvn compile
Maven将执行compile这个phase，这个phase会调用compiler插件执行关联的compiler:compile这个goal。

实际上，执行每个phase，都是通过某个插件（plugin）来执行的，Maven本身其实并不知道如何执行compile，它只是负责找到对应的compiler插件，然后执行默认的compiler:compile这个goal来完成编译。

所以，使用Maven，实际上就是配置好需要使用的插件，然后通过phase调用它们。

### pom.xml

Maven支持模块化管理，可以把一个大项目拆成几个模块：

可以通过继承在parent的pom.xml统一定义重复配置；
可以通过`<modules>`编译多个模块。

### mvnw

使用Maven Wrapper，可以为一个项目指定特定的Maven版本。

### 发布

使用Maven发布一个Artifact时：

可以发布到本地，然后推送到远程Git库，由静态服务器提供基于网页的repo服务，使用方必须声明repo地址；
可以发布到central.sonatype.org，并自动同步到Maven中央仓库，需要前期申请账号以及本地配置；
可以发布到GitHub Packages作为私有仓库使用，必须提供Token以及正确的权限才能发布和使用。