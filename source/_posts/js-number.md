---
title: js-number
date: 2019-02-28 14:29:22
categories:
  - js
tags: 
  - javascript
  - javascript core
  - 原生javascript
comments: false
---
<!-- post9 -->
# Number对象
js中number是基础数据类型，此外还存在Number对象类型（js提供的number的包装类）。js引擎会在**通过"."符号来引用数字变量的属性**时，为其自动创建一个**临时的Number实例**对value进行包装以便使用Number类型的方法。需要注意的是 **不能通过 数字直接量+"." 的形式来创建临时的包装对象**
```javascript
// 通过变量引入数字，js引擎自动创建Number实例
var n = 2; // 此时变量n存储者原始值2 
console.log(n.toString()); // "2"  通过n.尝试引用属性时，js引擎自动创建一个Number类型的包装对象，提供toString()方法

// 通过数字直接量引用Number对象方法，会报错
console.log(2.toString()); 
// Uncaught SyntaxError: Invalid or unexpected token

// 通过为number的包装对象赋值来确认这个包装对象是临时的
var n = 2;
n.test = '添加测试属性' // 创建出一个临时的包装对象，并且为这个对象添加test属性
console.log(n.test); // undefined  上一句的临时包装对象已经被“销毁”，n.test是在尝试引用一个新的包装对象的test，于是输出 undefined

// boolean 与 string也存在着上述临时包装对象的现象。

// 可以通过new Number的方式，显式创建Number包装对象
var n = new Number(2);
var m = new Number(2);
n === m; // false n与m分别指向不同的包装对象
n === 2; // false n 引用的是包装对象，不是原始值2
```

## [Number类型的MDN文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Number)

### Number([value]) 与 new Number([value]) 的区别
根据ecma5.1标准，[点此到达](https://www.ecma-international.org/ecma-262/5.1/#sec-15.7)

Number ( [ value ] )：将传入的value转换为number，返回一个number原始值
Returns a Number value (not a Number object) computed by ToNumber(value) if value was supplied, else returns +0.


new Number ( [ value ] )：将传入的value转换为number，然后返回一个Number类型的包装对象
The [[Prototype]] internal property of the newly constructed object is set to the original Number prototype object, the one that is the initial value of Number.prototype (15.7.3.1).

The [[Class]] internal property of the newly constructed object is set to "Number".

The [[PrimitiveValue]] internal property of the newly constructed object is set to ToNumber(value) if value was supplied, else to +0.

The [[Extensible]] internal property of the newly constructed object is set to true.

```javascript
Number(3) === 3 // true   Number(3)返回原始值3
new Number(3) === 3 // false new Number 返回一个包装对象
new Number(3).valueOf() === 3 // true valueOf 返回了包装对象内部的原始值3
```


### 静态属性
* Number.EPSILON：两个可表示(representable)数之间的最小间隔。   
* **Number.MAX_SAFE_INTEGER：JavaScript 中最大的安全整数 (2^53 - 1)**
* Number.MAX_VALUE：能表示的最大正数。最小的负数是 -Number.MAX_VALUE。   
* **Number.MIN_SAFE_INTEGER：JavaScript 中最小的安全整数 (-(2^53 - 1))**
* Number.MIN_VALUE：能表示的最小正数即最接近 0 的正数 (实际上不会变成 0)。最大的负数是 -Number.MIN_VALUE（最接近0的负数）
* Number.NaN：特殊的“非数字”值。
* Number.NEGATIVE_INFINITY：特殊的负无穷大值，在溢出时返回该值。
* Number.POSITIVE_INFINITY：特殊的正无穷大值，在溢出时返回改值。

需要注意的点：
1. Number.MAX_SAFE_INTEGER 与 Number.MAX_VALUE 不一样。前者是最大安全整数，后者是js能表示的最大数字。Number.MIN_SAFE_INTEGER 与 Number.MIN_VALUE 同理

### 静态方法
Number.isNaN(): 确定传递的值是否是 NaN。
Number.isFinite()：确定传递的值类型及本身是否是有限数。
Number.isInteger()：确定传递的值类型是“number”，且是整数。
Number.isSafeInteger()：确定传递的值是否为安全整数 ( -(2^53 - 1) 至 2^53 - 1之间)。
Number.toInteger()：计算传递的值并将其转换为整数 (或无穷大)。
Number.parseFloat()：和全局对象 parseFloat() 一样。
Number.parseInt()：和全局对象 parseInt() 一样。

### 类方法
Number.prototype.toExponential(n: number) 返回一个科学计数法表示的字符串
Number.prototype.toFixed(n: number) 对参数进行四舍五入，返回整数
Number.prototype.toLocaleString(n: number)：默认调用toString()方法，用来被开发者自定义toString方法
Number.prototype.toPrecision(n: number)：指定保留n位数字，四舍五入后返回一个字符串。
Number.prototype.toString(n: number = 10)：转换成n进制的数字，用字符串形式返回。默认是10进制 
Number.prototype.valueOf()：返回Number对象内部的值

# js数字的内部存储方式，以及安全整数的概念
## IEEE 754 64位双精度浮点数
js中所有的数字都使用IEEE 754标准定义的64位双精度浮点格式来实现，java中double类型也采用这个标准进行实现。   
IEEE 754标准定义的64位双精度浮点格式的相关资料如下，这里不再赘述。
* [IEEE 754百度百科](https://baike.baidu.com/item/IEEE%20754/3869922?fr=aladdin)
* [IEEE 754表示浮点数](https://www.jianshu.com/p/e5d72d764f2f)

这里需要记住几个概念和弄明白IEEE 754如何表示浮点数

计算机内浮点数表示示意图：
<img src="/img/post/post9/float.webp" alt="计算机内浮点数表示示意图">
上面图中， bbbb...表示尾数，p表示指数。

IEEE754 64位双精度浮点数采用1位记录正负符号，11位记录指数，52位记录尾数
下面是2^53 2^53-1 2^53-2 三个数字的表示

| 表示数字 | 符号位（1位） | 指数位（11位） | 尾数位（52位）|
|-------------|--------------|--------------|----------|
| 2^53 | 0（0代表正号）| 53           | 1.0000...000（小数点后一共52个0）|
|2^53 -1 | 0 | 52 | 1.111...111（小数点后一共52个1）|
|2^53-2 | 0 | 52 | 1.111...110 （小数点后一共51个1,1个0）|
| 2^0 | 0 | 0 | 1.000...00 （小数点后52个0） | 

0的表示需要特殊处理，这里不再赘述。

依此类推，可以看到这个过程中，每向下减一，通过修改尾数和指数（有时指数 不变），每一个整数都可以由一个浮点数‘模拟’。

## 带来的问题
由于IEEE754 64位浮点数的表示方式，会导致以下一些问题。注意这些问题是由标准导致的，所以只要实现了这个标准就会出现这样的问题，而并不是js自己的问题。


1、精度问题：十进制的数字在转换为浮点数表示时，可能出现尾数多于52位的现象，此时必然会把尾数多于52位的部分省略。此时会造成精度问题。
```javascript
// 0.1转换为双精度浮点数无法精确表示。
// 下面加法发生的三个过程
// 1、将0.1 转换为双精度浮点数 （无法精确表示，发生精度丢失）（0.1转换成二进制是 0.0 0011 0011 0011 ...无限循环） 
// 2、将0.2 转换为双精度浮点数 （无法精确表示，发生精度丢失）（0.2转成二进制是 0. 0011 0011 0011 ...无限循环）
// 3、两个双精度浮点数相加，然后再转换为十进制显示。因为相加的两个浮点数都只是十进制的近似值，所以相加后的浮点数不是严格的0.3，只是0.3的近似
0.1 + 0.2 // 输出 0.30000000000000004

0.1 + 0.2 === 0.3 // 输出false

0.3 - 0.2 // 输出 0.09999999999999998。因为0.2无法精确表示，发生进度丢失导致。
```

2、安全整数：由于所有整数实际上都由双精度浮点数表示，会导致当**超过一定范围后，出现多个整数的二进制存储位数一致的现象**，即整数和二进制存储不再是一对一的情况了，此时四则运算无法正常进行。这个范围内的整数称之为“安全整数”。出现这种现象，本质上其实还是浮点数精度缺失引发的问题。   
64位双精度浮点数的安全整数范围是-(2^53-1) ~ 2^53-1，包含边界。

下面解析为什么从2^53开始，不是安全整数。将2^53 和 2^53 +1 两个数字都用二进制指数化表示，**不考虑尾数位的限制**。

| 表示数字 | 符号位（1位） | 指数位（11位） | 尾数位（52位）|
|-------------|--------------|----------|---------|
| 2^53 | 0（0代表正号）| 53 | 1.0000...0000（小数点后一共53个0）|
| 2^53 + 1 | 0 | 53 | 1.000...0001（小数点后52个0,1个1）|

可以发现，上面两个数字在转换为二进制指数表示时，唯一的差别是尾数部分小数点后53位。但是64位双精度浮点数的尾数只有52位，所以实际上53位的差异被舍弃了。所以 2^53 和 2^53 + 1两个数字的二进制存储位数是一样的，所以它们不是安全整数。

js中通过 Number.MAX_SAFE_INTEGER 和 Number.MIN_SAFE_INTEGER 可以获取最大和最小安全整数。

## 如何解决
确定自己需要的精度，放大相应的倍数。比如货币单位运算，使用分进行计算。