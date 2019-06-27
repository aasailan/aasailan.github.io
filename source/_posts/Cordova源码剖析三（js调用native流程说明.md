---
title: Cordova源码剖析三（js调用native流程说明)
date: 2019-06-27 17:20:37
categories:
  - cordova系列
tags: 
  - 前端
  - cordova
  - 源码剖析
comments: false
---
# cordova中js调用android原生代码的过程剖析

这篇文章以toast插件为例，剖析cordova中js调用android原生代码以及native对js进行回调的整个过程，中间涉及到三个比较重要的点：
1. js插件如何通过cordova.exec调用到native方法
2. 进入native方法后如何调用到对应的插件
3. 插件执行后如何回调js

## 从Toast插件接口到调用Native方法
在js一侧调用toast的js插件暴露出来的方法
```javascript
window.plugins.toast.show('ps^^, you are in my code', 'short', 'bottom', function(res) {
  console.log(res);
}, function(error) {
  console.log(error);
});
```

window.plugins.toast对象是Toast插件的js部分代码添加的，深入plugins/cordova-plugin-x-toast/www/Toast.js文件中可以看到该对象的封装过程。该文件会在cordova.js运行启动框架的过程中被动态加载到html。（具体描述可以参考上一篇文章）
```javascript
cordova.define("cordova-plugin-x-toast.Toast", function (require, exports, module) {
  // 定义一个Toast类构造函数
  function Toast() {
  }

  ....

  Toast.prototype.showWithOptions = function (options, successCallback, errorCallback) {
    options.duration = (options.duration === undefined ? 'long' : options.duration.toString());
    options.message = options.message.toString();
    // 注意cordova.exec的传参方法 https://cordova.apache.org/docs/en/latest/guide/hybrid/plugins/index.html#the-javascript-interface
    // 调用利用cordova.exec方法，cordova.exec方法是js调用插件的统一入口
    cordova.exec(successCallback, errorCallback, "Toast", "show", [options]);
  };

  Toast.prototype.show = function (message, duration, position, successCallback, errorCallback) {
    this.showWithOptions(
      this.optionsBuilder()
        .withMessage(message)
        .withDuration(duration)
        .withPosition(position) 
        .build(),
      successCallback,
      errorCallback);
  };

  ...
  // 定义install方法把Toast插件实例放置到window对象上
  Toast.install = function () {
    if (!window.plugins) {
      window.plugins = {};
    }

    window.plugins.toast = new Toast();
    return window.plugins.toast;
  };

  // 在cordovaReady事件上注册一个回调，等待触发后，插件安装
  // cordovaReady是一个sticky channel
  cordova.addConstructor(Toast.install);
}
```
上面的js代码主要做了三件事情
1. 定义了一个cordova module
2. 在module的factory函数中，使用Toast类封装了对cordova.exec方法的调用。利用Toast类实例来对外暴露Toast插件的调用接口
3. 在cordovaReady这个channel中添加回调处理，在回调中将Toast类实例添加到window对象上

以上三步也是其他插件的js部分代码几乎都会做的通用三步。

所以window.plugins.toast.show方法会调用到cordova.exec方法，而cordova.exec方法其实就是定义在cordova.js内的androidExec方法。
```javascript
/**
     * @description cordova.exec方法定义，js调用插件的统一入口
     * @param {function} success 成功回调函数 
     * @param {function} fail 错误回调函数
     * @param {string} service 插件名
     * @param {string} action 插件内部的action名
     * @param {Array<any>} args 参数数组
     */
    function androidExec(success, fail, service, action, args) {
      if (bridgeSecret < 0) {
        // If we ever catch this firing, we'll need to queue up exec()s
        // and fire them once we get a secret. For now, I don't think
        // it's possible for exec() to be called since plugins are parsed but
        // not run until until after onNativeReady.
        throw new Error('exec() called without bridgeSecret');
      }
      // Set default bridge modes if they have not already been set.
      // By default, we use the failsafe, since addJavascriptInterface breaks too often
      if (jsToNativeBridgeMode === undefined) {
        // 首次调用原生方式，尝试使用js_object方式，setJsToNativeBridgeMode做了兼容处理，如果无法使用js_object方式，则使用prompt方式
        androidExec.setJsToNativeBridgeMode(jsToNativeModes.JS_OBJECT);
      }

      // If args is not provided, default to an empty array
      // 保证原生收到的参数起码是个空数组
      args = args || [];

      // Process any ArrayBuffers in the args into a string.
      for (var i = 0; i < args.length; i++) {
        // 如果参数对象是一个数组，则需要把这个数组编码成base64
        if (utils.typeName(args[i]) == 'ArrayBuffer') {
          console.log('before base64', args[i]);
          args[i] = base64.fromArrayBuffer(args[i]);
          console.log('after base64', args[i]);
        }
      }

      var callbackId = service + cordova.callbackId++,
        argsJson = JSON.stringify(args);
      if (success || fail) {
        // 将success回调函数和fail回调函数封装成对象，动态添加到cordova.callbacks对象中
        cordova.callbacks[callbackId] = { success: success, fail: fail };
      }

      // 通过nativeApiProvider获取原生api提供对象并执行原生方法，
      // NOTE: 这个msgs是否是执行原生方法后直接返回的结果，而不是native的callback回调
      // 具体的消息结果参见CordovaBridge.jsExec方法内的jsMessageQueue.popAndEncode方法调用
      console.log('plugin exec start:', bridgeSecret, service, action, callbackId, argsJson);
      var msgs = nativeApiProvider.get().exec(bridgeSecret, service, action, callbackId, argsJson);
      console.log('plugin exec end:', callbackId, msgs);
      // If argsJson was received by Java as null, try again with the PROMPT bridge mode.
      // This happens in rare circumstances, such as when certain Unicode characters are passed over the bridge on a Galaxy S2.  See CB-2666.
      if (jsToNativeBridgeMode == jsToNativeModes.JS_OBJECT && msgs === "@Null arguments.") {
        // 如果原生方法返回@Null arguments.则使用prompt方法尝试再调用一次
        androidExec.setJsToNativeBridgeMode(jsToNativeModes.PROMPT);
        androidExec(success, fail, service, action, args);
        androidExec.setJsToNativeBridgeMode(jsToNativeModes.JS_OBJECT);
      } else if (msgs) {
        // 将原生回传的信息存入消息队列
        messagesFromNative.push(msgs);
        // Always process async to avoid exceptions messing up stack.
        // 在下一个event loop处理消息
        nextTick(processMessages);
      }
    }
```
androidExec方法内通过nativeApiProvider.get方法，来获取一个特定的对象，然后调用exec方法，这个特定对象的exec方法就会真正执行到Native端的代码。    
nativeApiProvider.get方法对外隐藏了Js调用native存在两种方式的细节，深入nativeApiProvider.get方法可以查看到JS调用native的两种方式的差别。nativeApiProvider.get方法定义在 'cordova/android/nativeapiprovider' 这个module内

```javascript
define("cordova/android/nativeapiprovider", function (require, exports, module) {
    /**
     * Exports the ExposedJsApi.java object if available, otherwise exports the PromptBasedNativeApi.
     */
    // _cordovaNative 由原生注册，如果没有则使用基于prompt的api调用
    // 在CordovaWebView初始化时，会根据Android版本的不同，初始化不同的JS调用Native的方法，
    // 当Android版本小于4.2(API 17)时，会采用prompt的方式处理JS的调用，当Android版本大于4.2时，会采用JavaScriptInterface的方式调用
    var nativeApi = this._cordovaNative || require('cordova/android/promptbasednativeapi');
    var currentApi = nativeApi;

    module.exports = {
      // NOTE: 这就是nativeApiProvider.get方法
      get: function () { return currentApi; },
      // 是否优先选择使用基于prompt的调用方式
      setPreferPrompt: function (value) {
        currentApi = value ? require('cordova/android/promptbasednativeapi') : nativeApi;
      },
      // Used only by tests.
      set: function (value) {
        currentApi = value;
      }
    };
  });
```
nativeApiProvider.get方法其实就是 'cordova/android/nativeapiprovider' module导出的get方法，get方法返回的currentApi对象有两种可能：
1. window._cordovaNative是Native端用@javascriptInterface注解在webview注册的原生对象。window._cordovaNative存在时意味Js调用native是通过@javascriptInterface注解注册原生对象的方式来进行。
2. window._cordovaNative不存在时，会require 'cordova/android/promptbasednativeapi' module，这个module内会使用基于prompt的方式与native进行通讯

'cordova/android/promptbasednativeapi' module的代码如下:
```javascript
define("cordova/android/promptbasednativeapi", function (require, exports, module) {
    /**
     * Implements the API of ExposedJsApi.java, but uses prompt() to communicate.
     * This is used pre-JellyBean, where addJavascriptInterface() is disabled.
     */

    module.exports = {
      exec: function (bridgeSecret, service, action, callbackId, argsJson) {
        // window.prompt 打开一个对话框 https://developer.mozilla.org/zh-CN/docs/Web/API/Window/prompt
        // native端的SystemWebChromeClient类实现了onJsPrompt方法来对js的弹窗进行拦截，然后可以执行native代码
        return prompt(argsJson, 'gap:' + JSON.stringify([bridgeSecret, service, action, callbackId]));
      },
      setNativeToJsBridgeMode: function (bridgeSecret, value) {
        prompt(value, 'gap_bridge_mode:' + bridgeSecret);
      },
      retrieveJsMessages: function (bridgeSecret, fromOnlineEvent) {
        return prompt(+fromOnlineEvent, 'gap_poll:' + bridgeSecret);
      }
    };
  });
```

所以到了这一步，就已经调用到了native代码。而调用native代码的方式在这里出现了两条可能的通道：
1. 通道一 window._cordovaNative对象
2. 通道二 基于prompt的方式。
通常在android4.2版本以上都会走通道一，android4.2以下版本因为不支持@javasciptInterface注解，所以只能走通道二。

我们接下来先讨论通道一的情况。并且此时js侧的函数调用顺序大概如下：
```
window.plugins.toast.show -> Toast.prototype.showWithOptions -> cordova.exec(实际上是androidExec方法) -> nativeApiProvider.get().exec()(实际上是window._cordovaNative.exec()方法)
```

## 进入native方法后如何调用到对应的插件
js一侧调用window._cordovaNative.exec方法后，就进入了native中的SystemExposedJsApi类实例的exec方法，然后立即调用了CordovaBridge类实例的jsExec方法
```java
// CordovaBridge类内部
public String jsExec(int bridgeSecret, String service, String action, String callbackId, String arguments) throws JSONException, IllegalAccessException {
    if (!verifySecret("exec()", bridgeSecret)) {
        return null;
    }
    // If the arguments weren't received, send a message back to JS.  It will switch bridge modes and try again.  See CB-2666.
    // We send a message meant specifically for this case.  It starts with "@" so no other message can be encoded into the same string.
    if (arguments == null) {
        // 如果没有传入参数则直接返回"@Null arguments." js端会再次尝试用基于prompt的交流方式
        return "@Null arguments.";
    }
    // 调用native插件期间暂停nativeToJS的消息传递
    jsMessageQueue.setPaused(true);
    try {
        // Tell the resourceApi what thread the JS is running on.
        CordovaResourceApi.jsThread = Thread.currentThread();
        String threadName = Thread.currentThread().getName();
        // 利用pluginManager获取指定插件名插件，然后执行
        Log.d(LOG_TAG, "before pluginManager exec: " + service + " " + action + " " + callbackId + " " + threadName);
        pluginManager.exec(service, action, callbackId, arguments);
        Log.d(LOG_TAG, "after pluginManager exec: " + service + " " + action + " " + callbackId + " " + threadName);
        String ret = null;
        if (!NativeToJsMessageQueue.DISABLE_EXEC_CHAINING) {
            // DISABLE_EXEC_CHAINING一般为false
            // 如果消息队列为空则ret为null，否则会得到一个字符串例如：'len S10 callbackId len S10 callbackId*'
            ret = jsMessageQueue.popAndEncode(false);
        }
        Log.d(LOG_TAG, "cordova.exec return: " + ret);
        return ret;
    } catch (Throwable e) {
        e.printStackTrace();
        return "";
    } finally {
        jsMessageQueue.setPaused(false);
    }
}
```
CordovaBridge.jsExec方法调用了pluginManager.exec方法。在这个方法中pluginManager通过插件名找到相应的插件，然后执行插件的execute方法。
```java
// pluginManager类内部
public void exec(final String service, final String action, final String callbackId, final String rawArgs) {
    // 按照插件名获取插件实例
    CordovaPlugin plugin = getPlugin(service);
    if (plugin == null) {
        LOG.d(TAG, "exec() call to unknown plugin: " + service);
        PluginResult cr = new PluginResult(PluginResult.Status.CLASS_NOT_FOUND_EXCEPTION);
        app.sendPluginResult(cr, callbackId);
        return;
    }
    // 初始化一个CallbackContext 实例
    CallbackContext callbackContext = new CallbackContext(callbackId, app);
    try {
        long pluginStartTime = System.currentTimeMillis();
        // 执行插件的execute方法，wasValidAction为false说明是一个非法action调用
        // rawArgs此时任然是一个json格式字符串，CordovaPlugin类的execute方法内对其进行了反序列化
        Log.d(TAG, "before plugin execute: " + service + " " + action);
        // 执行指定插件的execute方法，返回的boolean值意味是否是有效action
        boolean wasValidAction = plugin.execute(action, rawArgs, callbackContext);
        Log.d(TAG, "after plugin execute: " + service + " " + action);
        long duration = System.currentTimeMillis() - pluginStartTime;

        if (duration > SLOW_EXEC_WARNING_THRESHOLD) {
            LOG.w(TAG, "THREAD WARNING: exec() call to " + service + "." + action + " blocked the main thread for " + duration + "ms. Plugin should use CordovaInterface.getThreadPool().");
        }
        if (!wasValidAction) {
            // 如果是一个非法action调用，则需要设置回调结果
            PluginResult cr = new PluginResult(PluginResult.Status.INVALID_ACTION);
            callbackContext.sendPluginResult(cr);
        }
    } catch (JSONException e) {
        PluginResult cr = new PluginResult(PluginResult.Status.JSON_EXCEPTION);
        callbackContext.sendPluginResult(cr);
    } catch (Exception e) {
        LOG.e(TAG, "Uncaught exception from plugin", e);
        callbackContext.error(e.getMessage());
    }
}
```
上面的代码中从plugin.execute方法进入到插件实例的execute方法，在我们的例子中是Toast插件的execute方法。  
Toast插件的execute方法首先判断了传入的action，然后根据不同的action进行不同的处理。这也是大多数插件的通用设计，js一侧通过传入不同的action，来调用插件的不同逻辑。
```java
// Toast类内部
@Override
public boolean execute(String action, JSONArray args, final CallbackContext callbackContext) throws JSONException {
  if (ACTION_HIDE_EVENT.equals(action)) {
    returnTapEvent("hide", currentMessage, currentData, callbackContext);
    hide();
    callbackContext.success();
    return true;

  } else if (ACTION_SHOW_EVENT.equals(action)) {
    if (this.isPaused) {
      return true;
    }

    final JSONObject options = args.getJSONObject(0);
    final String msg = options.getString("message");
    final Spannable message = new SpannableString(msg);
    message.setSpan(
        new AlignmentSpan.Standard(Layout.Alignment.ALIGN_CENTER),
        0,
        msg.length() - 1,
        Spannable.SPAN_INCLUSIVE_INCLUSIVE);

    final String duration = options.getString("duration");
    final String position = options.getString("position");
    final int addPixelsY = options.has("addPixelsY") ? options.getInt("addPixelsY") : 0;
    final JSONObject data = options.has("data") ? options.getJSONObject("data") : null;
    final JSONObject styling = options.optJSONObject("styling");

    currentMessage = msg;
    currentData = data;
    // JavaScript运行在WebCore线程，如果需要与UI交互，则需要在UI线程运行
    cordova.getActivity().runOnUiThread(new Runnable() {
      public void run() {
        int hideAfterMs;
        if ("short".equalsIgnoreCase(duration)) {
          hideAfterMs = 2000;
        } else if ("long".equalsIgnoreCase(duration)) {
          hideAfterMs = 4000;
        } else {
          // assuming a number of ms
          hideAfterMs = Integer.parseInt(duration);
        }
        final android.widget.Toast toast = android.widget.Toast.makeText(
            IS_AT_LEAST_LOLLIPOP ? cordova.getActivity().getWindow().getContext() : cordova.getActivity().getApplicationContext(),
            message,
            "short".equalsIgnoreCase(duration) ? android.widget.Toast.LENGTH_SHORT : android.widget.Toast.LENGTH_LONG
        );

        if ("top".equals(position)) {
          toast.setGravity(GRAVITY_TOP, 0, BASE_TOP_BOTTOM_OFFSET + addPixelsY);
        } else  if ("bottom".equals(position)) {
          toast.setGravity(GRAVITY_BOTTOM, 0, BASE_TOP_BOTTOM_OFFSET - addPixelsY);
        } else if ("center".equals(position)) {
          toast.setGravity(GRAVITY_CENTER, 0, addPixelsY);
        } else {
          callbackContext.error("invalid position. valid options are 'top', 'center' and 'bottom'");
          return;
        }

        // if one of the custom layout options have been passed in, draw our own shape
        if (styling != null && Build.VERSION.SDK_INT >= 16) {

          // the defaults mimic the default toast as close as possible
          final String backgroundColor = styling.optString("backgroundColor", "#333333");
          final String textColor = styling.optString("textColor", "#ffffff");
          final Double textSize = styling.optDouble("textSize", -1);
          final double opacity = styling.optDouble("opacity", 0.8);
          final int cornerRadius = styling.optInt("cornerRadius", 100);
          final int horizontalPadding = styling.optInt("horizontalPadding", 50);
          final int verticalPadding = styling.optInt("verticalPadding", 30);

          GradientDrawable shape = new GradientDrawable();
          shape.setCornerRadius(cornerRadius);
          shape.setAlpha((int)(opacity * 255)); // 0-255, where 0 is an invisible background
          shape.setColor(Color.parseColor(backgroundColor));
          toast.getView().setBackground(shape);

          final TextView toastTextView;
          toastTextView = (TextView) toast.getView().findViewById(android.R.id.message);
          toastTextView.setTextColor(Color.parseColor(textColor));
          if (textSize > -1) {
            toastTextView.setTextSize(textSize.floatValue());
          }

          toast.getView().setPadding(horizontalPadding, verticalPadding, horizontalPadding, verticalPadding);

          // this gives the toast a very subtle shadow on newer devices
          if (Build.VERSION.SDK_INT >= 21) {
            toast.getView().setElevation(6);
          }
        }

        // On Android >= 5 you can no longer rely on the 'toast.getView().setOnTouchListener',
        // so created something funky that compares the Toast position to the tap coordinates.
        if (IS_AT_LEAST_LOLLIPOP) {
          getViewGroup().setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View view, MotionEvent motionEvent) {
              if (motionEvent.getAction() != MotionEvent.ACTION_DOWN) {
                return false;
              }
              if (mostRecentToast == null || !mostRecentToast.getView().isShown()) {
                getViewGroup().setOnTouchListener(null);
                return false;
              }

              float w = mostRecentToast.getView().getWidth();
              float startX = (view.getWidth() / 2) - (w / 2);
              float endX = (view.getWidth() / 2) + (w / 2);

              float startY;
              float endY;

              float g = mostRecentToast.getGravity();
              float y = mostRecentToast.getYOffset();
              float h = mostRecentToast.getView().getHeight();

              if (g == GRAVITY_BOTTOM) {
                startY = view.getHeight() - y - h;
                endY = view.getHeight() - y;
              } else if (g == GRAVITY_CENTER) {
                startY = (view.getHeight() / 2) + y - (h / 2);
                endY = (view.getHeight() / 2) + y + (h / 2);
              } else {
                // top
                startY = y;
                endY = y + h;
              }

              float tapX = motionEvent.getX();
              float tapY = motionEvent.getY();

              final boolean tapped = tapX >= startX && tapX <= endX &&
                  tapY >= startY && tapY <= endY;

              return tapped && returnTapEvent("touch", msg, data, callbackContext);
            }
          });
        } else {
          toast.getView().setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View view, MotionEvent motionEvent) {
              return motionEvent.getAction() == MotionEvent.ACTION_DOWN && returnTapEvent("touch", msg, data, callbackContext);
            }
          });
        }
        // trigger show every 2500 ms for as long as the requested duration
        _timer = new CountDownTimer(hideAfterMs, 2500) {
          public void onTick(long millisUntilFinished) {
            // see https://github.com/EddyVerbruggen/Toast-PhoneGap-Plugin/issues/116
            // and https://github.com/EddyVerbruggen/Toast-PhoneGap-Plugin/issues/120
//              if (!IS_AT_LEAST_PIE) {
//                toast.show();
//              }
          }
          public void onFinish() {
            returnTapEvent("hide", msg, data, callbackContext);
            toast.cancel();
          }
        }.start();

        mostRecentToast = toast;
        toast.show();

        PluginResult pr = new PluginResult(PluginResult.Status.OK);
        // 暂时keep住callback
        pr.setKeepCallback(true);
        callbackContext.sendPluginResult(pr);
      }
    });

    return true;
  } else {
    callbackContext.error("toast." + action + " is not a supported function. Did you mean '" + ACTION_SHOW_EVENT + "'?");
    return false;
  }
}
```
我们的示例中调用show这个action。show这个action主要做的事情是在UI线程设置一段运行的代码，这段代码主要是在UI线程显示toast，然后调用callbackContext.sendPluginResult方法，发送插件运行结果，该方法调用会引发js侧的方法回调。设置好了UI线程调用的代码后，Toast类的execute方法返回了一个true，结束了这个方法。后续会依次返回pluginManager.exec、CordovaBridge.jsExec、SystemExposedJsApi.exec方法。当SystemExposedJsApi.exec方法返回的时候，对应了js侧的nativeApiProvider.get().exec()方法的返回。

此时的函数调用栈情况大概如下：
```javascript
// 以下方法运行在WebCore线程
window.plugins.toast.show()
  Toast.prototype.showWithOptions()
    cordova.exec()
      nativeApiProvider.get().exec() // 从这里进入原生代码
        SystemExposedJsApi.exec()
          CordovaBridge.jsExec()
            PluginManager.exec()
              CordovaPlugin.execute()
                Toast.execute() // 在这里执行业务逻辑
              CordovaPlugin.execute()
            PluginManager.exec()
          CordovaBridge.jsExec()
        SystemExposedJsApi.exec() // 从原生代码返回
      nativeApiProvider.get().exec()
    cordova.exec()
  Toast.prototype.showWithOptions()
window.plugins.toast.show()
```

## native中的线程与js中的异步
在继续讲述Native插件如何对js进行回调前，先介绍一下native中的线程与js中的异步。  
javaScript都是运行在WebCore线程上的，所以上述从js到native再到插件的execute方法的调用都发生在WebCore线程，当插件内需要操作UI时，则需要切换到UI线程,[详细可点击此处查看文档](https://cordova.apache.org/docs/en/2.6.0/guide/plugin-development/android/#threading)。在Toast插件的show action中就是如此，因为要在UI上显示toast，所以切换到UI线程来执行显示toast的代码，并且在UI线程上发送了插件运行结果，进而对js一侧进行了回调，所以在js一侧插件回调函数的执行是异步的。   
对于部分插件来说，并不需要在execute方法内切换到UI线程，可能在WebCore线程上执行完业务逻辑就马上调用callbackContext.sendPluginResult方法发送插件运行结果了，这种情况下js侧的插件回调函数是否就不是异步的呢？答案是插件回调函数依然是异步，继续深入callbackContext.sendPluginResult方法的调用栈，就会发现最终在BridgeMode类的onNativeToJsMessageAvailable方法中，js的回调最终还是被切换到UI线程中进行的（后续Native对js的回调会详细解释），所以无论插件的execute方法是否进行了线程切换，最后native对js的调用都是从WebCore线程切换到UI线程的，所以造成js侧的插件回调函数总是异步运行的。

## native端对js进行的回调
这里任然以Toast插件的回调为例子进行说明

Toast插件的show action处理中，先切换到了UI线程，进行toast的显示（js运行在WebCore线程，如果要操作UI必须切换到UI线程）。toast显示后调用callbackContext.sendPluginResult来发送插件运行结果。从这里开始了对js的回调过程。

```java
// 在Toast插件内
@Override
  public boolean execute(String action, JSONArray args, final CallbackContext callbackContext) throws JSONException {
    if (ACTION_HIDE_EVENT.equals(action)) {
      ...
    } else if (ACTION_SHOW_EVENT.equals(action)) {
      ...
      // JavaScript运行在WebCore线程，如果需要与UI交互，则需要在UI线程运行
      cordova.getActivity().runOnUiThread(new Runnable() {
        public void run() {

          ...

          toast.show();
          // 创建一个插件运行结果
          PluginResult pr = new PluginResult(PluginResult.Status.OK);
          // 暂时keep住callback
          pr.setKeepCallback(true);
          // 调用callbackContext.sendPluginResult发送插件运行结果
          callbackContext.sendPluginResult(pr);
        }
      });

      return true;
    } else {
      callbackContext.error("toast." + action + " is not a supported function. Did you mean '" + ACTION_SHOW_EVENT + "'?");
      return false;
    }
  }
```

PluginResult实例代表插件运行结果，callbackContext实例则代表着native对js的回调。sendPluginResult方法内部调用了webView.sendPluginResult方法，将PluginResult传递给webview。webView.sendPluginResult又调用nativeToJsMessageQueue.addPluginResult方法把PluginResult和callbackId传递给了sendPluginResult又调用nativeToJsMessageQueue。

```java
// CallBackContent类内部
// 插件运行完毕，设置结果，同时将结果实例传入webview
public void sendPluginResult(PluginResult pluginResult) {
    synchronized (this) {
        if (finished) {
            LOG.w(LOG_TAG, "Attempted to send a second callback for ID: " + callbackId + "\nResult was: " + pluginResult.getMessage());
            return;
        } else {
            // 如果keepCallback为true，则finished为false，否则为true
            finished = !pluginResult.getKeepCallback();
        }
    }
    webView.sendPluginResult(pluginResult, callbackId);
}
```
```java
// CordovaWebViewImpl类内部
// 发送一个插件结果实例，通常是由callbackContext类调用
@Override
public void sendPluginResult(PluginResult cr, String callbackId) {
    nativeToJsMessageQueue.addPluginResult(cr, callbackId);
}
```

nativeToJsMessageQueue实例存储着native发送到js的消息队列。addPluginResult方法内会把PluginResult实例封装成JsMessage实例，然后调用enqueueMessage方法将JsMessage实例推入队列，然后马上对消息队列进行处理。队列内的消息是由activeBridgeMode实例的onNativeToJsMessageAvailable方法进行处理的。

```java
// NativeToJsMessageQueue类内部
public void addPluginResult(PluginResult result, String callbackId) {
    if (callbackId == null) {
        LOG.e(LOG_TAG, "Got plugin result with no callbackId", new Throwable());
        return;
    }
    // Don't send anything if there is no result and there is no need to
    // clear the callbacks.
    boolean noResult = result.getStatus() == PluginResult.Status.NO_RESULT.ordinal();
    boolean keepCallback = result.getKeepCallback();
    // 如果结果状态是noResult，同时keepCallback，则没有必要回调js
    if (noResult && keepCallback) {
        return;
    }
    // 将插件结果封装成jsMessage对象
    JsMessage message = new JsMessage(result, callbackId);
    // 通常为false
    if (FORCE_ENCODE_USING_EVAL) {
        StringBuilder sb = new StringBuilder(message.calculateEncodedLength() + 50);
        message.encodeAsJsMessage(sb);
        message = new JsMessage(sb.toString());
    }
    // 将消息入队列
    enqueueMessage(message);
}
// 添加消息进入队列并且使用mode实例进行处理
private void enqueueMessage(JsMessage message) {
    synchronized (this) {
        if (activeBridgeMode == null) {
            LOG.d(LOG_TAG, "Dropping Native->JS message due to disabled bridge");
            return;
        }
        queue.add(message);
        if (!paused) {
            // 如果当前没有暂停,则调用当前激活的mode实例来处理
            activeBridgeMode.onNativeToJsMessageAvailable(this);
        }
    }
}
```
activeBridgeMode实例代表着native调用js的具体实现模式，通常会在cordova.js初始化的时候，通过js调用原生方法的方式设置为EvalBridgeMode类实例。所以这里查看EvalBridgeMode类的onNativeToJsMessageAvailable方法

```java
// EvalBridgeMode类内部
@Override
public void onNativeToJsMessageAvailable(final NativeToJsMessageQueue queue) {
    // plugin的execute方法运行在WebCore线程，在WebCore线程中return
    // 而执行callback的webView方法必须在UI线程运行的，所以在js中callback是异步的
    String threadName = Thread.currentThread().getName();
    Log.d(LOG_TAG, "in onNativeToJsMessageAvailable: " + threadName);
    cordova.getActivity().runOnUiThread(new Runnable() {
        public void run() {
            String threadName = Thread.currentThread().getName();
            Log.d(LOG_TAG, "beforeCallback: " + threadName);
            // 将jsMessage实例转换成js代码字符串
            String js = queue.popAndEncodeAsJs();
            if (js != null) {
                engine.evaluateJavascript(js, null);
            }
        }
    });
}
```
值得注意的是onNativeToJsMessageAvailable方法内，强制切换到UI线程，在UI线程内将队列内的jsMessage实例转换成js代码字符串，然后将js代码字符串传入engine。engine.evaluateJavascript调用了android原生的webView.evaluateJavascript来执行js代码字符串，最后在js一侧触发了插件回调函数。

onNativeToJsMessageAvailable方法内，强制切换到UI线程的原因是，webview的evaluateJavascript方法只能在UI线程进行调用。这也造成了js一侧插件调用和回调函数总是异步的特性。

最后附上native回调js的一个函数调用过程
```java
// 以下方法调用栈可能在任一线程
callbackContext.sendPluginResult(pluginResult) // 发送插件运行实例
  webView.sendPluginResult(pluginResult, callbackId); 
    nativeToJsMessageQueue.addPluginResult(cr, callbackId); // 在这里把PluginResult封装成JsMessage
      nativeToJsMessageQueue.enqueueMessage(jsmessage)
        activeBridgeMode.onNativeToJsMessageAvailable() // 在这个方法内切换到UI线程，运行执行js的逻辑
      nativeToJsMessageQueue.enqueueMessage(jsmessage)
    nativeToJsMessageQueue.addPluginResult(cr, callbackId);
  webView.sendPluginResult(pluginResult, callbackId);
callbackContext.sendPluginResult(pluginResult)

// 以下方法调用栈运行在UI线程，由activeBridgeMode.onNativeToJsMessageAvailable开启
queue.popAndEncodeAsJs(); // 在这里将jsMessage实例转换成js代码字符串
engine.evaluateJavascript(js, null);
  webView.evaluateJavascript(js, callback); // 进入js侧的回调函数
engine.evaluateJavascript(js, null);

// 调用webView.evaluateJavascript(js, callback);之后，js侧的函数调用栈
cordova.callbackFromNative(callbackId, isSuccess, status, args, keepCallback)
  callback.success or callback.fail // 根据isSuccess决定调用成功回调函数，或者失败回调函数。
cordova.callbackFromNative(callbackId, isSuccess, status, args, keepCallback)
```

以上就是js调用Native，以及Native回调Js的大概过程