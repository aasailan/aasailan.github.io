---
title: mvc mvp mvvm模式总结
date: 2019-02-14 10:12:16
categories:
  - js
tags: 
  - 前端
  - javascript
  - 设计模式
comments: false
---
<!-- post8 -->
# 对mvc mvp mvvm模式的总结

## 为什么需要mv* 模式

### 交织在一起的UI代码和业务代码
最早的程序是汇编，那时候并没有UI界面，所谓的开发几乎都是用汇编编写一些机器指令，使用程序的也大多数是一些专业人士。哪个年代自然也不需要mv* 的模式。  
随着行业发展，程序开始面向普通用户，程序也出现了用户界面（User Interface）和适用于界面的语言——标记语言（Markup Language）。普通用户通过操作UI来操作程序。这时程序内部的代码可以大概分为两部分：操纵UI和执行业务逻辑。在没有仔细规划的程序中，这两部分代码交织在一起，常常造成代码难以维护。  

### 分离展示层
根据上面的问题，GUI应用程序可以粗略的划分为两个部分：
1. Presentation Layer：用于展示内容的展示层
2. Domain Layer：包含数据和逻辑
<img src="/img/post/post8/Presentation-Domain.jpg" alt="Presentation 与 Domain">


继续细分下去，GUI应用程序在一个时间点上通常可以抽象为四个部分：
* 界面
* 数据
* 事件
* 业务   

其中的联系一般为：
<img src="/img/post/post8/GUI_app.png" alt="GUI程序中各个因素的联系">


这样抽象过后，便会发现UI程序面临着两个主要问题：
1. 操作UI和执行业务的代码需要解耦和分开处理，以便提升可维护性
2. 如何更新数据和更新界面并且维护它们之间的一致性。     

为了解决上述问题，开始出现了MV*模式。

## MVC模式
在20世纪70年代[**Smalltalk-80**](https://zh.wikipedia.org/wiki/Smalltalk)实现了MVC设计模式。Smalltalk-80实现的MVC模式有四个要点：
* A Model represented domain-specific data and was ignorant of the user-interface (Views and Controllers). When a model changed, it would inform its observers.
  model代表着领域层（Domain Layer）的数据，并且独立于UI，当model变化时，会通知它的观察者们。
* A View represented the current state of a Model. The Observer pattern was used for letting the View know whenever the Model was updated or modified.
  view显示着当前model的状态。通过观察者模式，view（充当Observer）可以知道model（充当Subject）的修改并且更新自己
* Presentation was taken care of by the View, but there wasn't just a single View and Controller - a View-Controller pair was required for each section or element being displayed on the screen.
  展示层通常由一个 view-Controller对实现，由这个view-Controller对来实现当前屏幕每一个元素的显示
* The Controllers role in this pair was handling user interaction (such as key-presses and actions e.g. clicks), making decisions for the View.
  控制器主要负责处理UI事件，并且决定view如何更新

这样形成的MVC模式关系图如下：  
<img src="/img/post/post8/mvc.png" alt="mvc模式关系图">    
①②：view 在创建时observer model并且运行时将用户操作传递给Controller（通常通过event的形式）   
③：controller对业务逻辑进行执行，期间可能需要修改model   
④：model被修改后notify所有的observers，view在收到通知后对自己进行更新


其中各个角色的定义和描述如下：

### model
 Models manage the data for an application. They are concerned with neither the user-interface nor presentation layers but instead represent unique forms of data that an application may require. When a model changes (e.g when it is updated), it will typically notify its observers (e.g views, a concept we will cover shortly) that a change has occurred so that they may react accordingly.    
 model管理着应用的数据，它本身独立于UI和展示层。当model的数据变化时，需要通知它的所有observers，model已经发生变化，使得observers能做出反应。

 ### view
 Views are a visual representation of models that present a filtered view of their current state. Whilst Smalltalk views are about painting and maintaining a bitmap, JavaScript views are about building and maintaining a DOM element. A view typically observes a model and is notified when the model changes, allowing the view to update itself accordingly.    
 view是model的虚拟展示，显示model的当前状态。在js中，view层通常负责创建和维护DOM元素。一个view通常会observer一个model，并且在model改变发出通知时，更新自己。

 ### controllers
 Controllers are an intermediary between models and views which are classically responsible for updating the model when the user manipulates the view. 
 controllers是model和view之间的中介，负责响应用户对view的操作和更新model。

 下面是MVC模式在js中的粗略实现。需求描述，页面存在add和remove两个按钮，点击add按钮在页面添加一张demo image，点击remove删除一张demo image。
 ```javascript
function createView(model, ctrl) {
  var _addButton, _removeButton, _container;
  // view 维护dom元素逻辑
  function _addPhone(data) {
    var newPhone = data.slice(-1);
    var img = document.createElement('img');
    img.src = newPhone.src;
    _container.appendChild(img);
  }
  // view 维护dom元素逻辑
  function _removePhone(data) {
    container.removeChild(container.lastElementChild);
  }
  return {
    init: function() {
      _addButton = document.getElementById('add-btn');
      _removeButton = document.getElementById('remove-btn');
      _container = document.getElementById('container');
      // 向model注册监听，并且在
      model.subscribe('add', _addPhone)
      model.subscribe('remove', _removePhone)

      _addButton.addEventListener('click', this.addPhone);

      _removeButton.addEventListener('click', this.removePhone);
    },
    destroy: function() {
      var _addButton = document.getElementById('add-btn');
      _addButton.removeEventListener('click', this.addPhone);
      _removeButton.removeEventListener('click', this.removePhone);

      model.unsubscribe('add', _addPhone);
      model.unsubscribe('remove', _removePhone);

      _addButton = null;
      _removeButton = null;
      _container = null;
    },
    addPhone: function() {
      ctrl.addPhone()
    },
    removePhone: function() {
      ctrl.removePhone();
    }
  }
};

function createModel(data) {
  var _events = {};
  var _phones = data;
  return {
    subscribe: function(event, subscriber) {
      if (!_events[event]) {
        _events[event] = [];
      }
      _events[event].push(subscriber);
    },
    unsubscribe: function(event, subscriber) {
      if (!_events[event]) {
        return;
      }
      var subscribers = _events[event];
      var index = subscribers.indexOf(subscriber);
      if (index !== -1) {
        subscribers.splice(index, 1);
      }
    },
    publish: function(event, modelData) {
      if (!_events[event]) {
        return;
      }
      var subscribers = _events[event];
      subscribers.forEach(subscriber => {
        subscriber(modelData);
      });
    },
    addPhone: function(phone) {
      _phones.push(phone);
      this.publish('add', _phones);
    },
    removePhone: function() {
      _phones.pop();
      this.publish('remove');
    }
  }
};

function createController(model) {
  return {
    addPhone: function() {
      console.log('用户添加了一张图片');
      model.addPhone('demo.jpg');
    },
    removePhone: function() {
      model.removePhone();
    }
  }
};

// 开始运行
var model = createModel([]);
var controller = createController(model);
var view = createView(model, controller);
view.init();
// 用户操作 ....
view.destroy();
 ```

MVC模式带来的好处：
1. 更好的可维护性。
2. model和view的解耦使得单元测试可以更好的进行
3. 将业务逻辑和用户图形接口分开，使得开发者可以更好的关注业务逻辑

MVC模式的问题：
1. 程序有变化时，往往需要维护三个对象和三个交互。

为了解决这个问题，MVC模式产生了一个变种MVP模式。

## MVP模式
MVP模式中切断了View和Model的联系，View不再是一个Observer，成为一个**只提供更新视图接口**并等待调用更新的“被动”视图。Presenter充当了原来controller的大部分职责并且负责主动调用更新view。

MVP描述如下：   
The P in MVP stands for presenter. It's a component which contains the user-interface business logic for the view. Unlike MVC, invocations from the view are delegated to the presenter, which are decoupled from the view and instead talk to it through an interface. This allows for all kinds of useful things such as being able to mock views in unit tests.    
The most common implementation of MVP is one which uses a Passive View (a view which is for all intents and purposes "dumb"), containing little to no logic. If MVC and MVP are different it is because the C and P do different things. In MVP, the P observes models and updates views when models change. The P effectively binds models to views, a responsibility which was previously held by controllers in MVC.    
MVP中的P指presenter。与MVC模式不一样，MVP中的view通过提供接口的方式将自己的调用完全委托给presenter。这对view的单元测试很有帮助。    
在MVP模式中通常会实现一个“Passive View”。在MVP中P负责observes model和调用view的接口来更新view并且负责以前Controller的责任。

MVP模式关系图如下：   
<img src="/img/post/post8/mvp.png" alt="mvc模式关系图">
① view层将用户操作委托给 presenter处理
② presenter处理业务逻辑，修改model
④ model notify presenter数据更新
④ presenter根据view提供的接口进行调用来更新view


使用MVP模式修改上面MVC的例子
```javascript
```

MVP的好处：
1. view与model进行了解耦，view包含较少的逻辑，只关注与对外提供接口更新界面。
2. 由于view的功能单一并且与model的解耦，view变得非常容易进行测试。

## MVVM
2005年的时候微软首次提出[**MVVM模式**]，并且在WPF平台进行了实现(https://blogs.msdn.microsoft.com/johngossman/2005/10/08/introduction-to-modelviewviewmodel-pattern-for-building-wpf-apps/)

MVVM的描述：
MVVM (Model View ViewModel) is an architectural pattern based on MVC and MVP, which aempts to more clearly separate the development of user-interfaces (UI) from that of the business logic and behavior in an application. To this end, many implementations of this pattern make use of declarative data bindings to allow a separation of work on Views from other layers.    
MVVM是基于MVC和MVP的变种，目的是让UI界面和业务逻辑进步一解耦。为了让View和其他层解耦，很多MVVM的实现都使用了 **声明式数据绑定** 这项技术。

MVVM由三部分组成Model、view、ViewModel。
* Model任然代表应用的数据
* view依然代表视图展示
* ViewModel与Controller和Presenter都不同，ViewModel不再单纯的只负责执行业务逻辑和更新view，而是成为**视图的一个抽象**（这是之前的两种模式所没有涉及的）。视图本身是有状态和行为的，视图的状态通常来自于model的数据并且和model的数据保持同步，而视图的行为通常是执行业务逻辑。ViewModel作为视图的抽象，拥有自己的状态属性（代表当前视图状态）和行为方法（当前视图对用户的响应逻辑，通常会执行业务逻辑）。ViewModel需要维护自身状态与model的一致性、view层与自身的一致性、执行业务逻辑

ViewModel与View之间的一致性：在mvvm中，通常使用 **数据绑定** 来维护view和 ViewModel 之间的一致性。数据绑定 通常由 声明式数据绑定（语法表现形式）和 隐藏的Binder层（实现声明式数据的具体实现）两部分实现。
<img src="/img/post/post8/Binder-View-ViewModel.png" alt="双向数据绑定">

<img src="/img/post/post8/mvvm.png" alt="mvvm的关系模式">

MVVM中view和ViewModel之间没有了MVP的界面接口，而是使用了数据绑定的形式，虽然提升了实现的难度，但却解决了 **用一种统一的集中的方式实现频繁需要被实现的数据更新** 问题。MVVM中最重要的不是如何同步view和ViewModel、ViewModel和model之间状态，而是mvvm创建了一个视图的抽象，让一个视图拥有了自己的状态和行为，这也更符合开发者对视图的思维逻辑。