# js 闭包写法的演变过程


闭包就是能够读取其他函数内部变量的函数。在javascript中，只有函数内部的子函数才能读取局部变量，所以闭包可以理解成"定义在一个函数内部的函数"。闭包是将函数内部和函数外部连接起来的桥梁，实现了变量的私有化问题。

有如下例子，我们想要用一个counter记录add函数的执行次数。

```js
function add() {
	var counter = 0;  //局部变量
	counter++;
	console.log("counter = " + counter);
}

add(); // counter = 1
add(); // counter = 1
```
由于counter是局部变量，每次我们执行add()函数，都是输出 counter = 1;
我们想要执行函数时改变counter的值，一种可行的办法是：

```js
var counter = 0;    //全局变量，谁都可以访问，修改
function add() {
	counter++;
	console.log("counter = " + counter);
}
add(); // counter = 1
add(); // counter = 2
```
但是这样会带来问题，由于counter是全局变量，我们可能会在函数外不小心改变了counter的值，
比如在函数外写了一句counter = -100；就打乱了我们原来的计数，显然我们并不希望在函数外任意地改变counter的值。
我们可以这样写：

```js
function add() {
	    var counter = 1; //局部变量
        console.log("counter = " + counter);
	plus = function() { //全局函数
		counter++; 
		console.log("counter = " + counter);
	}
}

add(); //counter = 1 counter初始化
plus(); //counter = 2
```
这样我们就无法在函数外任意改变counter了，进一步的写法是：

```js
function add() {
    var counter = 1; //局部变量
    console.log("counter = " + counter);
	var plus = function() { //局部函数
		counter++; 
		console.log("counter = " + counter);
	};
	return plus;
}
var plus1 = add(); // counter = 1
plus1();           // counter = 2
```
plus函数名有些多余，add可以简化为立即执行函数。

```js
var plus1 = (function add() {
	var counter = 0; //局部变量
	return function() { //全局函数
		counter++; 
		console.log("counter = " + counter);
	};
})();

plus1(); // counter = 1
plus1(); // counter = 2
```

发现add函数名也可以去掉了，简化为匿名函数，如下就是闭包的一般写法：

```js
var plus1 = (function() {
	var counter = 0; //局部变量
	return function() { 
		counter++; 
		console.log("counter = " + counter);
	};
})();

plus1(); // counter = 1
plus1(); // counter = 2
```
闭包实现了变量私有化，局部变量的本质，全局变量的生命周期。
