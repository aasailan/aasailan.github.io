---
title: Cordova源码剖析二（cordova.js启动流程）
date: 2019-06-23 14:37:53
categories:
  - cordova系列
tags: 
  - 前端
  - cordova
  - 源码剖析
comments: false
---
## android cordova.js的启动流程
在cordovaWebview加载html之后，html会加载cordova.js文件，cordova.js文件中包含了cordova框架在js一侧的启动和初始化流程，这一节就来探究一下cordova.js到底做了什么

## 立即执行函数
cordova.js内定义了一个立即执行函数，这个立即执行函数内完成了cordova框架在js一侧的启动和初始化。
它主要做了以下几件事情：
1. 定义require、define函数
2. 通过define函数定义一系列的cordova modules
3. require 'cordova' modules，并将export的对象添加到window.cordova
4. require 'cordova/init' modules，开始运行初始化代码
```javascript
;(function() {
  // 定义require、define变量
  var require;
  var define;

  // 然后通过一个立即执行函数为require和define变量赋值
  (function() {
    ...
  })();

  ...
  // 接下来通过define函数定义一系列的modules
  define('cordova', ....);

  ....

  // 加载'cordova'模块，导出对象并添加window.cordova对象
  window.cordova = require('cordova');

  // 初始化框架
  require('cordova/init');
})();
```

### 阶段一：初始化define和require函数
在定义了require和define两个变量后，紧接着就是一个立即执行函数，为这两个变量进行赋值。这个立即执行函数非常关键，主要做了以下两件事情：
1. 定义了modules对象，该对象以module id为key，以module对象为value。每一个module对象包括id、factory、exports三个属性
2. 定义了define和require方法，为cordova框架的模块化提供了基础。define方法负责定义cordova module，require方法负责加载cordova module，并且返回module.exports对象

下面深入这个立即执行函数进行查看
```javascript
(function() {
  // NOTE: 关键对象，存储module对象
  var modules = {};
  ...
 /**
  * @description 由require函数调用，运行module的factory函数，对module进行初始化
  * @param {{ id: string, factory: function }} module 需要构建的module对象
  * @returns {object} module.exports
  */
  function build(module) {
    var factory = module.factory;
    // 定义module factory函数内使用的require函数
    var localRequire = function (id) {
      var resultantId = id;
      // Its a relative path, so lop off the last portion and add the id (minus "./")
      if (id.charAt(0) === '.') {
        resultantId = module.id.slice(0, module.id.lastIndexOf(SEPARATOR)) + SEPARATOR + id.slice(2);
      }
      return require(resultantId);
    };
    module.exports = {};
    delete module.factory;
    // 运行factory，在这个函数产生副作用，完成对module.exports的初始化
    factory(localRequire, module.exports, module);
    return module.exports;
  }

 /** 
  * @description 模块加载函数，传入id，运行模块工厂函数，返回模块exports对象
  * @param {string} id module id
  * @return {object} module.exports module对象的导出对象
  */
  require = function(id) {
    ...
    return build(modules[id]);
  }
 /** 
  * @description 定义一个module，在modules对象中添加一个id和相应的module对象
  * @param {string} id module id
  * @param {function} factory module factory，该函数用来执行module的初始化，在该函数内完成对module.export的设置
  */
  define = function(id, factory) {
    ...
    // 在modules对象中添加一个module对象
    modules[id] = {
      id: id,
      factory: factory
    };
  }

  // 在define函数上添加modules对象的引用
  define.moduleMap = modules;
})();
```
modules对象的结构和factory函数的注释如下：
```javascript
// modules 对象的结构如下：
modules = {
  [id]: { // 通过module id引用module对象，一个module对象包含id、factory、exports三个属性
    id: id, // module id
    factory: factory, // 用来初始化module的factory函数
    exports: {} // module导出的对象
  }
}

// factory函数的接口定义如下
 /** 
  * @description 执行module初始化逻辑，完成对module.exports的初始化
  * @param {function} localRequire 模块内使用的require函数，返回module.exports对象
  * @param {object} module.exports module的导出对象
  * @param {object} module module对象
  */
function factory(localRequire, module.exports, module) {
  // 在函数内执行逻辑并且完成对
}
```

### 阶段二：通过define函数定义一系列的cordova module
在这个阶段定义的cordova module及其主要作用分别如下，其中加粗的module是重要module：
* **cordova**: 初始化cordova对象，对document和window对象上的addEventListener和removeEventListener方法进行替换
* **cordova/android/nativeapiprovider**：导出用来执行Native方法的对象
* **cordova/android/promptbasednativeapi**：导出一个对象，提供基于window.prompt来调用Native方法的功能
* cordova/argscheck：导出一个对象 提供参数检查功能（该模块似乎没有被任何一个module引用）
* cordova/base64：导出一个对象，将数组转换为base64编码，以及将base64编码转换为数组，在cordova/exec 模块中被引用
* cordova/builder：被cordova/modulemapper模块使用，功能未知
* **cordova/channel**：导出了一个channel对象，这个对象实现了发布订阅者模型，提供了自定义事件的注册和触发api
* **cordova/exec**：提供了androidExec方法，调用cordova.exec时，就是调用了androidExec方法
* cordova/exec/proxy：未知，没有被任何模块引用
* **cordova/init**：执行一些初始化框架的逻辑
* cordova/modulemapper：将指定的module.exports对象设置到指定的命名空间
* cordova/platform：导出一个对象，其中包含的**bootstrap**方法会执行一系列初始化行为
* cordova/plugin/android/app：该模块封装了对内置插件CoreAndroid的调用，监听返回键，音量键等等功能都在这个模块内完成
* **cordova/pluginloader**：提供动态加载cordova_plugins.js和plugins文件夹内js文件的能力
* cordova/urlutil：提供辅助函数的模块
* cordova/utils：提供辅助函数的模块

### 阶段三：加载'cordova' module
cordova module内主要做了三件事情
1. require 'cordova/channel' module，在这个module内注册了 'ondeviceready' 等重要事件
2. 替换了window和document对象上面的addEventListener和removeEventListener方法，这是后续能够使用document.addEventListener('deviceReady')的原因;
3. 初始化了cordova对象，该对象被添加到window.cordova上
```javascript
define("cordova", function (require, exports, module) {
  // require 'cordova/channel' 的时候注册好了cordova的各种自定义事件，等待触发
  var channel = require('cordova/channel');
  // cordova/platform module内部只是单纯export出了一个对象，没有多余操作，在这里不重要
  var platform = require('cordova/platform');

  // 保存document.addEventListener;和document.removeEventListener;
  var m_document_addEventListener = document.addEventListener;
  var m_document_removeEventListener = document.removeEventListener;
  // documentEventHandlers用来注册的自定义channel和对应的channel实例，Channel类定义在'cordova/channel' module中
  // documentEventHandlers = { event: channel实例 } 
  var documentEventHandlers = {};
  // 替换document.addEventListener
  document.addEventListener = function (evt, handler, capture) {
    var e = evt.toLowerCase();
    if (typeof documentEventHandlers[e] !== 'undefined') {
      // 如果是自定义事件则注册到自定义事件进行处理
      documentEventHandlers[e].subscribe(handler);
    } else {
      // 否则调用原生api进行注册
      m_document_addEventListener.call(document, evt, handler, capture);
    }
  };
  // 替换document.removeEventListener
  document.removeEventListener = function (evt, handler, capture) {
    var e = evt.toLowerCase();
    // If unsubscribing from an event that is handled by a plugin
    if (typeof documentEventHandlers[e] !== 'undefined') {
      documentEventHandlers[e].unsubscribe(handler);
    } else {
      m_document_removeEventListener.call(document, evt, handler, capture);
    }
  };

  // window对象中addEventListener和removeEventListener 也是如此处理，这里没有列出相应代码
  ...

  // 定义cordova对象，后续会被添加到window.cordova上
  var cordova = {
    // define和require就是上一步定义的define函数和require函数
    define: define,
    require: require,
    version: PLATFORM_VERSION_BUILD_LABEL,
    platformVersion: PLATFORM_VERSION_BUILD_LABEL,
    platformId: platform.id,

    /* eslint-enable no-undef */

    /**
      * Methods to add/remove your own addEventListener hijacking on document + window.
      * 在windowEventHandlers对象添加一个自定义事件
      */
    addWindowEventHandler: function (event) {
      return (windowEventHandlers[event] = channel.create(event));
    },
    // 在documentEventHandlers对象添加粘性的自定义事件
    addStickyDocumentEventHandler: function (event) {
      return (documentEventHandlers[event] = channel.createSticky(event));
    },
    // 在documentEventHandlers添加自定义事件
    addDocumentEventHandler: function (event) {
      return (documentEventHandlers[event] = channel.create(event));
    },
    removeWindowEventHandler: function (event) {
      delete windowEventHandlers[event];
    },
    removeDocumentEventHandler: function (event) {
      delete documentEventHandlers[event];
    },
    /**
      * Retrieve original event handlers that were replaced by Cordova
      * 获取原生的window和document事件处理函数
      * @return object
      */
    getOriginalHandlers: function () {
      return {
        'document': { 'addEventListener': m_document_addEventListener, 'removeEventListener': m_document_removeEventListener },
        'window': { 'addEventListener': m_window_addEventListener, 'removeEventListener': m_window_removeEventListener }
      };
    },
    /**
      * 原生代码用来触发事件的方法。
      * 会在'cordova/platform' module内的onMessageFromNative方法内被调用
      * Method to fire event from native code
      * bNoDetach is required for events which cause an exception which needs to be caught in native code
      * @param {string} type 需要触发的自定义channel
      * @param {object} data 触发channel需要携带的数据
      * @param {} bNoDetach ？？ 
      */
    fireDocumentEvent: function (type, data, bNoDetach) {
      var evt = createEvent(type, data);
      if (typeof documentEventHandlers[type] !== 'undefined') {
        if (bNoDetach) {
          // 不知道为何一个同步触发，一个异步触发
          documentEventHandlers[type].fire(evt);
        } else {
          setTimeout(function () {
            // Fire deviceready on listeners that were registered before cordova.js was loaded.
            if (type === 'deviceready') {
              // 有一些deviceready的监听可能在cordova框架没初始化好(document.addEventListener还没被替换)之前，就设置了
              // 需要触发这些监听
              document.dispatchEvent(evt);
            }
            // 触发相应的channel事件
            documentEventHandlers[type].fire(evt);
          }, 0);
        }
      } else {
        // 如果不是自定义channel，则需要触发document上相应事件
        document.dispatchEvent(evt);
      }
    },
    // 与fireDocumentEvent同理
    fireWindowEvent: function (type, data) {
      var evt = createEvent(type, data);
      if (typeof windowEventHandlers[type] !== 'undefined') {
        setTimeout(function () {
          windowEventHandlers[type].fire(evt);
        }, 0);
      } else {
        window.dispatchEvent(evt);
      }
    },

    /**
      * Plugin callback mechanism.
      */
    // Randomize the starting callbackId to avoid collisions after refreshing or navigating.
    // This way, it's very unlikely that any new callback would get the same callbackId as an old callback.
    // js调用了原生代码后，回调函数对应的回调id
    callbackId: Math.floor(Math.random() * 2000000000),
    // js调用原生代码后，回调函数会被添加到callbacks对象，对象结构如下:
    /**
     * callbacks = {
          [callbackId]: {
            success: successCallBack, // 插件调用注册的成功回调函数
            fail: failCallBack, // 插件调用注册的失败回调函数
          }
       }
     */
    callbacks: {},
    // 回调状态定义，对应Native代码中PluginResult类内部的Status枚举类
    callbackStatus: {
      NO_RESULT: 0,
      OK: 1,
      CLASS_NOT_FOUND_EXCEPTION: 2,
      ILLEGAL_ACCESS_EXCEPTION: 3,
      INSTANTIATION_EXCEPTION: 4,
      MALFORMED_URL_EXCEPTION: 5,
      IO_EXCEPTION: 6,
      INVALID_ACTION: 7,
      JSON_EXCEPTION: 8,
      ERROR: 9
    },

    /**
      * Called by native code when returning successful result from an action.
      * 由原生回调该方法，
      */
    callbackSuccess: function (callbackId, args) {
      cordova.callbackFromNative(callbackId, true, args.status, [args.message], args.keepCallback);
    },

    /**
      * Called by native code when returning error result from an action.
      * 由原生回调该方法，
      */
    callbackError: function (callbackId, args) {
      // TODO: Deprecate callbackSuccess and callbackError in favour of callbackFromNative.
      // Derive success from status.
      cordova.callbackFromNative(callbackId, false, args.status, [args.message], args.keepCallback);
    },

    /**
      * Called by native code when returning the result from an action.
      * 由原生回调该方法，调用之前插件调用注册callback函数
      */
    callbackFromNative: function (callbackId, isSuccess, status, args, keepCallback) {
      try {
        var callback = cordova.callbacks[callbackId];
        if (callback) {
          // 如果是成功并且状态是OK，则回调函数
          if (isSuccess && status === cordova.callbackStatus.OK) {
            callback.success && callback.success.apply(null, args);
          } else if (!isSuccess) {
            callback.fail && callback.fail.apply(null, args);
          }
          /*
          else
              Note, this case is intentionally not caught.
              this can happen if isSuccess is true, but callbackStatus is NO_RESULT
              which is used to remove a callback from the list without calling the callbacks
              typically keepCallback is false in this case
          */
          // Clear callback if not expecting any more results
          if (!keepCallback) {
            // 如果不需要保存回调函数，则回调完后删除该函数
            delete cordova.callbacks[callbackId];
          }
        }
      } catch (err) {
        // 如果处理回调时发生了错误，则触发一次cordovacallbackerror事件
        var msg = 'Error in ' + (isSuccess ? 'Success' : 'Error') + ' callbackId: ' + callbackId + ' : ' + err;
        console && console.log && console.log(msg);
        console && console.log && err.stack && console.log(err.stack);
        cordova.fireWindowEvent('cordovacallbackerror', { 'message': msg });
        throw err;
      }
    },
    // 在'cordovaReady' channel 上注册一个回调
    addConstructor: function (func) {
      channel.onCordovaReady.subscribe(function () {
        try {
          func();
        } catch (e) {
          console.log('Failed to run constructor: ' + e);
        }
      });
    }
  };

  module.exports = cordova;
});
```
上面的代码中，已经很清楚的说明了如何替换掉document上addEventListener和removeEventListener方法以及如何定义cordova对象。现在需要深入到 'cordova/channel' 模块中，这个模块非常重要，cordova中的自定义事件（自定义事件在cordova中用自定义channel代称，之后用自定义channel指代自定义事件）主要由这个模块实现。

cordova/channel做了两件非常重要的事情
1. 内部定义Channel类，类的每一个实例都代表一个自定义channel和提供subscribe、unsubscribe、fire等方法；对外暴露一个channel对象（该对象封装了Channel类实例，但本身不是Channel类实例，只是命名有些误导），依靠这个对象对外提供注册自定义channel的能力
2. 注册了'onDeviceReady'、'onCordovaReady'、'onPluginsReady'、'onNativeReady'等自定义channel。这些事件在后续cordova初始化的过程中会一一触发

```javascript
define("cordova/channel", function (require, exports, module) {
/**
  * Channel类，一个channel实例代表一个自定义事件(channel)
  * 其内部维护着自定义channel以及注册的handlers。并且提供subscribe、unsubscribe、fire等操作接口
  * @constructor
  * @param type  String the channel name
  * @sticky NOTE: 是否粘性的（粘性的channel在触发后会进入“已触发”这个状态，此时如果添加一个回调函数，则会立即调用。例如deviceready，如果已经触发过了，则会立即回调监听事件。非粘性的channel则每次注册回调函数后，都需要下一次fire时，才会触发回调函数）
  */
  var Channel = function (type, sticky) {
    this.type = type;
    // Map of guid -> function.
    this.handlers = {};
    // 0 = Non-sticky, 1 = Sticky non-fired, 2 = Sticky fired.
    this.state = sticky ? 1 : 0;
    // Used in sticky mode to remember args passed to fire().
    this.fireArgs = null;
    // Used by onHasSubscribersChange to know if there are any listeners.
    this.numHandlers = 0;
    // Function that is called when the first listener is subscribed, or when
    // the last listener is unsubscribed.
    // 当第一个函数注册时，或者最后一个函数被注销时，回调该属性引用的函数
    this.onHasSubscribersChange = null;
  };

  // 向指定channel注册一个处理函数
  Channel.prototype.subscribe = function (eventListenerOrFunction, eventListener) {
    ...
    if (this.state === 2) {
      // 当前channel的state为2，代表当前channel是Sticky的，并且已经fired，需要立即回调订阅函数
      handleEvent.apply(eventListener || this, this.fireArgs);
      return;
    }

    ...
    // 对注册函数添加一个observer_guid属性，用来标志该函数是注册在channel的监听函数，在unsubscribe的时候用得上
    handleEvent.observer_guid = guid;
    eventListenerOrFunction.observer_guid = guid;

    if (!this.handlers[guid]) {
      // 将回调函数注册到handlers，等待回调
      this.handlers[guid] = handleEvent;
      this.numHandlers++;
      if (this.numHandlers === 1) {
        // 当channel中注册了第一个回调函数时，尝试调用下面的钩子，进行逻辑处理
        this.onHasSubscribersChange && this.onHasSubscribersChange();
      }
    }
  }

  // 在指定channel上注销一个处理函数
  Channel.prototype.unsubscribe = function (eventListenerOrFunction) {
    ...
    // 获取注册函数内的guid，以此为key，在handlers中获取对应函数，然后删除
    guid = handleEvent.observer_guid;
    handler = this.handlers[guid];
    if (handler) {
      delete this.handlers[guid];
      this.numHandlers--;
      if (this.numHandlers === 0) {
        this.onHasSubscribersChange && this.onHasSubscribersChange();
      }
    }
  }

  // 触发指定channel上的所有处理函数 
  Channel.prototype.fire = function (e) {
    var fail = false; // eslint-disable-line no-unused-vars
    var fireArgs = Array.prototype.slice.call(arguments);
    // Apply stickiness.
    if (this.state === 1) {
      // 如果当前channel是stick的，那么在fire内将状态设置为2，代表已经fire过，并且保存fire的参数
      this.state = 2;
      this.fireArgs = fireArgs;
    }
    if (this.numHandlers) {
      // Copy the values first so that it is safe to modify it from within
      // callbacks.
      var toCall = [];
      for (var item in this.handlers) {
        toCall.push(this.handlers[item]);
      }
      // 逐一调用注册的函数
      for (var i = 0; i < toCall.length; ++i) {
        toCall[i].apply(this, fireArgs);
      }
      // 清空触发后的处理函数
      if (this.state === 2 && this.numHandlers) {
        this.numHandlers = 0;
        this.handlers = {};
        // 在清空函数后，回调一次对应钩子
        this.onHasSubscribersChange && this.onHasSubscribersChange();
      }
    }
  }

   // 导出的module.exports对象，该对象封装了Channel类，但是本身不是Channel类实例
   // 对象提供create等接口来创建channel
  var channel = {
    /**
      * Calls the provided function only after all of the channels specified
      * have been fired. All channels must be sticky channels.
      * 在指定的Channel（由参数二指定）都fire之后，会立即回调一次h 函数
      * @param {function} h 需要注册的回调函数
      * @param {Channel[]} c Channel类实例组成的数组，所有的Channel类实例必须都是sticky channels
      */
    join: function (h, c) {
      var len = c.length;
      var i = len;
      var f = function () {
        if (!(--i)) h();
      };
      for (var j = 0; j < len; j++) {
        if (c[j].state === 0) {
          throw Error('Can only use join with sticky channels.');
        }
        c[j].subscribe(f);
      }
      if (!len) h();
    },
    /* eslint-disable no-return-assign */
    // 创建一个channel，返回新建的channel对象
    create: function (type) {
      return channel[type] = new Channel(type, false);
    },
    // 创建一个sticky channel，返回新建的channel对象
    createSticky: function (type) {
      return channel[type] = new Channel(type, true);
    },
    /* eslint-enable no-return-assign */
    /**
      * cordova Channels that must fire before "deviceready" is fired.
      * 这两个属性用来记录 deviceready 触发之前，必须被触发的channels
      */
    deviceReadyChannelsArray: [],
    deviceReadyChannelsMap: {},

  /**
    * Indicate that a feature needs to be initialized before it is ready to be used.
    * This holds up Cordova's "deviceready" event until the feature has been initialized
    * and Cordova.initComplete(feature) is called.
    * @description 创建一个自定义channel放入到 deviceReadyChannelsArray 和 deviceReadyChannelsMap
    * deviceReadyChannelsArray数组内的channel都触发后，就会触发 deviceready channel，这段代码在 'cordova/init' module内
    * @param feature {String}     The unique feature name
    */
    waitForInitialization: function (feature) {
      if (feature) {
        var c = channel[feature] || this.createSticky(feature);
        this.deviceReadyChannelsMap[feature] = c;
        this.deviceReadyChannelsArray.push(c);
      }
    },

    /**
      * Indicate that initialization code has completed and the feature is ready to be used.
      * @description 触发deviceReadyChannelsMap中的指定channel
      * @param feature {String}     The unique feature name
      */
    initializationComplete: function (feature) {
      var c = this.deviceReadyChannelsMap[feature];
      if (c) {
        c.fire();
      }
    }
  };

  // NOTE:  以下注册一系列的重要channel
  channel.createSticky('onDOMContentLoaded');
  // Event to indicate the Cordova native side is ready.
  channel.createSticky('onNativeReady');
  // Event to indicate that all Cordova JavaScript objects have been created
  // and it's time to run plugin constructors.
  channel.createSticky('onCordovaReady');
  // Event to indicate that all automatically loaded JS plugins are loaded and ready.
  // FIXME remove this
  channel.createSticky('onPluginsReady');
  // Event to indicate that Cordova is ready
  channel.createSticky('onDeviceReady');
  // Event to indicate a resume lifecycle event
  channel.create('onResume');
  // Event to indicate a pause lifecycle event
  channel.create('onPause');
  // Channels that must fire before "deviceready" is fired.
  // 让deviceready在onCordovaReady和onDOMContentLoaded都触发后立即触发
  channel.waitForInitialization('onCordovaReady');
  channel.waitForInitialization('onDOMContentLoaded');

  module.exports = channel;
})
```

### 阶段四：运行'cordova/init' module
'cordova/init' module内部主要做了四件比较重要的事情：
1. 使用自定义的对象替换掉了原生的window.navigator
2. 在document上注册pause、resume、activated、deviceready等自定义事件，使得后续可以使用document.addEventListener来监听这些事件
3. 运行 'cordova/platform' module导出的bootstrap方法，该方法在document上注册了 backbutton、menubutton、searchbutton、volumeup、volumedown等自定义channel，并且调用原生方法，将NativeToJs的mode设置为EvalBridgeMode
4. 设置好一系列自定义channel的触发时机和回调函数，其中最后触发的是deviceready。

```javascript
  define("cordova/init", function (require, exports, module) {
    var channel = require('cordova/channel');
    var cordova = require('cordova');
    var modulemapper = require('cordova/modulemapper');
    var platform = require('cordova/platform');
    var pluginloader = require('cordova/pluginloader');
    var utils = require('cordova/utils');

    // 后续会用channel.join方法在这两个事件触发后添加回调方法，进行一些初始化
    var platformInitChannelsArray = [channel.onNativeReady, channel.onPluginsReady];

    function logUnfiredChannels(arr) {
      for (var i = 0; i < arr.length; ++i) {
        if (arr[i].state !== 2) {
          console.log('Channel not fired: ' + arr[i].type);
        }
      }
    }

    // 5s后如果onDeviceReady还没有触发，则打印相关提示
    window.setTimeout(function () {
      if (channel.onDeviceReady.state !== 2) {
        console.log('deviceready has not fired after 5 seconds.');
        logUnfiredChannels(platformInitChannelsArray);
        logUnfiredChannels(channel.deviceReadyChannelsArray);
      }
    }, 5000);

    // Replace navigator before any modules are required(), to ensure it happens as soon as possible.
    // We replace it so that properties that can't be clobbered can instead be overridden.
    // 不清楚替换原生的navigator有什么好处
    function replaceNavigator(origNavigator) {
      var CordovaNavigator = function () { };
      CordovaNavigator.prototype = origNavigator;
      var newNavigator = new CordovaNavigator();
      // This work-around really only applies to new APIs that are newer than Function.bind.
      // Without it, APIs such as getGamepads() break.
      if (CordovaNavigator.bind) {
        for (var key in origNavigator) {
          if (typeof origNavigator[key] === 'function') {
            newNavigator[key] = origNavigator[key].bind(origNavigator);
          } else {
            (function (k) {
              utils.defineGetterSetter(newNavigator, key, function () {
                return origNavigator[k];
              });
            })(key);
          }
        }
      }
      return newNavigator;
    }

    // NOTE: 步骤1：替换掉原生的navigator对象
    if (window.navigator) {
      window.navigator = replaceNavigator(window.navigator);
    }

    if (!window.console) {
      window.console = {
        log: function () { }
      };
    }
    if (!window.console.warn) {
      window.console.warn = function (msg) {
        this.log('warn: ' + msg);
      };
    }

    // NOTE: 步骤二：在document上注册pause、resume、activated、deviceready等自定义事件
    // Register pause, resume and deviceready channels as events on document.
    // 在document上注册pause、resume、activated、deviceready事件，
    // pause、resume、activated事件后续在Activity进入相应状态时，会通过CoreAndroid内置插件，由原生代码触发
    // 这些事件到时候由native触发
    channel.onPause = cordova.addDocumentEventHandler('pause');
    channel.onResume = cordova.addDocumentEventHandler('resume');
    channel.onActivated = cordova.addDocumentEventHandler('activated');
    // 这里重新注册了一个deviceready channel替换掉了之前注册的onDeviceReady（不知道之前的注册有什么意义），之后可以使用document.addEventListener('deviceready', handler);
    channel.onDeviceReady = cordova.addStickyDocumentEventHandler('deviceready');

    // Listen for DOMContentLoaded and notify our channel subscribers.
    // 在DOMContentLoaded的时候触发'DOMContentLoaded' channel
    if (document.readyState === 'complete' || document.readyState === 'interactive') {
      channel.onDOMContentLoaded.fire();
    } else {
      document.addEventListener('DOMContentLoaded', function () {
        channel.onDOMContentLoaded.fire();
      }, false);
    }

    // _nativeReady is global variable that the native side can set
    // to signify that the native code is ready. It is a global since
    // it may be called before any cordova JS is ready.
    // cordova android中并没有设置window._nativeReady
    if (window._nativeReady) {
      channel.onNativeReady.fire();
    }

    // 将cordova模块导出对象映射到cordova对象上
    modulemapper.clobbers('cordova', 'cordova');
    // 将'cordova/exec' module的exports对象 映射到cordova.exec属性上
    modulemapper.clobbers('cordova/exec', 'cordova.exec');
    modulemapper.clobbers('cordova/exec', 'Cordova.exec');

    // Call the platform-specific initialization.
    // NOTE: 步骤三：运行相应平台的初始化代码，TODO: 其内部完成了什么任务
    platform.bootstrap && platform.bootstrap();

    // NOTE: 步骤四：设置好一系列自定义channel的触发时机和回调函数
    // Wrap in a setTimeout to support the use-case of having plugin JS appended to cordova.js.
    // The delay allows the attached modules to be defined before the plugin loader looks for them.
    setTimeout(function () {
      // 在下一个event loop动态加载cordova_plugins.js文件和plugins文件夹下的js插件文件
      // 加载完后触发一个onPluginsReady 事件
      pluginloader.load(function () {
        console.log('all plugins loaded');
        channel.onPluginsReady.fire();
      });
    }, 0);

    /**
     * Create all cordova objects once native side is ready.
     * 在onNativeReady, onPluginsReady触发之后，回调一个函数
     */
    channel.join(function () {
      // 运行模块映射代码
      modulemapper.mapModules(window);

      // 并没有这个方法
      platform.initialize && platform.initialize();

      // Fire event to notify that all objects are created
      // 触发CordovaReady事件
      channel.onCordovaReady.fire();

      // Fire onDeviceReady event once page has fully loaded, all
      // constructors have run and cordova info has been received from native
      // side.
      // 在deviceReadyChannelsArray数组内的channel都触发了之后，触发deviceready 事件
      // 此时deviceReadyChannelsArray内包含 onCordovaReady、onDOMContentLoaded 事件
      channel.join(function () {
        console.log('deviceready is going to fire');
        // 触发deviceready事件
        require('cordova').fireDocumentEvent('deviceready');
      }, channel.deviceReadyChannelsArray);

    }, platformInitChannelsArray);

    console.log('init cordova/init end');

  });
```
platform.bootstrap方法是由'cordova/platform' module导出的方法。bootstrap主要实现了四个比较重要的步骤：
1. 调用androidExec.init方法。在方法内部使用prompt方式调用原生代码，设置NativeToJs mode为EvalBridgeMode，并且获取bridgeSecret的初始值，然后触发onNativeReady事件
2. 将 'cordova/plugin/android/app' module内export出来的对象添加到 window.navigator.app 上。'cordova/plugin/android/app' module是对CorAndroid内置插件的封装，其导出的对象包含了一系列调用该插件的接口方法
3. 在document上注册backbutton、menubutton、searchbutton、volumeup、volumedown等自定义channel，这些事件会在相应动作发生时，通过CoreAndroid插件来通知js，进而触发这些事件
4. 调用CoreAndroid插件的messageChannel action，注册onMessageFromNative方法，这个方法会被Native端keep住作为Native向Js发送消息的通道。
```javascript
define("cordova/platform", function (require, exports, module) {

    console.log('init cordova/platform start');

    // The last resume event that was received that had the result of a plugin call.
    var lastResumeEvent = null;

    module.exports = {
      id: 'android',
      bootstrap: function () {
        var channel = require('cordova/channel'),
          cordova = require('cordova'),
          exec = require('cordova/exec'),
          modulemapper = require('cordova/modulemapper');

        // Get the shared secret needed to use the bridge.
        // NOTE:  步骤1：调用androidExec.init方法，使用prompt方式调用原生代码，设置NativeToJs mode为EvalBridgeMode
        // 并且获取bridgeSecret的初始值，然后触发onNativeReady事件
        exec.init();

        // Extract this as a proper plugin.
        // NOTE: 步骤2：将 'cordova/plugin/android/app' module内export出来的对象添加到 window.navigator.app 上
        // 'cordova/plugin/android/app' module是对CorAndroid内置插件的封装，其导出的对象包含了一系列调用该插件的接口方法
        modulemapper.clobbers('cordova/plugin/android/app', 'navigator.app');

        var APP_PLUGIN_NAME = Number(cordova.platformVersion.split('.')[0]) >= 4 ? 'CoreAndroid' : 'App';

        // Inject a listener for the backbutton on the document.
        // NOTE: 步骤3：在document上注册backbutton、menubutton、searchbutton、volumeup、volumedown等自定义channel，这些事件会在相应动作发生时，通过CoreAndroid
        // 插件来通知js，进而触发这些事件
        // 注册backbutton事件，使得后续可以使用document.addEventListener('backbutton', ...)
        var backButtonChannel = cordova.addDocumentEventHandler('backbutton');
        backButtonChannel.onHasSubscribersChange = function () {
          // If we just attached the first handler or detached the last handler,
          // let native know we need to override the back button.
          // 当注册第一个backbutton监听函数时，调用插件的overrideBackbutton，传入true，表示native端遇到用户点击backbutton时，
          // 阻止webview的默认行为（回退浏览器历史），当注销掉所有的backbutton监听时，传入 false，表示native恢复backbutton的默认行为
          // 执行内置插件 CoreAndroid 的overrideBackbutton action动作
          exec(null, null, APP_PLUGIN_NAME, "overrideBackbutton", [this.numHandlers == 1]);
        };

        // Add hardware MENU and SEARCH button handlers
        // NOTE: 在document上添加menubutton和searchbutton的自定义channel
        cordova.addDocumentEventHandler('menubutton');
        cordova.addDocumentEventHandler('searchbutton');

        // NOTE: 与backbutton的逻辑一样，提供volumeupbutton和volumedownbutton两种channel
        // 当js这边注册了这两种channel监听时，则会阻止原生的默认行为
        // js这边注销了所有channel监听后，则恢复原生的默认行为
        function bindButtonChannel(buttonName) {
          // generic button bind used for volumeup/volumedown buttons
          var volumeButtonChannel = cordova.addDocumentEventHandler(buttonName + 'button');
          volumeButtonChannel.onHasSubscribersChange = function () {
            exec(null, null, APP_PLUGIN_NAME, "overrideButton", [buttonName, this.numHandlers == 1]);
          };
        }
        // Inject a listener for the volume buttons on the document.
        bindButtonChannel('volumeup');
        bindButtonChannel('volumedown');

        // The resume event is not "sticky", but it is possible that the event
        // will contain the result of a plugin call. We need to ensure that the
        // plugin result is delivered even after the event is fired (CB-10498)
        // 再一次替换document.addEventListener，之前已经替换过一次
        var cordovaAddEventListener = document.addEventListener;

        document.addEventListener = function (evt, handler, capture) {
          cordovaAddEventListener(evt, handler, capture);

          // 如果监听的事件是resume，并且有lastResumeEvent则立即运行，意义何在？？
          if (evt === 'resume' && lastResumeEvent) {
            handler(lastResumeEvent);
          }
        };

        // Let native code know we are all done on the JS side.
        // Native code will then un-hide the WebView.
        channel.onCordovaReady.subscribe(function () {
          // NOTE: 步骤四 调用CoreAndroid插件的messageChannel action，注册onMessageFromNative回调函数
          // native端会keep住这个回调函数，作为native到js的一条发布消息的通道。
          exec(onMessageFromNative, null, APP_PLUGIN_NAME, 'messageChannel', []);
          exec(null, null, APP_PLUGIN_NAME, "show", []);
        });
      }
    };

    // 该函数是Native到js的发布消息的通道，由CoreAndroid插件调用
    function onMessageFromNative(msg) {
      var cordova = require('cordova');
      var action = msg.action;

      switch (action) {
        // Button events
        case 'backbutton':
        case 'menubutton':
        case 'searchbutton':
        // App life cycle events
        case 'pause':
        // Volume events
        case 'volumedownbutton':
        case 'volumeupbutton':
          cordova.fireDocumentEvent(action);
          break;
        case 'resume':
          if (arguments.length > 1 && msg.pendingResult) {
            if (arguments.length === 2) {
              msg.pendingResult.result = arguments[1];
            } else {
              // The plugin returned a multipart message
              var res = [];
              for (var i = 1; i < arguments.length; i++) {
                res.push(arguments[i]);
              }
              msg.pendingResult.result = res;
            }

            // Save the plugin result so that it can be delivered to the js
            // even if they miss the initial firing of the event
            lastResumeEvent = msg;
          }
          cordova.fireDocumentEvent(action, msg);
          break;
        default:
          throw new Error('Unknown event action ' + action);
      }
    }

    console.log('init cordova/platform end');
  });
```

至此Cordova.js的初始化工作就完毕了，总结一下Cordova.js的初始化工作主要做了以下几件最重要的事情：
1. 添加了window.cordova对象，该对象作为cordova对外提供接口方法的媒介，特别是cordova.exec方法，成为调用插件的统一入口
2. 修改了docuemnt和window的addEventListener和removeEventListener方法，进而实现了'deviceReady'等自定义事件
3. 动态加载了cordova_plugin.js文件，以及根据文件内的记录动态加载安装的cordova插件的js文件，并且在做好所有初始化工作后，触发'deviceReady'事件

## Cordova.js中自定义channel的注册表和触发表
| channel注册表（channel在表格内的顺序代表了注册顺序） | | | | | |
| ------ | ------ | ------ | ------ | ------ | ------ |
| channel名称 | 是否sticky | 注册位置 | 注册时函数调用栈 | 是否注册在document上 | 备注 |
| onDOMContentLoaded | true | 'cordova/channel' module内 | cordova立即执行函数 -> require('cordova') -> require('cordova/channel') | false | 当document的DOMContentLoaded事件触发时，会触发'onDOMContentLoaded' |
| onNativeReady | true  | 'cordova/channel' module内 | cordova立即执行函数 -> require('cordova') -> require('cordova/channel') | false | 在androidExec.init方法内触发，触发前设置了NativeToJs的mode |
| onCordovaReady | true | 'cordova/channel' module内 | cordova立即执行函数 -> require('cordova') -> require('cordova/channel') | false | 在'onNativeReady'和'onPluginsReady'触发之后立即触发 |
| onPluginsReady | true | 'cordova/channel' module内 | cordova立即执行函数 -> require('cordova') -> require('cordova/channel') | false | 在动态加载cordova_plugin.js和plugins文件夹下面的js文件都动态加载完后触发 |
| onDeviceReady | true | 'cordova/channel' module内 | cordova立即执行函数 -> require('cordova') -> require('cordova/channel') | false | 后续会在'cordova/init' module内用 cordova.addDocumentEventHandler重新生成channel类实例覆盖，channel.onPause与channel.pause指向同一个Channel类实例；onDeviceReady会在onCordovaReady和onDOMContentLoaded都触发后，立即触发 |
| onResume | false | 'cordova/channel' module内 | cordova立即执行函数 -> require('cordova') -> require('cordova/channel') | false | 后续会在'cordova/init' module内用 cordova.addDocumentEventHandler重新生成channel类实例覆盖，channel.onPause与channel.pause指向同一个Channel类实例 |
| onPause | false | 'cordova/channel' module内 | cordova立即执行函数 -> require('cordova') -> require('cordova/channel') | false | 后续会在'cordova/init' module内用 cordova.addDocumentEventHandler重新生成channel类实例覆盖，channel.onPause与channel.pause指向同一个Channel类实例 |
| pause | false | 'cordova/init' module内 | cordova立即执行函数 -> require('cordova/init') | true | 当Activity进入后台时，由CoreAndroid在js中触发该事件 |
| resume | false | 'cordova/init' module内 | cordova立即执行函数 -> require('cordova/init') | true | 当Activity从后台返回到前台的时候，由CoreAndroid在js中触发该事件 |
| activated | false | 'cordova/init' module内 | cordova立即执行函数 -> require('cordova/init') | true | |
| deviceready | true | 'cordova/init' module内 | cordova立即执行函数 -> require('cordova/init') | true | 当'onCordovaReady' 和 'onDOMContentLoaded' 触发后，'deviceready'会立即触发 |
|  |

channel触发过程
```
调用androidExec.init方法设置NativeToJs的mode -> 触发'onNativeReady' -> 步骤1
动态加载cordova_plugins.js和plugins文件夹下文件完成 -> 触发'onPluginsReady' -> 步骤2
等待原生DOMContentLoaded事件触发 -> 触发'onDOMContentLoaded' -> 步骤3

步骤1 -> 
         -> 触发onCordovaReady -> 步骤4
步骤2 -> 

步骤4 ->
          -> 触发'deviceready'
步骤3 -> 
```
    