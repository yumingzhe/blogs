---
layout: post
title: "JavaScript 作用域和变量声明提升"
date: 2015-03-15 13:27:15 +0800
comments: true
categories: [javascript,hoisting] 
---
首先给大家出道题，运行下面的js代码后，foo的值是什么？

```javascript
var foo = 1;
function bar() {
	if (!foo) {
		var foo = 10;
	}
	alert(foo);
}
bar();
```
答案是 10，如果这个结果让你感到不解的话，请接着看下面的题，相信我，它会让你抓狂的：
```javascript
var a = 1;
function b() {
	a = 10;
	return;
   function a() {}
}
b();
alert(a)
```

这次答案是“1”。怎么样，又答错了吧？这段代码看起来比较奇怪，让人困惑，但这也正体现了 JS 这门语言所具有的强大而又具有展现力的语言特性。上面代码的行为之所以会产生让你意想不到的结果，关键在于 JS 有‘变量声明提升’这个特性，本文将对此机制进行分析，但是首先让我们先回顾一下 JavaScript 的作用域这个知识点。

### JavaScript作用域

对于 JavaScript 初学者来说，最难以接受的恐怕就是其作用域了。实际上，即使是有经验的 JS 程序员有时也得犯迷糊。主要原因是 JS 看起来具有 C 语言的风格，首先请看下面的C代码：

```cpp
#include <stdio.h>
int main() {
	int x = 1;
	printf("%d, ", x); // 1
	if (1) {
		int x = 2;
		printf("%d, ", x); // 2
	}
	printf("%d\n", x); // 1
}
```

上面的程序将会输出 1,2,1。因为 C 语言的作用域是块级的，当程序执行到一个块级代码时，比如上面的 if 语句，在块内可以声明新的变量，与外部的变量不会产生命名冲突。而这在 JavaScript 中可是会产生问题的。请在控制台中运行下面的代码：

```javascript
var x = 1;
console.log(x); // 1
if (true) {
	var x = 2;
	console.log(x); // 2
}
console.log(x); // 2
```

运行后可发现会输出 1,2,2。这是因为 JavaScript 的作用域是函数作用域，函数的边界确定了变量的生命周期，只有函数才能产生新的作用域。

对于 C，C++，C# 或 Java 程序员来说，这种特性可能难以接受，但好在 JavaScript 函数比较灵活，还是值的我们去学习的。如果想要在一个函数内创建一个临时的作用域，可以使用像下面的代码来实现：
```javascript
function foo() {
	var x = 1;
	if (x) {
		(function () {
			var x = 2;
			// some other code
		}());
	}
	// x is still 1.
}
```
上面的代码比较灵活，可以用在任何需要临时作用域的环境下。如果理解了 JS 的作用域，那么掌握变量的声明提升就容易的多了。

### 声明，名字和提升

在 JavaScript 中, 可以通过以下 4 种方式将变量(或者命名)引入到某个作用域中:

* **语言内置**的方式: JS 的所有作用域(即全局和函数作用域)中都默认带有 this 和 arguments 这两个变量，这两个变量会自动引入到所在的作用域中，无需用户干涉。
* **函数参数**的方式: 每个函数都可以声明形参，而这些形参的作用域就限定在此函数内。
* **函数声明**的方式: 形如 function foo() {} 这样的声明会在函数所在的作用域内引入 foo 这个命名。
* **变量声明**的方式: 形如 var foo 的变量声明，会在变量所在的作用域内引入 foo 这个命名。

函数声明和变量声明总是会被 JavaScript 解释器隐式地移动(提升)到它们所在作用域的最顶端，而函数参数和语言内置的命名本来就已经处于最顶端了。什么意思呢？请看下面的代码：
```javascript
function foo() {
	bar();
	var x = 1;
}
```

上面的代码实际上会被解释器转换为下面的形式：
```javascript
function foo() {
    var x;
    bar();
    x = 1;
}
```

可以看出变量的声明与执行顺序无关。而下面的两个函数也是等价的：
```javascript
function foo() {
	if (false) {
		var x = 1;
	}
	return;
	var y = 1;
}
function foo() {
	var x, y;
	if (false) {
		x = 1;
	}
	return;
	y = 1;
}
```

需要注意的是变量的赋值是不会提升的，只有声明才会被提升。而函数的声明又有些细微的不同，因为函数声明是将整个函数体一起提升。但是要知道函数声明有以下两种方式，请看下面的代码：
```javascript
function test() {
	foo(); // TypeError "foo 不是一个函数"
	bar(); // "这行代码是有效的!"
	var foo = function () { // 将函数表达式赋值给局部变量 'foo'
		alert("this won't run!");
	}
	function bar() { // 函数声明, 引入 'bar' 命名
		alert("this will run!");
	}
}
test();
```

在上面的例子中，只有 bar 函数及其函数体被提升，以及 foo 变量的声明被提升，但对 foo 的赋值并未提升，整个方法体仍保留在了原来的位置上，因为赋值只有在运行时才会执行。

上面就是 JS 声明提升的基础知识，还不太复杂吧。但是在某些特例中，声明提升还是会让人摸不着头脑。

### 命名解析次序

有了上面的基础后，我们再来说说特例，其实这些所谓的特例(或者说是按常理推算但结果却是大相径庭的情况)大多数是由命名解析次序导致的。上文中我们提到有四种方式可以将某个变量引入到作用域中，其实我列出的顺序就是它们被解析的顺序。通常，如果某个变量已经定义了，再定义同名变量并不会覆盖原来的值，JS 会忽略定义语句。但是需要注意的是函数声明的优先级要比变量声明的优先级要高，这并不意味着没法把函数赋值给变量，只是这种情况下函数声明会被跳过。 还有几个需要注意的地方:

* JS内置的变量 arguments 的行为比较怪异。它的声明是位于函数形参之后，函数声明之前，也就是说如果函数的形参也叫 arguments, 即使其值是 undefined，它声明的优先级也高于内置的 arguments。这是 JS 里比较恶心人的地方，所以不要使用 arguments 作为参数名。
* 如果使用 this 作为变量名将会报 SyntaxError 错误，也是 JS 好的一面。
* 如果函数的多个参数同名，那么最后一个优先级最高。

### 命名函数表达式
我们可以通过使用函数表达式的方式给函数命名，其语法与函数声明很类似。但这种方式并不是函数声明的变体，名字不会引入到作用域中，不会对函数体进行声明提升，示例代码如下：
```javascript
foo(); // TypeError "foo 不是一个函数"
bar(); // 有效
baz(); // TypeError "baz 不是一个函数"
spam(); // ReferenceError "spam 未定义"

var foo = function () {}; // 匿名函数表达式 ('foo' 变量的声明会提升)
function bar() {}; // 函数声明 ('bar' 命名以及函数体都会提升)
var baz = function spam() {}; // 命名函数表达式 (只有变量 'baz' 的声明会提升)

foo(); // 有效
bar(); // 有效
baz(); // 有效
spam(); // ReferenceError "spam 未定义"
```

### 编码实践
到目前为止，大家应该都对 JS 的作用域和提升有了比较深的理解了，但是这些知识对 JS 编码有何帮助呢？我想从中得出最重要的经验就是总是用 var 来声明变量。我个人强烈推荐大家在每个作用域的顶部用一个 var 来声明所有的变量。 如果大家都能按这种方式编码，那么几乎不可能踩到与变量提升相关的坑。然而这种做法也有些不足，比如它让编码变得不清晰，我们无法保证作用域内的所有变量都是用 var 声明过得，如若漏掉一个，那么变量就会泄露到全局作用域中。我推荐使用 JSLint，开启 onevar 选项来对代码进行安全检查，使用方法如下：
```javascript
/*jslint onevar: true [...] */
function foo(a, b, c) {
    var x = 1,
    	bar,
    	baz = "something";
}
```

### 引述规范

我想当遇到问题时，无论查找什么资料，规范(ECMAScript Standard (pdf))无疑是最具有权威性的。那么本文所讲解的关于变量声明和作用域等知识在规范中是如何定义的呢？请看下面(摘录自12.2.2节, 老版本):

> If the variable statement occurs inside a FunctionDeclaration, the variables are defined with function-local scope in that function, as described in section 10.1.3. Otherwise, they are defined with global scope (that is, they are created as members of the global object, as described in section 10.1.3) using property attributes { DontDelete }. Variables are created when the execution scope is entered. A Block does not define a new execution scope. Only Program and FunctionDeclaration produce a new scope. Variables are initialised to undefined when created. A variable with an Initialiser is assigned the value of its AssignmentExpression when the VariableStatement is executed, not when the variable is created.

如果变量出现在函数声明内部，那么变量的作用域就是它所在函数的(函数)作用域。否则，它就处于全局作用域中(也就是说它将作为全局对象的一个属性)。变量创建的时机是代码一旦进入某个作用域后就立即创建。而代码块并不会创建作用域。只有程序和函数声明才会创建新的作用域。变量创建后将会初始化为 undefined。变量的值只有在对变量的赋值语句执行后才有，而不是变量创建的时候就赋值。

我希望本文能为那些还在为不理解 JS 的某些令人困惑的问题而苦苦挣扎的同学带来些许光明，同时我也尽可能的按照通俗、透彻的原则进行讲解从而避免挖更多的坑。

注：本文翻译自http://www.adequatelygood.com/JavaScript-Scoping-and-Hoisting.html
