---
layout: post
title: "Java 9 模块化入门"
date: 2017-11-15 18:09:10 +0800
comments: true
categories: [java, jigsaw]
published: true
---
### 背景
本篇文章，我们将了解下 Java 9 带给我们的新特性—— Java 平台模块化系统(JPMS, Java Platform Module System)，项目代号为 [Jigsaw](http://openjdk.java.net/projects/jigsaw/)。我们都知道 Java 自 1995 年发布以来已经在上亿的设备上运行过，无论是体积庞大的大型机服务器还是只有手掌大小的嵌入式设备都能看到 Java 的身影，而随着 Java 平台的不断演进，Java 代码库也越发的庞大和臃肿，这样就很难在资源非常有限的 IoT 设备和嵌入式设备上部署 Java 应用，因此对 Java 平台进行模块化也越发的成为 Java 未来发展的首要任务之一。下面我们先看一下它的诞生历程：

Java 平台模块化从提出到出现在我们面前，可谓命途多舛，早在 2005 年 Java 7 时代，就以 [JSR 277: Java Module System](https://jcp.org/en/jsr/detail?id=277) 规范的形式提出了，号称要在 Java 8 中出现，给 Java 带来历史上可以与 Java 5 相媲美的一次重要里程碑式的更新，之后又被 [JSR 376: Java Platform Module System](https://jcp.org/en/jsr/detail?id=376) 规范所取代，可是由于该项目进度缓慢，困难重重，导致了 Java 8 发布时间一拖再拖，最终等不起了只好先把 Jigsaw 功能又延期到 Java 9，先将 Java 8 发布出来供大家使用，虽然没有 Jigsaw 特性，但 Java 8 也是一款非常重量级的更新，它带来了 Lambda、Stream 和 [改进的日期时间 API](https://mingzhe.org/blog/2016/12/25/java8-new-date-api) 等功能特性，让程序员享受到函数式和流式编程的乐趣。而到了后 Java 8 时代，眼看 Java 9 的发布截止日期马上就要到了，可是 Jigsaw 项目仍然被许多问题困扰，如设计上的问题，过多的 bug 还没解决，因此又延期了，最终 Java 平台首席架构师 [Mark Reinhold](https://mreinhold.org) 下了最后通牒，要求 Java 9 必须在 2017 年 9 月份发布，在还有半年就要发布前，Jigsaw 项目发布了一版 Public Review 版，但是遭到了IBM 和 Redhat 的反对，遭到反对的原因在于他们认为模块化破坏了现有系统的设计原则，会导致现有系统迁移到新的模块化系统上的开销会非常巨大(其实是为了各自的利益所考虑的，这两家公司都有自己的中间件，而为了遵循统一的标准可能改动会比较大)，具体信息请看这篇文章 [Concerns Regarding Jigsaw](https://developer.jboss.org/blogs/scott.stark/2017/04/14/critical-deficiencies-in-jigsawjsr-376-java-platform-module-system-ec-member-concerns)。在与这两家经过协商和妥协后，Java 模块化系统终于在 2017 年 09 月 21 日[发布](https://mreinhold.org/blog/jigsaw-complete)了，模块化在经历了多重困难后终于与我们相见。下面我们将了解一些最基本的概念，实验环境：Java 9.0.1。

### 模块是什么
我们都知道在 Java 9 之前代码都是按包进行组织的，而**模块则是在包的概念上又增加了一个更高层的抽象，它将多个逻辑上、功能上相关的包以及相关的资源文件，如 xml 文件等组织成一个可重用的逻辑单元，这个逻辑单元就称为模块**。通常情况下一个模块包含着一个**模块描述符文件(module descriptor)用来指定模块的名字、依赖(需要注意的是每个模块必须显式得声明其依赖)、对外可使用的包(其余的包则对外不可见)、模块提供的服务、模块使用的服务以及允许哪些模块可以对其进行反射等配置信息**。模块最终都会打包成 jar 包来分发和使用的，那么模块打包的 jar 包和普通的 jar 包有什么区别呢？模块和普通的 jar 包几乎是一样的，只不过比普通的 jar 包在根目录下多了一个模块描述符文件—— module-info.class而已。

### 模块的类型
Java 9 的模块可以分为 3 种类型

#### 1.具名模块(Named Module)
具名模块也称为应用模块(Application Module)，通常在模块根目录下有 module-info.java 文件的话，那么该模块就称为具名模块，我们编写的模块一般都属于这种类型，还有很多第三方的依赖也可以归类于此，只要第三方依赖的维护者将库迁移到 Java 9 的模块即可，那么我们就可以在自己的模块中引用这些依赖。

#### 2.无名模块(Unnamed Module)
无名模块指的就是不包含 module-info.java 的 jar 包，通常这些 jar 包都是 Java 9 之前构建的。无名模块可以读取到其他所有的模块，并且也会将自己包下的所有类都暴露给外界。需要注意的是无名模块导出了所有的包，但并不意味着任何具名的模块可以读取无名模块，因为具名模块在 module-info.java 中无法声明对无名模块的引用，无名模块导出所有包的目的在于让其他无名模块可以加载这些类。但是无名模块存在一个问题，假如我们需要依赖某个第三方的构件，但这个依赖还没有迁移到 Java 9 模块化，那么我们就无法引用其中的类，进而就无法编写应用了，难道我们要一直等到依赖迁移完成才能使用吗？请看下面的自动模块。

#### 3.自动模块(Automatic Module)
任何无名模块(没有 module-info.java 的模块)放到模块路径(module path)上会自动变为自动模块，允许 Java 9 模块能引用到这些模块中的类。**自动模块对外暴露所有的包并且能引用到其他所有模块的类，其他模块也能引用到自动模块的类**。由于自动模块并不能声明模块名，那么 JDK 会根据 jar 包名来自动生成一个模块名以允许其他模块来引用。生成的模块名按以下规则生成：首先会移除文件扩展名以及版本号，然后使用"."替换所有非字母字符。例如 `spring-core-4.3.12.jar` 生成的模块名为 `spring.core`，那么其他模块就可以通过 `requires spring.core` 来使用其中的类。

#### 

<!--more-->

### 模块化的目标
根据 JSR 376 规范的描述，模块化系统要实现以下几个目标：

* **可靠的配置**。模块应该提供一种机制用来显式声明模块间的依赖问题，而从系统可以寻着依赖路径来提取出所有模块的一个子集来支撑系统的运行。

* **强封装**。模块化机制要求模块中的包只有在显式地导出后才可以被其他模块所使用，并且其他的模块必须显式地声明它需要这个模块的包后才能使用这些包，也就是说需要两个模块同时声明导出的包和使用的包时，这些包(packages)才能被使用。这种机制可以提高安全性，攻击者能够访问的类越少越安全。此外，模块化也有助于我们思考如何组织代码，获得更简洁、更合理的设计。

* **可扩展的平台**。之前 Java 平台是作为一个整体部署到操作系统之上的，其中包含了不计其数的包和类，无论我们是否使用，它们都在那里，如果某些类需要更新或打补丁，则需要替换整个 Java 平台，难以维护和扩展。而 Java 9 之后将平台划分成了 95 个模块，**通过使用模块化技术，我们可以创建定制的运行环境，只包含硬件和应用需要的包和类**即可。例如，设备不支持图形界面的话，那么我们就可以创建一个不包含 GUI 模块的运行环境，那么这将极大的降低运行环境的大小，节省不少空间。

### 模块声明
Java 9 将 JDK 划分成了多个模块可供我们随意组合，在安装完 JDK 9 之后我们可以通过 `java --list-modules` 命令来列出所有的模块，这些模块都位于 `$JAVA_HOME/jmods` 目录下，运行结果如下所示：
```java
java.activation@9.0.1
java.base@9.0.1
……
javafx.base@9.0.1
javafx.swing@9.0.1
……
jdk.attach@9.0.1
jdk.deply@9.0.1
……
oracle.desktop@9.0.1
oracle.net@9.0.1
```

从结果中可以看出所有的模块被划分成 4 大类，一类是以 java 开头的模块，表明这些模块是 Java SE 规范的实现；以 javafx 开头的模块表明这是与图形界面编程技术 JavaFX 相关的模块；以 jdk 开头的模块是与 JDK 相关的;以及最后以 oracle 开头的模块表明这是 Oracle 厂商相关的模块。每个模块后面都跟着以 `@9.0.1` 字符串结尾，表明该模块属于 Java 9.0.1 版本的。

我们上面提到过，每个模块都必须有一个**模块描述符(module descriptor)**文件，也叫模块声明文件，它记录了模块的一些元数据信息，如模块的名字、依赖和暴露给外界的包等信息。这个文件是通过在模块的根目录下定义一个名为 **module-info.java** 文件来实现的，当我们编译模块后就会在模块的根目录下生成一个 module-info.class 文件，模块的信息就存放在其中。每个模块声明都是以关键字 **module** 开头，后接一个唯一的模块名字和一对用于包含模块声明体的花括号，示例如下：

```java
module moduleName {

}
```
模块的声明体可以为空，也可以包含多条指令，如 requires, exports, provides...with, uses, opens, to, transitive。下面我们将分别了解下这几条指令的意义，需要注意的是这些**模块指令都属于受限关键字，只在模块声明体中是关键字，但是在代码中可以随意使用**。

### 模块指令

1. **requires 指令**。该指令用于指定当前模块的依赖模块，**每个模块必须显式声明依赖的模块**。如果模块 A requires 模块 B，那么我们就说模块 A 依赖模块 B，语法如下：

```java
requires moduleName;
```

requires 指令有好多衍生用法，如 requires static，表示依赖的模块是编译时是必需的，但运行时是可选的，因此这种依赖也称为可选依赖，本文不做探讨。还有一种语法是 requires transitive，假如模块 A 依赖模块 B，外界的模块不仅想使用 A 模块还想使用 B 模块，那么就使用该指令来获取 B 的访问权限，这种对外暴露传递依赖的模块也称为"**隐式读取(implied read)**"，语法如下：

```java
requires transitive moduleName;
```
下面我们看一下 java.desktop 模块的声明文件：

```java
module java.desktop {
    requires java.prefs;
    requires transitive java.xml;
    exports java.awt;
    ……
}
```
从上面代码的第 3 行可以看出，java.desktop 模块使用了 transitive 语法，在这种情况下，任何使用了 java.desktop 模块的代码都能隐式的使用 java.xml 模块中的类。例如，如果 java.desktop 模块中某个方法返回了一个来自于 java.xml 模块中的类型，那么使用 java.desktop 模块的代码就间接依赖 java.xml 模块，如果没有 transitive 的话，那么使用 java.desktop 模块的模块就必须显式声明依赖 java.xml 模块才能正常编译，否则 java.xml 模块是不可见的。

2. **exports 和 exports to 指令**。exports 指令用于指定一个模块中哪些包对外是可访问的，而 exports...to 指令则用来限定哪些模块可以访问导出类，允许开发者通过逗号分隔的列表指定哪些模块及模块的哪些代码可以访问导出的包，这种方式也称为**限定导出(qualified export)**。

3. **use 指令**。use 指令用于指定一个模块所使用的**服务**，使模块成为服务的消费者，**服务其实就是一个实现了某个接口或抽象类的对象**。

4. **provides...with 指令**。该指令用于说明模块提供了某个服务的实现，因此模块也称为服务提供者。provides 后面跟接口名或抽象类名，与 use 指令后的名称一致，with 后面跟实现类该接口或抽象类的类名。

5. **open, opens, opens...to 指令**。在 Java 9 之前，我们可以通过反射技术来获取某个包下所有的类及其内部乘员的信息，即使是 private 类型我们也能获取到，所以类信息并不是真的与外界完全隔离的。而模块系统的主要目标之一就是实现**强封装**，默认情况下，除非显式地导出或声明某个类为 public 类型，那么模块中的类对外部都是不可见的，模块化要求我们对外部模块应最小限度地暴露包的范围。open 相关的指令就是用来限制在运行时哪些类可以被反射技术探测到。
* 首先我们先看 **opens** 指令，语法如下：
```java
opens package
```
opens 指令用于指定某个包下所有的 public 类都只能在运行时可被别的模块进行反射，并且该包下的所有的类及其乘员都可以通过反射进行访问。

* **opens...to** 指令，语法如下：
```java
opens package to modules
```
该指令用于指定某些特定的模块才能在运行时对该模块下特定包下的 public 类进行反射操作，to 后面跟逗号分隔的模块名称。

* **open** 指令，语法如下：
```java
open module moduleName{
}
```
该指令用于指定外部模块可以对该模块下所有的类在运行时进行反射操作。

以上就是模块化系统主要的指令了，下面我将通过使用上面讲解的指令来演示下模块的基本操作。

### Hello World

