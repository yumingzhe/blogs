---
layout: post
title: "Java 7 新特性之语法新特性"
date: 2014-09-13 15:57:58 +0800
comments: true
categories: [java]
---
Java 7 于 2011 年 7 月发布，给大家带来了一系列的新特性。不论从功能上还是语法上都给程序员带来了便利。这篇文章围绕 java 7 在语法上的改进进行介绍。

本文主要介绍内容如下：

* switch语句支持字符串类型作为变量
* 增强型的数字表达方式
* 同时捕获多个异常处理
* try-with-resources 语句
* 钻石符改善类型引用

### 1. switch语句可以用字符串类型作为变量

Java7 之前的 switch 语句只支持 integer 类型的变量，如今也开始支持 String 类型的变量了。其实仔细想想，用字符串作为判断条件是很常见的，switch 这一新特性将在判断时简化一堆 if 语句。下面用一个例子来演示一下用法：
```java
public static void main(String[] args) {
        String userName = args[0];
        switch (userName) {
            case "boss":
                System.out.println("Hi boss.Have a good day");
                break;
            case "employee":
                System.out.println("Good morning. Work hard");
                break;
            default:
                System.out.println("welcome");
        }
    }
```

在使用字符串作为变量时需要注意两点：一是要先检查字符串是否为 null，否则执行 switch 语句时将引发空指针异常；二是 case 语句后的字符串是大小写敏感的，否则 case 比较失败。

### 2.增强型的数字表达方式

所谓的增强型数字表达式即书写数值时可以用下划线，下划线将字符分组从而提高代码的可阅读性。下划线不仅可以应用到原生数据类型(如二进制、八进制、十六进制或十进制)上，而且也适用于整型和浮点型数值上。下面看一下例子：
```
public static void main(String[] args) {
        float deposit = 5_000f;
        float withdraw = 2_500f;
        float minAmount = 2_000f;
        if ((deposit - withdraw) > minAmount) {
            System.out.println("current deposit: " + (deposit - withdraw));
        }
    }
```
程序输出的结果是：current deposit: 2500.0

可以看出，在书写较大数字的时候下划线会给程序员带来极大的帮助，防止错误发生，并且在输出的结果中并不带有下划线。还需要注意的是，在实际项目中，如果涉及货币的计算，不能用 float 或 double 来表示数值，因为浮点型是无法精确表示小数，在进行运算时极易发生错误，因此要用 java.util.Currency 或将货币转换成最小的货币单位再用 BigDecimal 类进行包装计算。

下划线的唯一目的就是方便程序员阅读，编译器在生成字节码时会忽略掉下划线，并且连续的下划线也会被当作为一个下划线，最终编译器也会将其忽略。尽管下划线使用起来很简单，但语法上也有一些限制：不能放在数值的开头；不能与小数点相邻；书写 float 或 double 类型的数值时不得放在 D,F,L 后缀的前面。下面分别演示下错误的用法：
```java
long deposit=_1234_5678_90L;
float pi=3._1415926f;
long number=123_456_789_L;
```
<!--more-->

### 3.try-with-resources

在Java7之前，打开或关闭资源是非常繁琐的，比如打开 InputStream，通常要放在 try 块中，还要有 catch 块用于捕获异常，并且要有 finlally 块用于关闭流。但 Java7 引进的 try-with-resources 特性将极大的简化代码，可以避免嵌套或过多的 try-catch 块，使用这样的新特性，打开的资源在 try 块执行完后会自动关闭，前提是这些资源都要实现 java.lang.AutoCloseable 接口，不过不用担心，jdk7 中几乎所有的资源类都实现了这个接口。下面用代码简单演示下：
```java
public static void main(String[] args) {
        try (
             BufferedReader inputReader = Files.newBufferedReader(Paths.get("/home/user/user.txt"), Charset.defaultCharset());
             BufferedReader outputWriter = Files.newBufferedWriter(Paths.get("/home/user/user.bak"), Charset.defaultCharset())
        ) {
            String line;
            while ((line = inputReader.readLine()) != null) {
                outputWriter.write(line);
                outputWriter.newLine();
            }
            System.out.println("back up complete!");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

这个程序演示的是将“/home/user”目录下的”user.txt”文件进行备份，备份到相同目录下，文件名为”user.bak”。从代码中可以看出，try-with-resources 的语法是将要打开的资源放到 try 后的括号中，并且每条语句用分号隔开。就这个程序来看，我们将 BufferedReader, BufferedReader 放在 try 后，由于在打开流时可能会产生 IO 异常所以增加了 catch 块用于捕获运行时的异常。最重要的是，程序最后没有用于关闭这些字节流的代码，这说明我们打开的资源将会在 try 块结束后自动被关闭。

需要注意的是不论 try 块是否正常执行完毕，打开的资源最终都会被关闭，如果资源无法关闭则会抛出异常。并且从程序的执行流程上来说，无论资源是否正常关闭，try 和 finally 块的代码将会永远被执行。

### 4.钻石运算符
先看个代码：
```java
List list = new ArrayList<>();
```
“ArrayList<>”中的”<>”就是所谓的钻石运算符。钻石运算符通常用于简化创建带有泛型对象的代码，可以避免运行时的异常，并且它不再要求程序员在编码时显示书写冗余的类型参数。实际上，编译器在进行词法解析时会自动推导类型，自动为代码进行补全，并且编译的字节码与以前无异。

### 5.同时捕获多个异常处理

在一个 try 块中，可能会抛出多个异常，在 Java7 之前我们需要编写多个 catch 块分别来捕获异常，并且通常情况下我们处理这些异常的方式都一样，例如将这些异常记录到日志中就完事了，但这样会导致很多冗余的代码。

到了 Java7，我们可以在一个 catch 块中处理多个异常，从而精简了代码。之前想要一次处理多个异常，通常需要将 catch 块中的异常声明为更高层次的异常类才能捕获不同类型的异常，但这样的处理不是很优雅。使用方法如下：
```java
public static void main(String[] args) {
        try {
            Reader reader = Files.newBufferedReader(Paths.get("/home/user/file.txt"), Charset.defaultCharset());
            DriverManager.getConnection("jdbc:mysql://localhost:3306/db", "root", "toor");
        } catch (IOException | SQLException e) {
            e.printStackTrace();
        }
    }
```
从上面的代码可以看出，在 try 块中有 2 条语句，会分别抛出 IOException 和 SQLException 异常，但我们只用了一个 catch 块就捕获了这两个异常，并且对异常只做了相同的处理。因此捕获多个异常的语法是将 try 块中要抛出的受检异常类型全部声明到 catch 语句中，不同类型间用“|”分割。需要注意的是，使用这种语法只能对多个异常进行相同的处理，若要针对不同的异常进行不同处理，还需要使用多个 catch 块进行捕获、处理。
