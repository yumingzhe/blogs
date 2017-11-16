---
layout: post
title: "Gradle 入门教程(1)——Groovy 基础知识"
date: 2017-04-12 11:30:01 +0800
comments: true
categories: [groovy, gradle] 
---
其实很早就想写关于 Gradle 的教程了，这个很早可以追溯到大约 2 年前，可是鬼知道我这 2 年经历了什么，总之拖延症晚期患者你们伤不起啊。废话不多说了，下面进入正题。

首先说说 Gradle 是个啥，很简单，它是一个项目构建工具，“项目构建工具”听起来很高大上，其实做过项目的人天天都在用，我们最熟悉的maven就是一个项目构建工具。为了照顾没用过 Maven 的小伙伴们，我还是简单的解释下吧，引用下《Maven 实战》一书中关于项目构建工具的解释，”项目构建工具能够帮我们自动化构建过程，从清理、编译、测试到生成报告，再到打包和部署，不需要也不应该一遍遍地输入命令“。作为程序员，我们每天开工时都要更新代码并进行编译，项目开发完毕后还要打war包，然后还要部署到服务器上，部署时可能还需要运行一些脚本进行初始化，可以看出在整个开发过程中我们需要把对项目的不同阶段的产物当作输入，进行一些操作后输出产物作为下一阶段的输入，例如，把开发阶段把源代码当作输入，通过编译操作输出编译后的class文件，打包阶段把编译产生的class文件压缩到war包中等等。而对每个阶段的操作根据项目情况一般都是以固定的方式进行的，而这就为自动化带来了契机。所以先设想一下，我们要是能把所有阶段的操作都写成脚本，想要执行某一阶段的操作，我们只需从头执行脚本，一直执行到目标阶段的脚本结束为止即可实现以往手动耗时耗力的工作，节省了时间。而这就是项目构建工具的初衷，所以诸如 Maven，Gradle 之类的工具就诞生了。

再来说说为啥要用 Gradle 而不用 Maven。大家都知道 Maven 独霸 Java 市场好多年，个人感觉秒 ant 等工具好几条街，但它仍然使用 xml 作为配置文件，虽说可以通过 schema 进行语法校验，防止人为出错，但从学习曲线、需要输入字数等方面考虑还是很让人受伤的，如果项目比较大，定义的依赖、编译配置等信息一长串，满屏都是 xml 标签，对开发人员来说是非常的不友好。而 Gradle 正是看中了 Maven 这一弱点，以 Groovy 语言为基础定义了一套自己的语法规则(DSL)，学习难度不算大，只是一开始可能语法接受不了，但习惯之后配置出的项目相当简洁，比 Maven 不知省了多少字。虽说 Gradle 推出的时间不长，资质比 Maven 也低很多，但其简洁的风格，简直就是 Java 项目构建领域的一股“清流”哇，一经推出不久便获得了Spring 的大奖，现在许多知名的开源项目都开始迁移到了 Gradle 上了，比如 Spring、Hibernate等等。再加上近几年移动应用的崛起，Android 项目默认就是以 Gradle 为默认构建工具的，所以 Gradle 无论是从提高效率还是顺应潮流都是值得我们去学习的。下面就开始我们的学习之旅吧，正如前面提到的， Gradle 是以 Groovy 语言为基础的，所以我们先学习下 Groovy 的基础知识，有了这些我们学习 Gradle 的速度就大大提高，就如学会了九阳神功再学习乾坤大挪移只是分分钟的事儿。

### 一、Groovy 简介
Groovy 是一门可以运行在 Java 虚拟机上的动态编程语言，也就是说 Groovy 代码经过编译器编译后也生成字节码文件，然后可以由 Java 虚拟机加载并运行，虽说 Groovy 语言是动态的，但它同时具备像 Python 这种脚本语言的动态功能，同时也具备像 Java 这种面向对象的静态语言的特性。由于 Groovy 编译后也是字节码文件，它能与 Java 进行很好的集成，甚至两种语言可以混写或将 Groovy 文件打成 jar 包供程序调用。Groovy 在语法上有很多地方和 Java 相类似，所以如果你有一定的 Java 基础的话，那么学习 Groovy 将会是一件很轻松的事，因为只需要拿 Java 的语法做类比就能轻松地掌握。下面我们就用类比的方式学习 Groovy 的基本语法。

我们在学习 Java 时，第一堂课老师都会告诉我们一句非常重要的话—— Object 类是所有 Java 类的父类，这句话同样适用于 Groovy，**在 Groovy 中，所有的类型的父类也都是 java.lang.Object**。

<!--more-->

### 二、HelloWorld入门

首先，我们还是以在控制台打印”hello world” 为例作为我们第一个 Groovy 入门程序。Groovy 的安装这里不再讲了，和 Java 一样，下载并安装 GDK，也需要配置下环境变量GROOVY_HOME，以及把GROOVY_HOME/bin 添加到 PATH 中。

用文本编辑器编写下面的代码并保存到 HelloWorld.groovy 文件中：
```groovy
println "Hello World"
```

然后执行下面的命令运行 Groovy 脚本，然后我们就可以在控制台看到打印出来的语句了：
```bash
groovy HelloWorld.groovy
```

从上面的代码中我们可以看出，简洁是 Groovy 语言的特点，什么都是拿来就用，不像 Java 那样要定义包、类，声明 main 函数，然后调用冗长的 System.out.println(…) 来实现打印功能，并且语句也没有分号。需要注意的是我们的脚本文件不一定要以 .groovy 结尾，只要是文本文件 groovy 命令都能运行，但是从可阅读的角度出发还是建议所有的 groovy 代码都保存在 .groovy 文件中。

除了上面所演示的通过脚本的方式来运行 Groovy 代码外，我们还可以将其编译为 class 字节码文件运行，方法如下：
```bash
// 编译
groovyc HelloWorld.groovy
//运行
java -cp %GROOVY_HOME%/embeddable/groovy-all-2.3.1.jar;. HelloWorld
```

上面的代码，首先使用 groovyc 编译器将 groovy 代码编译为 class 文件，然后用 java 命令运行，整个过程和 Java 没什么区别，但是在运行时需要把 groovy-all.jar 加入到 classpath 中，否则会找不到某些支持运行的基础类。**其实无论是通过脚本的方式运行还是先编译再运行的方式，Groovy 代码最终都是运行在 JVM 上，这两种方式都会先将 Groovy 代码编译成字节码，只不过以脚本方式运行时，编译后的字节码是存储在内存中，而编译的方式是将字节码存放在磁盘上**。

### 三、数据类型
学习任何一门语言，首先要学习的就是类型系统，也就是这门语言内置了哪些数据类型供我们使用。和 Java 一样，Groovy 基本类型有 Byte, Short, Integer, Long, Float, Double, Character, Boolean, String 类型，但是与 Java 相比最不同的就是 Groovy 不支持原生类型，即int, short, long, float, double，所以 Groovy 是一门纯面向对象的编程语言，也就是说它里面的任何东西(类、方法等)都是对象，我们可以在其上进行方法调用。

我们需要着重讲一下 String 的用法，因为 Groovy 里字符串的表示方法很多，不像 Java 里仅用引号表示一个字符串那样单调苍白。请看下面的代码：
```groovy
def s1='This is single quote string.'
def s2="This is double quote string."
def s3="""This is multi line String.
You can write multiple lines here."""
def s4 ="Example of Gstring, You can refer to variable also like
${s1}"
def s5='''This is multi line String.
You can write multiple lines here.'''
```

上面的代码中，s1 使用单引号来表示字符串；s2 使用双引号表示字符串，这种方式和 Java 的一样；s3 使用三个双引号来表示字符串，这种方式允许字符串跨行；s4 使用了双引号表示字符串，但其中包含了类似模板占位符的东西，当使用字符串时占位符会被替换为相应的值，这种字符串也称为 GString，占位符的写法可以写成 ${var}，也可以写成 $var；s5 使用三个单引号表示字符串，允许字符串跨多行。

### 四、动态类型

Groovy 既支持静态类型又支持动态类型。静态类型可以在编译时允许编译器进行语法检查、内存优化，甚至 IDE 也可以利用静态类型来为我们提供智能的代码自动完成功能。然而 Groovy 强大之处就在于对动态类型的支持，也就是说在运行时我们无法得知某个变量到底存储着什么类型的数据，并且变量可以在运行时动态指向任意类型的数据。我们可以通过 def 关键字来定义一个动态类型的变量或方法，如下面代码所示：

```groovy
def var1
var1 ='a'
println var1.class // 打印 java.lang.String
var1 = 1
println var1.class // 打印 java.lang.Integer
def method1() {/*method body*/}
```

上面的代码中，我们定义了一个变量 var1，先给它赋了一个字符串，打印出变量的类型是 String，随后我们又将其值改为整数 1，再打印其类型就变成了 Integer，这就是 Groovy 的动态类型，允许在运行时动态改变值的类型。

动态类型的另一个常见用法就是在对象上调用方法，由于对象的类型未知，所以在运行时动态变换对象类型并调用相同的方法时就可能会产生不同的效果，请看下面的例子：
```groovy
def addition(a, b) { return a + b}
addition (1, 2) // Output: 3
addition ([1,2], [4, 5]) // Output: [1, 2, 4, 5]
addition('Hi ', 3) // Output: Hi 3
```

上面的代码中，我们定义了 addition() 方法，接收 2 个参数，并返回这两个参数的和。如果我们传入的类型是两个整数，那么方法就会返回整数的和；如果我们传入的是两个 list，那么方法会合并 list 并返回，如果传入的是字符串和数字，那么就会进行字符串连接操作。但是要是我们传入的是用户自定义类型的变量，那么 Groovy 将会如何处理呢？假设我们自定义了一个类如下：
```groovy
class Person{
    String name
}
p1 = new Person()
p2 = new Person()
addition(p1, p2) // groovy.lang.MissingMethodException
```

可以看出代码会报异常，因为我们自定义的类并没有为加法定义实现，所以解决方法就是为 Person 实现一个 plus()方法或者实现 methodMissing() 方法，这个方法很重要，当调用对象的某个方法不存在时就会调用此方法进行处理，这里就不再赘述，感兴趣的同学可以查看官方文档。

### 五、Class和方法

本节我们介绍下 Groovy 类相关的知识，Groovy 中的类和 Java 类很类似，都用 class 关键字进行声明，但是一个很重要的地方是会默认导入 6 个包和 2 个类，而 Java 默认只导入 java.lang 包，如下面代码所示：

```java
import java.lang.*
import java.util.*
import java.io.*
import java.net.*
import groovy.lang.*
import groovy.util.*
import java.math.BigInteger
import java.math.BigDecimal
```

Groovy 类和方法默认都是 public 的访问权限，而 Java 默认是包访问级别的，下面我们定义一个类：
```groovy
class Customer{
    String name
    String email
    String address
    String showMail(){
        email
    }
    static main(args) {
        Customer c = new Customer(name: "zhangsan", email: "zhangsan@mail.com");
        c.setAddress("zhangsan's address");
        println c.showMail();
    }
}
```

上面的代码中，我们定义了一个 Customer 类，它有 3 个字段和 1 个方法，默认都是 public 的，然后我们在 main 方法中实例化这个类的一个对象，每个类都有一个默认构造函数也就是无参构造函数，而这里我们传入了具名的参数，此时 Groovy 会调用默认构造函数并在里面根据参数的名字调用相应的 settter 方法为字段赋值，而我们并没有为字段定义 setter 方法，那这又是怎么回事呢？原来如果我们的字段使用默认访问级别的话，Groovy 会自动为字段生成 getter 和 setter 方法，如果我们显式地指定字段的访问级别，如 public、private、protected 的话那么就不会自动生成 get 和 set 方法，需要我们自己手动编写，这也是代码中可以直接在对象上调用 setAddress() 方法的原因。Groovy 中的方法和 Java 的很类似，想要调用类中的某个方法，需要先实例化一个对象，然后在对象上调用方法即可；还有一种情况就是如果我们在编写 Groovy 脚本时并没有定义类，只定义了方法，那么只要调用方法名即可。如果方法支持返回动态类型的话，那么方法的声明必须以 def 关键字开头。Groovy 对方法进行了扩展，支持方法有默认参数，请看下面的例子：

```groovy
def sum(x,y=10,z=1) {x+y+z}
// x = 1
sum(1)
// x = 1, y= 2
sum(1, 2)
```

上面的代码中我们定义了一个函数 sum，接收 3 个参数x,y,z，其中x,y 默认值是 10， z 默认为 1，函数返回三者的和。当我们传入的参数个数与形参个数不一致时，会依次先赋值给前面的参数。

### 六、流程控制语句
* if…else
if…else 语句与 Java 的类似，只有一点不同，就在于 Groovy 的 if 在计算条件的方式不同。Groovy 中非零整数、非空引用、非空字符串、初始化的集合都会计算为 true 值。

* 猫王操作符(Elvis operator)
尽管 Groovy 支持三元操作符”a?b:c”，但是为了简化，推出了猫王操作符(?:)，头左转 90 度看是不是很像猫王的发型？猫王操作符用于测试一个值是否为 null，示例代码如下：

```groovy
def inputName
String username = inputName?:"guest"
```
上面的代码中，我们定义了一变量 inputName，然后用猫王操作符判断变量是否为 null，如果为空则为其赋默认值“guest”。

* switch

Groovy 支持 Class, Object, Range, Collection, Pattern 和 Closure 等类型作为 switch 语句的条件表达式，凡是实现了 isCase() 方法的实体都可以作为 switch 语句表达式，下面代码演示了各种类型作为 switch  表达式语句：
```groovy
def checkInput(def input){
    switch(input){
        case [3, 4, 5] : println("Array Matched"); break;
        case 10..15 : println("Range Matched"); break;
        case Integer : println("Integer Matched"); break;
        case ~/\w+/ : println("Pattern Matched"); break;
        case String : println("String Matched"); break;
        default : println("Nothing Matched"); break;
    }
}
checkInput(3) // Array Matched
checkInput(1) // Integer Matched
checkInput(10) // Range Matched
checkInput("abcd abcd") // String Matched
checkInput("abcd") // Pattern Matched
```

* 循环
Groovy 支持经典的 for 循环，也支持 foreach 这种语法糖，foreach 的语法为 for(var in iterable)，其中 foreach 可迭代的集合包括数组、range和集合，情况下面的实例代码：
```groovy
// 传统的for循环
for(int i = 0; i< 3; i++) {...}
// foreach 迭代 range
for(i in 1..5) println(i)
// 迭代数组
def arr = ["Apple", "Banana", "Mango"]
for(i in arr) println(i)
// 迭代集合
for(i in ([10,10,11,11,12,12] as Set)) println(i)
```

* while 循环
Groovy 中的 while 循环和 Java 的一样，只不过不支持 do…while 语法。

### 七、集合

本节将介绍一下 Groovy 提供的集合类以及常用的方法，如果对 Java 的集合框架比较熟悉的话，那么学习本节也很轻松，因为 Groovy 除了具有 Java 中常用的集合类如 Set, List, Map 外还提供了 Range 类，此外 Groovy 还提供了更精简的语法，相比于 Java 更容易使用，开发效率也很高。

* Set

Set 用于存放无序的不重复的对象，也可以存放 null，它暴露的方法和 Java 里的 Set 一样，常用的有 add , addAll , remove , removeAll 方法，而且也提供了集合的并集和交集的操作。实例代码如下：

```groovy
// 创建集合
def Set1 = [1,2,1,4,5,9] as Set
Set Set2 = new HashSet(['a','b','c','d'])
// 修改集合
Set2.add(1)
Set2.add(9)
Set2.addAll([4,5]) // [1, d, 4, b, 5, c, a, 9]
Set2.remove(1)
Set2.removeAll([4,5]) // [d, b, c, a, 9]
// 求集合的并集
Set Union = Set1 + Set2 // [1, 2, 4, 5, 9, d, b, c, a]
// 求集合的交集
Set intersection = Set1.intersect(Set2) // [9]
// 求集合的补集
Set Complement = Union.minus(Set1) // [d, b, c, a]
```

* List

List 用于存放有序的对象集合，里面的元素可以重复，创建一个空 List 语法为： List list = []，这个list 默认实现类为 java.util.ArrayList，下面的代码演示了从创建 List到如何使用的示例：

```groovy
// 创建list
def list1 = ['a', 'b', 'c', 'd']
def list2 = [3, 2, 1, 4, 5] as List
// 读取list中的元素
println list1[1] // b
println list2.get(4) // 5
println list1.get(5) // 抛出IndexOutOfBoundsException异常
// 一些有用的工具方法
// 对list进行排序
println list2.sort() // [1, 2, 3, 4, 5]
// 反转
println list1.reverse() // [d, c, b, a]
// 查找
println ("Max:" + list2.max() + ":Last:" + list1.last()) // Max:5:Last:d
```

从上面的代码中可以看出创建 list 有 2 中方式，get() 方法使用与 Java 一样，但是 Groovy 对 List 进行了扩展，添加了很多便利的方法如 sort、reverse、max 等，极大地提高了效率。

* Map

Map 也称为字典，由 key 和 value 组成并且 key 也是唯一的不能重复，key-value 对通过冒号分隔，创建一个空 map 语法为：Map map = [:]，默认实现类是 java.util.HashMap，如果 key 是字符串类型，引号可以忽略，如 Map map = [name:’zhangsan’]。需要注意的是 Map 的键 默认是字符串，如果想要使用其他类型作为 key 的话，必须把 key 放在括号里，如下所示：

```groovy
Integer key = 24
Map map = [(key): "twenty-four"]
```

通过 key 访问 value 和 Java 的一样，都通过 get() 方法，如 map.get(“id”)。注意，如果 key 是字符串时，必须用双引号将 key 引用起来然后再传给 get() 方法，否则 key 会被当做变量进行解析从而引发错误。

下面讲一下有用的工具方法的使用：

```groovy
map.each({key,value -> println key + ':' + value});
map.each({entry -> println entry.key + ':' + entry.value});
map.any({entry -> entry.value > 0});
map.every({entry -> entry.value > 0});
```

上面演示了 each、any、every 的用法，这些方法都接收闭包作为参数。each 用于遍历集合中的所有键值对，而 any 和 every 都用于判断集合中的元素是否符合闭包定义的条件，返回值均为布尔值。

* Range

Range 对于 Java 程序员来说是一个比较新颖的集合类型，它能够定义一个数据范围，由两个值组成，一个是起始点，一个是结束点，实例如下：

```groovy
def range1 = 1..10
Range range2 = 'a'..'e'
range1.each { println it }
List l = range1.step(2) // [1, 3, 5, 7, 9]
```

上面代码中我们首先定义了 range1，范围从 1 到 10，然后又定义了 range2 范围从 a 到 e，然后打印出 range1 中的每个元素，我们还可以使用 step() 方法改变 Range 的步长用于改变生成的元素。

* 闭包

闭包一般属于函数式编程(Functional Programming)里的概念(函数式编程属于一种编程风格，偏重以简洁的函数来实现编程目的而不是以指令实现目标，与之相对的是命令式编程，函数式编程中的函数应该是没有副作用的，即函数不应该改变其作用域之外的变量，想要了解函数式编程，请看这篇文章)，Groovy 里的闭包其实就是一个用花括号包起来的参数列表加上代码块， Java 8 通过 Lambda 表达式和函数式接口为 Java 引入了函数式编程，于是有人就认为 Lambda 表达式和 Groovy 里的闭包是一种东西，其实它们还是有很大的区别的，其中最重要的就是 Groovy 的闭包支持委托机制，下面会讲这里提一下。闭包是 Groovy 中的一级公民，可以将其赋值给变量，可以当做参数传给方法，也可以像方法一样直接调用并且总是会有返回值，默认会返回最后一个语句的。 Groovy 里的闭包的语法是这样的： {参数列表 -> 代码块}，其中参数用逗号分隔，参数和代码块之间用 -> 分隔。如果闭包不声明参数列表的话，闭包体内可以访问一个隐式的无类型的参数叫 it，it 默认指向参数列表的第一个参数 。参数是可选的，如果不指定参数，那么 it 默认值是 null，下面我们演示下闭包的用法：

```groovy
def constant = 2
def addTwo = {it+constant}
addTwo(2) // 4
addTwo() // NullPointerException
```

上面的代码中我们定义了一个闭包，但是没有定义参数列表，在方法体内我们引用了 it 变量，它会存储传入的参数，当我们调用这个闭包时向其传入一个参数 2 ，此时 it 就被赋值为 2 然后经过计算后返回 4，如果我们无参调用闭包时由于 it 是 null 会引起空指针异常。上面的代码由于使用了 def 定义了一个动态类型的变量来引用闭包，其实我们还可以使用更具体的闭包类型来声明变量，所以上面的代码可以改写为：

```groovy
groovy.lang.Closure addTwo = {it+constant}
```

还需要注意的是，上面的代码中，我们在闭包里引用了一个闭包作用域外部的变量 constant，这样的变量称为自由变量，而定义在闭包内部的变量称为局部变量。

关于 Groovy 的闭包还有一个语法糖就是，如果闭包作为方法的唯一一个参数或最后一个参数，那么闭包就可以放到方法的圆括号外面，甚至圆括号也可以省略，请看下面的代码：

```groovy
list.each({println it })
list.each(){println it}
list.each {println it}
```

上面的代码中我们为 each() 方法传入一个闭包，第一种写法是将闭包作为参数放到圆括号中，第二种写法是将闭包放到括号的外面，第三种写法就直接省略括号了。

此外闭包还有一个非常重要的特点就是委托机制(delegate)，什么意思呢？闭包的执行都是与某个“上下文”相关的，闭包在执行的过程中对变量、方法的解析都默认在这个上下文中查找，而闭包可以指定这个上下文的值，这就是委托机制，将执行环境委托给某个类或对象，让闭包在其中执行，请看下面的代码：

```groovy
class Person {
    String name
}
def p = new Person(name:'Zhangsan')
def cl = { name.toUpperCase() }             
cl.delegate = p
c1() // ZHANGSAN
```

上面的代码中我们定义了一个闭包 c1，其方法体内 name 这个变量还不知道属于谁，但是通过设置 c1 的 delegate 属性，指定其执行环境为 p 这对象，再调用 c1 闭包后 name 就会解析到 p 对象的 name 属性上。

上面就是 Groovy 的一些基础知识，如果你有 Java 的基础那么学习起来会很快，有了这些知识我们学习 Gradle 也会很快上手，下一讲我们就开始讲解 Gradle 的基础知识。
