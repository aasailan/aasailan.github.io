---
title: js构造函数与继承
date: 2019-02-11 10:05:48
categories:
  - js
tags: 
  - 前端
  - javascript
comments: false
---
# js构造函数
js中定义一个function，当使用new关键字调用这个function的时候，这个function成为一个构造函数。如下所示：
```javascript
function Person(name) {
  this.name = name;
}
// 使用new关键字调用Person函数，Person函数成为构造函数
var xiaoming = new Person();
```

## 使用new关键字时内部细节
### 使用new关键字时的关键处理（构造函数的内部处理）
当使用new关键字调用构造函数生成一个对象实例时，js做出的关键处理如下：   
1. 创建一个新的对象，将构造函数内this指针指向新建对象。
2. 将新建对象的__protp__属性设置成构造函数的prototype属性，确保新建对象是构造函数实例
3. 运行一遍构造函数，如果构造函数本身需要返回一个object或者array对象，则舍去新创建对象，返回构造函数需要返回的对象;否则返回新建对象
### 使用es5模拟new关键字调用构造函数
```javascript
/**
 * @bug 这里假设new出来实例的都是object，不是array对象
 * @description 传入构造函数，以及构造函数参数，模拟构造函数返回一个新建实例
 * @param {function} constructor 需要传入的构造函数
 * @param {any} params 跟在constructor参数后面的所有参数，用来模拟传入constructor的参数
 * @returns 返回一个新的对象
 */
function mockNew(constructor) {
  // 新建一个对象

  var obj = {};
  // 修改对象的__proto__ 确保使用instanceof关键字时没有问题
  obj.__proto__ = constructor.prototype;

  // 将构造函数的this指针绑定到obj，然后调用
  var params = Array.prototype.splice.call(arguments, 1);
  var result = constructor.apply(obj, params);

  // 判断构造函数是否返回一个对象，是的话直接返回构造函数返回的对象
  if (typeof result === 'object' || typeof result === 'array') {
    return result;
  }

  return obj;
}

// 定义一个构造函数
function Person(name, age) {
  this.name = name;
  this.age = age;
} 

// 使用mockNew调用构造函数模拟new关键字
var xiaoming = mockNew(Person, 'xiaoming', 26);
```
测试截图如下:   
定义两个构造函数，分别返回一个对象和不返回对象，并且分别使用mockNew调用和new关键字调用。
测试代码如下：
```javascript
function Person(name, age) {
  this.name = name;
  this.age = age;
}

function Person2(name, age) {
  this.name = name;
  this.age = age;
  var result = {};
  return result;
}

// 使用mock函数
console.log('============ xiaoming start ==============');
var xiaoming = mockNew(Person, 'xiaoming', 26);
console.log(xiaoming);
console.log(xiaoming instanceof Person);

console.log('============ xiaohong start ==============');
var xiaohong = mockNew(Person2, 'xiaohong', 26);
console.log(xiaohong);
console.log(xiaohong instanceof Person2);

// 使用new关键字
console.log('============ xiaoqiao start ==============');
var xiaoqiao = new Person('xiaoqiao', 26);
console.log(xiaoqiao);
console.log(xiaoqiao instanceof Person);

console.log('============ xiaobing start ==============');
var xiaobing = new Person2('xiaobing', 26);
console.log(xiaobing);
console.log(xiaobing instanceof Person2);
```
输出截图如下：   
<img src="/img/post/post6/test1.png" alt="mockNew测试用例">

## 使用es5实现安全的构造函数
编写构造函数后，如果调用者不使用new关键字调用，可能会产生意想不到的效果，因此需要对这种情况做出一定处理，处理方式一般有两种：  
1. 对不使用new关键字的调用抛出error，提醒需要使用new关键字
2. 对不使用new关键字的调用做出兼容处理，同样返回正常的新建对象

**两种方法的关键在于在构造函数中判断this指针是否是构造函数的实例，然后做出处理**

方式1代码如下：
```javascript
// 抛出error
function Person(name, age) {
  if (!(this instanceof Person)) {
    throw new Error('constructor must be invoked by "new" keyword');
  }
  this.name = name;
  this.age = age;
}

// 测试用例，以下调用都会抛出error
var xiaoming = Person('xiaoming', 26);
var xiaohong = Person.call(Object, 'xiaohong', 26);
```
方式2代码如下：
```javascript
// 正常执行返回
function Person(name, age) {
  if (!(this instanceof Person)) {
    return new Person(name, age);
  }
  this.name = name;
  this.age = age;
}

// 测试用例，以下调用都会正常返回Person实例
var xiaoming = Person('xiaoming', 26);
var xiaohong = Person.call(Object, 'xiaohong', 26);
```

## es5实现私有属性与静态属性
私有属性的特点在于无法让外界直接访问，js中可借助闭包的方式实现
```javascript
// 私有属性
function Person(name, age) {
  if (!(this instanceof Person)) {
    return new Person(name, age);
  }
  this.name = name;
  this.age = age;

  // 私有属性 这种方式实现的私有属性有两个缺点
  // 1、需要访问私有属性的方法只能定义在构造函数内
  // 2、访问私有属性时无法用this指针进行引用，因为私有属性本质不在创建实例上
  var sex = Person.SEX.woman;
  this.setSex = function(newSex) {
    sex = newSex;
  }
  this.getSex = function() {
    return sex;
  }
}

// 类静态属性
Person.SEX = {
  man: 'man',
  woman: 'woman',
};

// 测试用例
var xiaoming = new Person('xiaoming', 26);
console.log(xiaoming.name); // xiaoming
console.log(xiaoming.sex); // undefine
console.log(xiaoming.getSex()); // woman
xiaoming.setSex(Person.SEX.man); 
console.log(xiaoming.getSex()); // man

```

# js继承
使用js实现继承时，如果用es6的语法则很简单，直接使用class 与 extend 关键字即可
```javascript
// 类
class Person {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
  sayName() {
    console.log('I am ' + this.name);
  }
}

// 继承
class Employee extends Person {
  constructor(name, age, department) {
    super(name, age);
    this.department = department;
  }
  sayDepartment() {
    console.log('I am in ' + this.department);
  }
}

var xiaoming = new Employee('xiaoming', 26, '研发中心');
xiaoming.sayName(); // I am xiaoming
xiaoming.sayDepartment(); // I am in 研发中心
console.log(xiaoming instanceof Employee); // true
console.log(xiaoming instanceof Person); // true
console.log(xiaoming);
```
输出截图如下：
<img src="/img/post/post6/class.png" alt="es6 class继承展示">

下面使用es5实现继承，继承需要注意两个点：  
1. 分别继承类的属性和类的方法
2. new出的实例应该是其父类以及继承的祖先类的实例，使用instanceof 关键字判断时返回true

如果对instanceof关键字的执行原理不明白，可参照[**JavaScript instanceof 运算符深入剖析**](https://www.ibm.com/developerworks/cn/web/1306_jiangjj_jsinstanceof/index.html)

## 组合式继承   
组合式继承：组合式继承顾名思义这种**继承有两个部分组合在一起**实现：构造函数中实现属性继承；构造函数的prototype属性中实现方法继承。这两个组合在一起实现继承

组合式继承的两个要点：   
1. 在构造函数中调用父类构造函数，实现父类属性继承
2. 使用父类的实例设置子类构造函数的的prototype属性，实现父类方法的继承

```javascript
// 定义父类
function Person(name, age) {
  if (!(this instanceof Person)) {
    return new Person(name, age);
  }
  this.name = name;
  this.age = age;
}
Person.prototype.sayName = function() {
  console.log('I am ' + this.name);
}

// 定义子类
function Employee(name, age, department) {
  if (!(this instanceof Employee)) {
    return new Employee(name, age, department);
  }

  // 关键1 调用父类构造函数，实现属性继承
  Person.call(this, name, age);

  this.department = department;

  this.sayDepartment = function() {
    this.sayName();
    console.log('I am in ' + this.department);
  }
}
// 关键2 使用父类实例设置子类构造函数prototype属性，实现方法继承
Employee.prototype = new Person();
// 修正子类构造函数的constructor属性，非必须
Employee.prototype.constructor = Employee;

// 测试用例
var xiaoming = new Employee('xiaoming', 26, '研发中心');
console.log(xiaoming.name); // xiaoming
console.log(xiaoming.department); // 研发中心
xiaoming.sayName(); // I am xiaoming
xiaoming.sayDepartment(); // I am xiaoming \n I am in 研发中心

console.log(xiaoming instanceof Person); // true
console.log(xiaoming instanceof Employee); // true

console.log(xiaoming);
```
输出截图如下：
<img src="/img/post/post6/es5_extends.png" alt="es5组合式继承">

组合式继承的缺点
1. 在子类实例的原型上多了许多不必要的父类属性，浪费内存

## 寄生组合式继承
寄生组合式继承：顾名思义，与组合式继承一样，继承来自于两部分继承的组合，区别在于对子类prototype的处理，为了避免组合式继承的缺点，将父类prototype对象处理过后赋值给子类构造函数。

寄生组合式继承的两个要点：   
1. 在构造函数中调用父类构造函数，实现父类属性继承
2. 使用父类的prototype设置子类构造函数的prototype属性，实现父类方法的继承

```javascript
function Person(name, age) {
  if (!(this instanceof Person)) {
    return new Person(name, age);
  }
  this.name = name;
  this.age = age;
}
Person.prototype.sayName = function() {
  console.log('I am ' + this.name);
}

function Employee(name, age, department) {
  if (!(this instanceof Employee)) {
    return new Employee(name, age, department);
  }

  // 关键1 调用父类构造函数，实现属性继承
  Person.call(this, name, age);

  this.department = department;
}
// 关键2 使用父类的prototype设置子类构造函数prototype属性，实现方法继承
Employee.prototype = Object.create(Person.prototype, {
  // 修复constructor，非必须
  constructor: {
    value: Person,
    enumerable: false,
    writable: true,
    configurable: true
  }
});
Employee.prototype.sayDepartment = function() {
  this.sayName();
  console.log('I am in ' + this.department);
}


// 测试用例
var xiaoming = new Employee('xiaoming', 26, '研发中心');
console.log(xiaoming.name); // xiaoming
console.log(xiaoming.department); // 研发中心
xiaoming.sayName(); // I am xiaoming
xiaoming.sayDepartment(); // I am xiaoming \n I am in 研发中心

console.log(xiaoming instanceof Person);
console.log(xiaoming instanceof Employee);

console.log(xiaoming);
```
输出截图如下：
<img src="/img/post/post6/es5_extends2.png" alt="es5寄生组合式继承">

# 多继承（mixin）
js中事实上无法实现多继承，只能通过mixin的手段来模拟，这种方式可以实现继承多个类的属性、方法。但是无法使用instanceof关键字做出正确判断。事实上这种方式不叫多继承，称之为mixin

mixin的思想是将需要复用的属性或者方法**混入**目标类构造函数的prototype对象中

```javascript
// js mixin实现
```
