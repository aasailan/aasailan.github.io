---
title: js设计模式总结
date: 2019-02-12 10:06:52
categories:
  - js
tags: 
  - 前端
  - javascript
  - 设计模式
comments: false
---
<!-- post7 -->
# 设计模式概要
## 什么是设计模式：   
A pattern is a reusable solution that can be applied to commonly occurring problems in software design.  Another way of looking at pattern are as templates for how we solve problems - ones which can be used in quite a few different situations.    

## 设计模式的好处：
  1. pattern are proven solutions（设计模式是被证明过的解决方案）: They provide solid approaches to solving issues in software development using proven techniques that reflect the experience and insights the developers that helped define them bring to the paern. 
  2. pattern can be easily reused（设计模式易被重复使用）: A paern usually reflects an out of the box solution that can be adapted to suit our own needs. This feature makes them quite robust. 
  3. pattern can be expressive（设计模式富有表现力）: When we look at a paern there’s generally a set structure and vocabulary to the solution presented that can help express rather large solutions quite elegantly. 

## 设计模式的由来
最早的时候有一位叫 Christopher Alexander.的建筑师，总结了自己在建筑领域的一些经验并且出版了相关的出版物，这些经验在建筑领域起到了很好的效果。   
后来一些软件工程师汲取了 Alexander的经验，把一些设计模式写到文件中作为开发者们提升编程技巧的指导，设计模式这个概念开始在编程领域流行。   
1995年，Erich Gamma, Richard Helm, Ralph Johnson and John Vlissides 这四个人出版了著名的书籍: Design pattern: Elements Of Reusable Object-Oriented Software 。 这四个人被称为 Gang of Four (or GoF for short).   
这本书中提出了23种面向对象的设计模式，并且对这23中设计模式做了分类：   
* 创建型设计模式（Creational）：以创建对象为目标的设计模式
  * 工厂模式（factory）
  * 抽象工厂模式（Abstract Factory）
  * 建造者模式（builder）
  * 原型模式（prototype）
  * 单例模式（singleton）
* 结构型设计模式（Structural）
  * 适配器模式（adapter）
  * 桥接模式（Bridge）
  * 组合模式（Composite）
  * 外观模式（Facade）
  * 享元模式（Flyweight）
  * 代理模式（proxy）
* 行为型设计模式（Behavioral）  
  * 解释器模式（Interpreter）
  * 模板方法模式（template method）
  * 职责链模式（Chain of  Responsibility ）
  * 命令模式（Command）
  * 迭代器模式（Iterator）
  * 中介者模式（Mediator）
  * 备忘录模式（Memento）
  * 观察者模式（Observer）
  * 状态模式（state）
  * 策略模式（strategy）
  * 访问者模式（visitor）

## 设计模式在不同语言中的区别
Design pattern 这本书大部分代码由C++写成，其讲述的也是基于静态类型语言的设计模式。在C++，java等静态语言中，函数作为逻辑运算代码块无法单独传递，只能依附于某一个对象，作为该对象的方法进行传递，因此在这些静态语言中面向对象编程几乎成为必然。Design pattern 这本书完全是从**面相对象设计**的角度出发，通过对封装、继承、多态、组合等技术的反复使用，提炼出一些**可重复使用的面向对象的设计技巧**。   
在js等动态语言中，有两个特性与静态语言显著不同：1、js中函数是一等公民，可以直接传递函数引用而无需依附在某个对象。2、js中可以动态修改对象。这两个明显不同导致Gof提出的一些设计模式在动态语言中出现了变种或者大相径庭的实现方式。比如Command模式在java中通常需要一个命令类、一个接受类、一个调用者类，逻辑运算函数依附在命令类对象中四处传递。然而在js中可以直接传递函数引用，这导致命令类成为非必要的存在，因为只要传递函数就可以了。

## 分辨模式的关键在于意图而不是结构
设计模式中有一些设计模式的代码结构粗看起来几乎一模一样，让人分不清有啥区别，实际上这是普遍情况。辨别模式的关键是这个模式出现的场景，以及为我们解决了什么问题。

## 本文参考资料概要
本人主要阅读了三本关于js设计模式的书籍，这篇文章是对三本书做出的截取和总结，本文中许多文字和例子截取自这三本书籍，并且加上一些个人理解和总结。以下是本人对三本书籍优缺点的一些总结：
* Learning JavaScript Design pattern：Addy Osmani 编写，本书讲述清晰简短，比较权威，由于设计模式在网上相关的文章太多，杂乱无章，因此本文尽量以这本英文原版书籍为主。本书已经在github上开源，非常值得一读。仓库地址：https://github.com/addyosmani/essential-js-design-patterns 
* javascript设计模式：张容铭著，这本书讲述简短，适合入门和快速阅读。但是其中一些设计模式的例子编写得并不好，并且书中强行使用java的思维写js，使得一些例子看起来非常牵强，容易让人对一些设计模式产生误解。
* javascript设计模式与开发实践：曾探著，这本书讲解深刻，讲解了一些设计模式在js和其他静态语言中的区别，给出的代码示例比较贴合，但是因为讲解详细，篇幅较长，适合细读。

# js设计模式总结
以下是本人看完三本书后，对一些js设计模式的总结和理解

## 创建型设计模式
Creational design pattern focus on handling object creation mechanisms where objects are created in a manner suitable for the situation we're working in. The basic approach to object creation might otherwise lead to added complexity in a project whilst these pattern aim to solve this problem by controlling the creation process.    
创建型设计模式关注于如何在恰当的环境下以恰当的方式生成所需要的对象。一些创建对象的基本方式（例如直接使用new + constructor，这种创建方式会在系统中到处存在，而当某个constructor需要改动时，需要修改系统内所有使用了这个constructor的地方）可能会增加系统的复杂度。创建型设计模式也可以解决这类问题。

### 工厂模式（Factory）
工厂模式：工厂模式就是利用一个工厂对象（方法），来生成需要的对象，避免了直接使用new+构造函数的方式来生成对象，同时工厂方法生成目标对象的过程可以自由控制，来按需生成对象。

特点：
* 目的是为了生成对象
* 控制生成过程，按需生成

```javascript
function Car( options ) {
  this.doors = options.doors || 4;
  this.state = options.state || "brand new";
  this.color = options.color || "silver";
}

function Truck( options){
  this.state = options.state || "used";
  this.wheelSize = options.wheelSize || "large";
  this.color = options.color || "blue";
}
   
// 工厂类
function VehicleFactory() {}
// Our default vehicleClass is Car
VehicleFactory.prototype.vehicleClass = Car;
// 工厂类工厂方法
VehicleFactory.prototype.createVehicle = function ( options ) {
  switch(options.vehicleType){
    case "car":
      this.vehicleClass = Car;
      break;
    case "truck":
      this.vehicleClass = Truck;
      break;
  }
  return new this.vehicleClass( options ); 
};
  

var carFactory = new VehicleFactory();
var car = carFactory.createVehicle({
  vehicleType: "car",
  color: "yellow",
  doors: 6 
});

console.log( car instanceof Car ); // true
console.log( car );
```

### 抽象工厂（Abstract Factory）
抽象工厂：抽象工厂依然是工厂模式的样子，只是它与它所创建的对象更加**解耦**。当一个系统与它所创建对象的创建方式是隔离的，或者这个系统需要创建许多不同类型的对象，这个时候可以考虑抽象工厂。抽象工厂在生成时并不清楚它能产生哪些对象，只是用一组规则描述它能产生哪些类型的对象。

```javascript
// 生成抽象工厂函数，此时抽象工厂不清楚自己能生成哪些对象
var abstractVehicleFactory = (function () {

  var types = {};
  
  return {
    getVehicle: function ( type, customizations ) {
      var Vehicle = types[type];
      return (Vehicle ? new Vehicle(customizations) : null);
    },

    // 动态注册生成对象
    registerVehicle: function ( type, Vehicle ) {
      var proto = Vehicle.prototype;
      // 用一组规则描述它能产生哪些类型的对象
      if ( proto.drive && proto.breakDown ) {
          types[type] = Vehicle;
      }
      return abstractVehicleFactory;
    }
  };
})();

// 动态注册，Car， Truck类的定义省略不写...
abstractVehicleFactory.registerVehicle( "car", Car );
abstractVehicleFactory.registerVehicle( "truck", Truck );
   
// 生成对象
var car = abstractVehicleFactory.getVehicle( "car", {
  color: "lime green",
  state: "like new" 
});
   
// 生成对象
var truck = abstractVehicleFactory.getVehicle( "truck", {
  wheelSize: "medium",
  color: "neon yellow" 
});

console.log(car instanceof Car); // true
console.log(truck instanceof Truck) // true
```

### 原型模式（prototype）
GoF 提出的原型模式本是指将一个已经存在的object作为模板，通过clone的手段来生成一个新的对象。参照java的实现会更加清晰（http://www.runoob.com/design-pattern/prototype-pattern.html
）。在js中，原型模式发生了转变，泛指通过修改对象的\_\_proto\_\_ 属性或者修改构造函数的prototype属性，来达成clone属性与方法的目的，这与js原型继承或者mixin的概念本质是差不多的。

示例1：通过修改对象的__proto__ 属性来clone属性与方法
```javascript
// 定义 模板对象
var myCar = {
  name: "Ford Escort",
 
  drive: function () {
    console.log( "Weeee. I'm driving!" );
  },
 
  panic: function () {
    console.log( "Wait. How do you stop this thing?" );
  }
};
// 创建target对象，通过修改target对象的__proto__ 属性来clone模板对象的属性与方法
var yourCar = Object.create( myCar );

console.log( yourCar.name ); // Ford Escort
```

示例2：通过修改构造函数的prototype属性来clone属性与方法
```javascript
// 定义模板对象
var vehiclePrototype = {
  init: function ( carModel ) {
    this.model = carModel;
  },
  getModel: function () {
    console.log( "The model of this vehicle is.." + this.model);
  }
};
 
// 定义对象生成方法
function vehicle( model ) {
 
  function F() {};
  // 修改构造函数prototype属性，实现对模板对象的方法的clone（与继承本质一样）
  F.prototype = vehiclePrototype;

  var f = new F();
  f.init( model );
  return f;
}
 
var car = vehicle( "Ford Escort" );
car.getModel(); // The model of this vehicle is..Ford Escort 
```

### 单例模式（singleton）
单例模式是指严格约束一个类只有一个实例对象。一个典型的单例模式可以这样实现，通过一个方法来生成单例对象，当该对象不存在时，生成对象并返回，当对象存在时，直接返回对象。   
js中，单例模式通常与namespace的实现联系在一起，利用namespace来为单例对象提供一个全局统一的获取入口，而单例对象作为一个闭包对象存储在namespace中。   
js中的单例模式通常还伴随着lazy generate的概念，即只在首次获取时产生单例对象，否则不产生。
```javascript
// 定义 mySingleton namespace (namespace在js中通常由一个对象实现)
var mySingleton = (function () {
 
  var instance;
  
  // 单例对象生成方法
  function init() {
    // 私有属性
    var privateRandomNumber = Math.random();
  
    return {
      // Public methods and variables
      publicMethod: function () {
        console.log( "The public can see me!" );
      },
  
      publicProperty: "I am also public",
  
      getRandomNumber: function() {
        return privateRandomNumber;
      }
    };
  
  };
  
  return {
    getInstance: function () {
      // lazy generate
      if ( !instance ) {
        instance = init();
      }
      return instance;
    }
  };
})();

var singleA = mySingleton.getInstance();
var singleB = mySingleton.getInstance();
console.log( singleA.getRandomNumber() === singleB.getRandomNumber() ); // true
```

## 结构型设计模式（Structural）
Structural pattern are concerned with object composition and typically identify simple ways to realize relationships between different objects. They help ensure that when one part of a system changes, the entire structure of the system doesn't need to do the same. They also assist in recasting parts of the system which don't fit a particular purpose into those that do.    
结构型设计模式主要关注对象之间的组合结构。结构型设计模式使得当系统中的一部分发生改变时，整个系统不需要有大的改动。

### 外观模式（facade）
外观模式是将底层方法封装，向上提供简单明了的接口，隐藏自身内部的复杂性。比较典型的例子是JQuery封装的一系列dom操作api，内部隐藏了不同浏览器适配的逻辑，只向用户提供统一的简单的api操作。
```javascript
// 使用外观模式封装注册监听函数
function addMyEvent(el, ev, fn){
  if( el.addEventListener ){
    el.addEventListener( ev,fn, false );
  }else if(el.attachEvent){
    el.attachEvent( "on" + ev, fn );
  } else{
    el["on" + ev] = fn;
  }
};

// JQuery的方法 $(el)、 $(el).css()等等
```

### 享元模式（Flyweight）
享元模式是通过共享复用的数据结构或对象，来达到节省应用内存，提升应用效率的目的。   
享元模式在js中主要有两个方面的应用：
1. 数据层面：通过共享数据对象来节约内存
2. dom层面：复用dom元素或者使用事件委托模型（在容器上注册监听事件，然后分发给子元素，避免所有子元素都注册监听事件）

```javascript
<div id="container">
  <div class="toggle" href="#">More Info (Address)
    <span class="info">
      This is more information
    </span></div>
  <div class="toggle" href="#">Even More Info (Map)
    <span class="info">
      <iframe src="..."></iframe>
    </span>
  </div>
</div>
//********************** Snippet 2 **********************//
var stateManager = {
  fly: function () {
    var self = this;
    // 事件委托，在容器上进行监听，避免多个相同的监听浪费内存
    $( "#container" )
    .unbind()
    .on( "click", "div.toggle", function ( e ) {
      self.handleClick( e.target );
    });
  },
  
  handleClick: function (elem) {
    // 复用dom元素，避免重新创建DOM对象
    $(elem).find( "span" ).toggle( "slow" );
  }
};
```

### 装饰器模式（decorator）
传统定义中，装饰器模式是指能在一个系统中动态地为已存在的class添加新的行为。利用这种方式可以在不需要大量改造旧代码的情况下，为系统添加新的功能。这里有两个关键点：
  1. 不改变原有对象（类）的结构的前提下
  2. 添加新的功能（行为）  
  
得益于js是一门动态语言，装饰器模式在js中可以有两种实现方式  
  1. 类似于静态语言，创建一个装饰器对象或者装饰器函数作为代理，去调用**被装饰对象**的方法，然后在代理中添加额外的新功能。
  2. 直接把新功能函数赋值给被装饰对象的同名属性，把旧函数替换掉，而新函数中包含了旧函数的调用以及添加的新功能逻辑

实现方式一：通过调用装饰器函数作为代理，来调用被装饰对象的方法，这种方式与代理模式非常类似
```javascript
function MacBook() {
  this.cost = function () { return 997; };
  this.screenSize = function () { return 11.6; };
}
   
// Decorator 1
function memory( macbook ) {
  // 装饰器函数内保留被装饰对象方法的功能
  var v = macbook.cost();
  // 新增功能
  macbook.cost = function() {
    return v + 75;
  };
}
  
// Decorator 2
function engraving( macbook ){
  var v = macbook.cost();
  macbook.cost = function(){
    return v + 200;
  };
}
   
// Decorator 3
function insurance( macbook ){
  var v = macbook.cost();
  macbook.cost = function(){
      return v + 250;
  };
}
   
var mb = new MacBook();
// 调用装饰器函数，这样就不用修改MacBook类，来达到新增功能的目的
memory( mb );
engraving( mb );
insurance( mb );
```

实现方式2：由于js可以动态修改对象，所以可以直接修改被装饰对象方法的方式来完成新功能
```javascript
function MacBook() {
}
Macbook.prototype.cost = function () { return 997; };
Macbook.prototype.screenSize = function () { return 11.6; };
   
// 保存原方法引用，然后定义新的方法替换旧方法
_cost = Macbook.prototype.cost;
Macbook.prototype.cost = function() {
  var v = _cost.apply(this);
  return v + 75;
}

var mb = new Macbook();
console.log(mb.cost());

// 下面利用AOP的思想完成装饰器模式
// 装饰器函数1
var before = function(fn, beforefn) {
  return function() {
    beforefn.apply(this, arguments);
    return fn.apply(this, arguments);
  }
}
// 装饰器函数2
var after = function(fn, afterfn) {
  return function() {
    fn.apply(this, arguments);
    return afterfn.apply(this, arguments);
  }
}
// 定义原函数
var count = function () {
  console.log(2);
}
// 第一次装饰后，修改原函数变量引用
var count = before(count, function() {
  console.log(1);
});
// 第二次装饰后，修改原函数变量引用
var count = after(count, function() {
  console.log(3);
});

count(); // 1 \n 2 \n 3
```

## 行为型设计模式（Behavioral）
Behavioral pattern focus on improving or streamlining the communication between disparate objects in a system.   
行为型设计模式关注于提高系统内相互独立的对象之间的交流通讯

### 观察者模式（Observe）
The Observer is a design paern where an object (known as a subject) maintains a list of objects depending on it (observers), automatically notifying them of any changes to state.  
观察者模式中存在两种关键对象以及三种关键操作
1. subject对象：维护一系列的观察者对象，提供三种基本操作方式：被订阅（注册监听方法 register），被取消订阅（移除监听方法 remove），触发事件（trigger）。
2. observers对象：业务逻辑执行对象，监听subject对象触发的事件。

观察者模式非常有利于对象之间的解耦
```javascript
function Subject() {
  if (!(this instanceof Subject)) {
    return new Subject();
  }
  this.observerList = [];
}

Subject.prototype.register = function(observer) {
  if (!observer || !observer.update) {
    return;
  }
  this.observerList.push(observer);
}

Subject.prototype.remove = function(observer) {
  var index = this.observerList.indexOf(observer);
  this.observerList.splice(index, 1);
}

Subject.prototype.notify = function() {
  this.observerList.forEach(observer => {
    observer.update && observer.update()
  });
}

function Observer(updatefn) {
  this.update = updatefn;
}

// 测试
var ob1 = new Observer(function() {
  console.log('ob1 udpate');
});

var ob2 = new Observer(function() {
  console.log('ob2 update');
});

var subject = new Subject();
subject.register(ob1);
subject.register(ob2);

subject.notify();
```

#### 发布订阅模式（Publish/Subscribe）
比起观察者模式，js中更常见的是发布订阅模式，发布订阅模式是观察者模式的一个变种，它们之前的不同在于，观察者模式没有自定义事件，当subject对象发出更新通知，所有的observer都需要做出响应，而发布订阅模式实际上是一个自定义的事件系统，通过不同的事件订阅来区分不同的observer。   
发布订阅者模式的两种对象和三个关键方法
1. publisher：维护一系列的subscriber对象，提供三种基本操作方式：被订阅（注册监听方法 subscribe(event, handler)），被取消订阅（移除监听方法 unsubscribe(event, handler)），触发事件（publish(event)）。
2. subscribers，相当于原来的observer对象，通过监听事件进行业务处理

```javascript
function Publisher() {
  this.eventSet = {};
}
Publisher.prototype.subscribe = function(event, subscriber) {
  if (!subscriber || !subscriber.update) {
    return;
  }
  if (!this.eventSet[event]) {
    this.eventSet[event] = [];
  }
  this.eventSet[event].push(subscriber);
}
Publisher.prototype.unsubscribe = function(event, subscriber) {
  if (!event || !this.eventSet[event]) {
    return;
  }
  if (!subscriber) {
    this.eventSet[event] = [];
    delete this.eventSet[event];
  } else {
    var subscribers = this.eventSet[eventSet];
    var index = subscribers.indexOf(subscriber);
    subscribers.splice(index, 1);
  }
}
Publisher.prototype.publish = function(event) {
  if (!event || !this.eventSet[event]) {
    return;
  }
  var subscribers = this.eventSet[event];
  subscribers.forEach(subscriber => {
    subscriber.update && subscriber.update();
  })
}

function Subscriber(updatefn) {
  this.update = updatefn;
}

// 测试用例
var sb1 = new Subscriber(function() {
  console.log('sb1 update');
});
var sb2 = new Subscriber(function() {
  console.log('sb2 update');
});

var publisher = new Publisher();
publisher.subscribe('js', sb1);
publisher.subscribe('java', sb2);

publisher.publish('js');
publisher.publish('java');
```

## 中介者模式
If it appears a system has too many direct relationships between components, it may be time to have a central point of control that components communicate through instead. The Mediator promotes loose coupling by ensuring that instead of components referring to each other explicitly, their interaction is handled through this central point. This can help us decouple systems and improve the potential for component reusability.   
如果一个系统内有大量的组件之间存在直接关联（通常表现为相互持有引用），此时应该设置一个中央节点来控制组件之间的联系。这个中央节点既能保持组件之间的联系（组件之间任然能通信），又能让组件之间充分解耦（让组件不再相互持有引用）。这就是中介者模式，中央节点就是中介者。   
真实世界中 机场控制系统就是典型的中介者模式，机场控制系统接管了所有的通信和业务处理，而不是把通信留给飞机和飞机之间。集中化控制是这个系统的特点，机场控制系统就是其中中介者的角色

使用中介者模式来模拟游戏
```javascript
// 使用中介者模式完成游戏设计
// 游戏规则，玩家分成红蓝两队，某一队的所有玩家都死亡后，另一队所有玩家获胜

// 定义玩家
function Player(name, team) {
  this.name = name;
  this.team = team;
  this.state = Player.STATE.alive;
}
Player.prototype.win = function() {
  console.log(this.name + ' win');
}
Player.prototype.lose = function() {
  console.log(this.name + ' lose');
}
Player.prototype.die = function() {
  this.state = Player.STATE.die;
  this.lose();
  // 每一位玩家只与中介者发生联系
  GameCenter.publish('playerDead', this);
}
Player.TEAM = {
  red: 'red',
  blue: 'blue'
}
Player.STATE = {
  alive: 'alive',
  die: 'die'
}

// 定义游戏中心，中介者
var GameCenter = function(){

  var teams = {
    [Player.TEAM.red]: [],
    [Player.TEAM.blue]: [],
  };
  
  var operations = {
    playerDead: function(player) {
      if (player && player.team && teams[player.team]) {
        var players = teams[player.team];
        if (!players.some(player => player.state === Player.STATE.alive)) {
          var winPlayers = player.team === Player.TEAM.red ? teams[Player.TEAM.blue] : teams[Player.TEAM.red];
          winPlayers.forEach(player => player.win());
        }
      }
    },
    addPlayer: function(player) {
      if (player && player.team && teams[player.team]) {
        teams[player.team].push(player);
      }
    }
  };

  return {
    publish: function(event) {
      params = Array.prototype.splice.call(arguments, 1);
      // 由中介者处理玩家之间的联系
      operations[event] && operations[event].apply(this, params);
    }
  }
}();

// 定义玩家生成工厂
function playerFactory(name, team) {
  var player = new Player(name, team);
  GameCenter.publish('addPlayer', player);
  return player;
};

// 测试用例
var xiaoming = playerFactory('xiaoming', Player.TEAM.red);
var xiaohong = playerFactory('xiaohong', Player.TEAM.red);

var xiaoqiao = playerFactory('xiaoqiao', Player.TEAM.blue);
var xiaobing = playerFactory('xiaobing', Player.TEAM.blue);

xiaoming.die(); // xiaoming lose
xiaohong.die(); // xiaohong lose
// xiaoqiao win
// xiaobing win
```

通过以上例子可以发现，中介者模式通常伴随着观察者模式（发布订阅者模式），这两种模式的区别在于：
1. 观察者（事件）模式基于事件系统让不同的对象进行通信，目的在于让系统耦合松散。而中介者模式使用一个中介者对象进行业务逻辑处理，其余对象都与中介者通信，通信的手段通常使用事件系统，然而不一定就只用事件系统。所以可以说中介者模式通常使用了观察者模式作为通信手段。
2. 关于第三方对象：观察者（事件）模式中，publisher or subject 是一个第三方对象，他们的作用在于提供事件总线来传递事件。而中介者模式中，中介者是第三方对象，它的职责在于处理业务逻辑，保持其余对象之间的联系，为其余对象解耦

## 状态模式

## 策略模式
