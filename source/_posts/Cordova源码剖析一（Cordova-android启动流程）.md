---
title: Cordova源码剖析一（Cordova android启动流程）
date: 2019-06-23 14:28:04
categories:
  - cordova系列
tags: 
  - 前端
  - cordova
  - 源码剖析
comments: false
---
这个系列的文章主要用来研究Cordova Android中的jsbridge的原理。系列文章主要分为四部分：
1. android native的启动流程（插件初始化，以及_cordovaNative对象初始化）
2. android cordova.js的启动流程
3. js调用native的过程
4. native调用js的过程

这篇文章主要包括Cordova Android native的启动流程和一些Cordova Android中重要类的说明两大部分。

## cordova android native的启动流程
首先运行CordovaActivity的onCreate方法，其中调用loadConfig()方法，用来从config.xml文件中加载cordova插件
```java
// current stack：CordovaActivity.onCreate
  @Override
  public void onCreate(Bundle savedInstanceState) {
    loadConfig();
    ...
  }
```
### 阶段一：loadConfig
loadConfig方法主要作用是从config.xml文件中加载cordova插件信息，并将这些信息放置到CordovaActivity中的pluginEntries属性中保存
```java
// current stack: CordovaActivity.onCreate -> CordovaActivity.loadConfig
protected void loadConfig() {
  ConfigXmlParser parser = new ConfigXmlParser();
  parser.parse(this);
  preferences = parser.getPreferences();
  preferences.setPreferencesBundle(getIntent().getExtras());
  launchUrl = parser.getLaunchUrl(); // 从config.xml中获取html的加载路径
  pluginEntries = parser.getPluginEntries(); // 从config.xml中加载好的插件信息保存在CordovaActivity.pluginEntries
  Config.parser = parser;
}
```
#### ConfigXmlParser类：通过内部的parse方法对/res/xml/config.xml文件进行解析。
在ConfigXmlParser类的handleStartTag方法中，可以看到解析config.xml文件的过程：
1. 解析每一个feature标签并将其中的name属性值当做插件的名称
2. 解析name属性为'android-package'或者'package'的param标签内的value属性值作为插件的class名称
3. 解析param标签内的onload属性是否为true等等

解析完后在handleEndTag方法中，将解析出来的service(插件名)、pluginClass(插件类)、onload(是否立即初始化插件)等信息封装成PluginEntry实例，添加到CordovaActivity.pluginEntries中保存。
**pluginEntries**是一个ArrayList<PluginEntry>。用来保存插件信息
```java
// current stack: CordovaActivity.onCreate -> CordovaActivity.loadConfig -> ConfigXmlParser.parse -> ConfigXmlParser.handleEndTag
public void handleEndTag(XmlPullParser xml) {
  String strNode = xml.getName();
  if (strNode.equals("feature")) {
    pluginEntries.add(new PluginEntry(service, pluginClass, onload));

    service = "";
    pluginClass = "";
    insideFeature = false;
    onload = false;
  }
}
```
至此，loadConfig函数的重要作用已经结束。

### 阶段二：loadUrl
执行完loadConfig之后，会执行CordovaActivity.loadUrl方法，进入初始化的第二个阶段。该阶段会完成webview的初始化和javaScriptInterface的注册。
loadUrl中，先判断webview是否为null，若是null，则调用init方法开始初始化
```java
// current stack: CordovaActivity.loadUrl
public void loadUrl(String url) {
    if (appView == null) {
        init(); // 初始化webview，在这里完成jsbridge的初始化
    }

    // If keepRunning
    this.keepRunning = preferences.getBoolean("KeepRunning", true);

    appView.loadUrlIntoView(url, true); // 加载html文档
}
```
init方法中完成了webview和jsbridge的初始化，深入init方法进行查看
```java
// current stack: CordovaActivity.loadUrl -> CordovaActivity.init
protected void init() {
    appView = makeWebView(); // 创建webview实例
    createViews();
    if (!appView.isInitialized()) {
        appView.init(cordovaInterface, pluginEntries, preferences); // NOTE: 关键方法，初始化webview
    }
    cordovaInterface.onCordovaInit(appView.getPluginManager());

    // Wire the hardware volume controls to control media if desired.
    String volumePref = preferences.getString("DefaultVolumeStream", "");
    if ("media".equals(volumePref.toLowerCase(Locale.ENGLISH))) {
        setVolumeControlStream(AudioManager.STREAM_MUSIC);
    }
}
```
appView声明类型是CordovaWebView，CordovaWebView是一个接口。appView对象实际上是CordovaWebView的实现类CordovaWebViewImpl。   
appView.init方法完成了三件非常重要的事情：
1. 传入了cordovaInterface, pluginEntries, preferences三个参数，完成**PluginManager**的初始化（PluginManager是一个非常重要的对象，待会详细介绍）
2. 生成了nativeToJsMessageQueue对象，该对象负责Native到JS的通讯，完成Native调用JS方法的实现。（NativeToJsMessageQueue类非常重要，后面会详细介绍）
3. 调用engine.init方法，完成javaScriptInterface在webview的注册，生成window._cordovaNative对象
```java
// current stack: CordovaActivity.loadUrl -> CordovaActivity.init -> CordovaWebViewImpl.init
@SuppressLint("Assert")
@Override
public void init(CordovaInterface cordova, List<PluginEntry> pluginEntries, CordovaPreferences preferences) {
    if (this.cordova != null) {
        throw new IllegalStateException();
    }
    this.cordova = cordova;
    this.preferences = preferences;
    pluginManager = new PluginManager(this, this.cordova, pluginEntries); // NOTE: 关键方法，根据pluginEntries实例化pluginManager
    resourceApi = new CordovaResourceApi(engine.getView().getContext(), pluginManager);

    // 初始化NativeToJsMessageQueue实例，并且注册两种nativeTojs的调用方式对象
    nativeToJsMessageQueue = new NativeToJsMessageQueue(); // NOTE: 关键方法
    nativeToJsMessageQueue.addBridgeMode(new NativeToJsMessageQueue.NoOpBridgeMode()); // 使用js轮询的方式调用JS
    nativeToJsMessageQueue.addBridgeMode(new NativeToJsMessageQueue.LoadUrlBridgeMode(engine, cordova)); // 使用webview.loadUrl的方式调用JS

    if (preferences.getBoolean("DisallowOverscroll", false)) {
        engine.getView().setOverScrollMode(View.OVER_SCROLL_NEVER);
    }
    // NOTE: 初始化CordovaWebViewEngine类，CordovaWebview中所有关于Native和JS的通讯由这个类负责
    engine.init(this, cordova, engineClient, resourceApi, pluginManager, nativeToJsMessageQueue); // NOTE: 关键方法，注册JSInterface
    // This isn't enforced by the compiler, so assert here.
    assert engine.getView() instanceof CordovaWebViewEngine.EngineView;

    pluginManager.addService(CoreAndroid.PLUGIN_NAME, "org.apache.cordova.CoreAndroid");
    pluginManager.init();
}
```
**PluginManager**类内部维护了一个LinkedHashMap<String, PluginEntry>，其中key是插件名，value是插件实例或插件信息，在后续js调用cordova插件的过程，PluginManager会根据js端传过来的插件名，从内部的map中获取到相应的插件实例，然后执行插件的execute方法。所以**PluginManager是插件的管理器，负责查找、调用、维护插件实例**。关于PluginManager类的详细讲解稍后会有。    
**NativeToJsMessageQueue**类负责Native端到JS的通信，其内部又定义了BridgeMode抽象类以及实现了四个子类。它们分别代表了四种Native调用JS的方式。关于NativeToJsMessageQueue的讲解稍后再看
先深入engine.init方法中查看。    
engine的声明类型是**CordovaWebViewEngine**接口，实现类是**SystemWebViewEngine**。**CordovaWebview所有有关JS的通讯都委托给这个类进行处理**    
engine.init方法完成了三个关键步骤：
1. 在nativeToJsMessageQueue中注册其他两种BridgeMode。至此注册了四种BridgeMode：NoOpBridgeMode， LoadUrlBridgeMode，OnlineEventsBridgeMode，EvalBridgeMode。后续cordova.js会在启动时调用原生方法，指定nativeToJsMessageQueue使用EvalBridgeMode。
2. 创建了CordovaBridge对象，SystemWebViewEngine会将Native跟JS的通讯都委托给这个类实现
3. 调用exposeJsInterface方法注册window._cordovaNative对象
```java
// current stack: CordovaActivity.loadUrl -> CordovaActivity.init -> CordovaWebViewImpl.init -> SystemWebViewEngine.init
@Override
public void init(CordovaWebView parentWebView, CordovaInterface cordova, CordovaWebViewEngine.Client client,
          CordovaResourceApi resourceApi, PluginManager pluginManager,
          NativeToJsMessageQueue nativeToJsMessageQueue) {
    ...
    this.pluginManager = pluginManager;
    this.nativeToJsMessageQueue = nativeToJsMessageQueue;
    // NOTE: 调用SystemWebView的init方法，其内部完成了SystemWebChromeClient类的初始化
    // SystemWebChromeClient类实现了onJsPrompt方法用于拦截js Prompt弹窗。使得JS通过可以通过Prompt来调用Native的方式
    webView.init(this, cordova);

    initWebViewSettings();

    // 继续在nativeToJsMessageQueue中注册两种NativeTojs的方式
    nativeToJsMessageQueue.addBridgeMode(new NativeToJsMessageQueue.OnlineEventsBridgeMode(new NativeToJsMessageQueue.OnlineEventsBridgeMode.OnlineEventsBridgeModeDelegate() {
        @Override
        public void setNetworkAvailable(boolean value) {
            //sometimes this can be called after calling webview.destroy() on destroy()
            //thus resulting in a NullPointerException
            if(webView!=null) {
                webView.setNetworkAvailable(value); 
            }
        }
        @Override
        public void runOnUiThread(Runnable r) {
            SystemWebViewEngine.this.cordova.getActivity().runOnUiThread(r);
        }
    }));
    // EvalBridgeMode类非常关键，Native调用js最终是通过这里类实例来完成。
    nativeToJsMessageQueue.addBridgeMode(new NativeToJsMessageQueue.EvalBridgeMode(this, cordova));
    // NOTE: 关键步骤,创建CordovaBridge实例，CordovaWebViewEngine类中关于Native和JS的通讯委托给这个类负责
    bridge = new CordovaBridge(pluginManager, nativeToJsMessageQueue);
    // NOTE: 关键步骤，在JS中暴露原生对象
    exposeJsInterface(webView, bridge);
}
```
继续深入exposeJsInterface方法，exposeJsInterface方法内传入CordovaBridge对象，然后新建SystemExposedJsApi对象，最后把这个对象注册到JS侧的window._cordovaNative上，后续cordova.js中会使用这个原生对象来调用原生方法
```java
// current stack: CordovaActivity.loadUrl -> CordovaActivity.init -> CordovaWebViewImpl.init -> SystemWebViewEngine.init -> SystemWebViewEngine.exposeJsInterface
@SuppressLint("AddJavascriptInterface")
private static void exposeJsInterface(WebView webView, CordovaBridge bridge) {
    SystemExposedJsApi exposedJsApi = new SystemExposedJsApi(bridge);
    // 调用addJavascriptInterface方法在webview内注册原生对象
    webView.addJavascriptInterface(exposedJsApi, "_cordovaNative");
}
```
SystemExposedJsApi类中一共向JS注册了三个方法
```java
class SystemExposedJsApi implements ExposedJsApi {
    private final CordovaBridge bridge;

    SystemExposedJsApi(CordovaBridge bridge) {
        this.bridge = bridge;
    }

    // JS调用原生插件时使用
    @JavascriptInterface
    public String exec(int bridgeSecret, String service, String action, String callbackId, String arguments) throws JSONException, IllegalAccessException {
        return bridge.jsExec(bridgeSecret, service, action, callbackId, arguments);
    }

    // JS中设置NativeToJS的通讯方式
    @JavascriptInterface
    public void setNativeToJsBridgeMode(int bridgeSecret, int value) throws IllegalAccessException {
        bridge.jsSetNativeToJsBridgeMode(bridgeSecret, value);
    }

    @JavascriptInterface
    public String retrieveJsMessages(int bridgeSecret, boolean fromOnlineEvent) throws IllegalAccessException {
        return bridge.jsRetrieveJsMessages(bridgeSecret, fromOnlineEvent);
    }
}
```

运行完loadUrl，Native代码完成了_cordovaNative原生对象的注册、完成了SystemWebViewEngine、CordovaBridge、NativeToJsMessageQueue等对象的初始化。为后续JS调用Native做好了准备。

### Cordova中低版本android的JSToNative通讯手段
@JavascriptInterface的方式只有在android4.2及其以上的android版本才支持，也就是说当android4.2及其以上时，才会通过注册window._cordovaNative的方式来实现JS到Native的调用。当版本小于4.2时或者window._cordovaNative原生对象注册失败时，js会通过prompt的方式来实现JS对Native的调用。具体的实现方式如下：
1. js调用window.prompt方法来尝试进行弹窗，并传入特定前缀的value
2. Native端在SystemWebChromeClient类中实现onJsPrompt方法来拦截js弹窗，然后通过解析value是否携带特定前缀来判断是否要调用Native方法做出响应。

SystemWebChromeClient类初始化的入口在上面提到的 engine.init 方法内

```java
// current stack: CordovaActivity.loadUrl -> CordovaActivity.init -> CordovaWebViewImpl.init -> SystemWebViewEngine.init
public void init(CordovaWebView parentWebView, CordovaInterface cordova, CordovaWebViewEngine.Client client,
    CordovaResourceApi resourceApi, PluginManager pluginManager,
    NativeToJsMessageQueue nativeToJsMessageQueue) {
    ...
    // 调用SystemWebView的init方法，其内部完成了SystemWebChromeClient类的初始化
    // SystemWebChromeClient类实现了onJsPrompt用于拦截
    webView.init(this, cordova);
    ...
}
```
深入到webView.init内部继续查看
```java
// current stack: CordovaActivity.loadUrl -> CordovaActivity.init -> CordovaWebViewImpl.init -> SystemWebViewEngine.init -> SystemWebView.init
void init(SystemWebViewEngine parentEngine, CordovaInterface cordova) {
    this.cordova = cordova;
    this.parentEngine = parentEngine;
    if (this.viewClient == null) {
        setWebViewClient(new SystemWebViewClient(parentEngine));
    }

    if (this.chromeClient == null) {
        // 新建一个SystemWebChromeClient实例，传入的是SystemWebViewEngine对象。cordovaWebview中所有Native和js的通讯都委托给SystemWebViewEngine来实现。
        setWebChromeClient(new SystemWebChromeClient(parentEngine));
    }
}
```
SystemWebChromeClient中onJsPrompt方法实现如下：
```java
// 在SystemWebChromeClient类内部
@Override
public boolean onJsPrompt(WebView view, String origin, String message, String defaultValue, final JsPromptResult result) {
    // Unlike the @JavascriptInterface bridge, this method is always called on the UI thread.
    // 当@JavascriptInterfacew无法使用时的备用通讯手段，将js prompt传过来的value委托给engine内部的bridge实例（CordovaBridge）进行处理
    String handledRet = parentEngine.bridge.promptOnJsPrompt(origin, message, defaultValue);
    if (handledRet != null) {
        // 调用result.confirm后对应js侧的window.prompt方法返回
        result.confirm(handledRet);
    } else {
        dialogsHelper.showPrompt(message, defaultValue, new CordovaDialogsHelper.Result() {
            @Override
            public void gotResult(boolean success, String value) {
                if (success) {
                    result.confirm(value);
                } else {
                    result.cancel();
                }
            }
        });
    }
    return true;
}
```
promptOnJsPrompt方法对js传过来的value有一套事先协商好的通讯协议，当value以特定字符开头时，promptOnJsPrompt会执行相应的逻辑。
promptOnJsPrompt的方法定义如下：
```java
// 在CordovaBridge类内部
public String promptOnJsPrompt(String origin, String message, String defaultValue) {
    if (defaultValue != null && defaultValue.length() > 3 && defaultValue.startsWith("gap:")) {
        // 以gap:开头则尝试调用Cordova 插件
        JSONArray array;
        try {
            array = new JSONArray(defaultValue.substring(4));
            int bridgeSecret = array.getInt(0);
            String service = array.getString(1);
            String action = array.getString(2);
            String callbackId = array.getString(3);
            // NOTE: 调用插件
            String r = jsExec(bridgeSecret, service, action, callbackId, message);
            return r == null ? "" : r;
        } catch (JSONException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        return "";
    }
    // Sets the native->JS bridge mode.
    else if (defaultValue != null && defaultValue.startsWith("gap_bridge_mode:")) {
        // 以gap_bridge_mode:开头则尝试设置NativeToJs的BridgeMode
        try {
            int bridgeSecret = Integer.parseInt(defaultValue.substring(16));
            jsSetNativeToJsBridgeMode(bridgeSecret, Integer.parseInt(message));
        } catch (NumberFormatException e){
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        return "";
    }
    // Polling for JavaScript messages
    else if (defaultValue != null && defaultValue.startsWith("gap_poll:")) {
      ...
    }
    else if (defaultValue != null && defaultValue.startsWith("gap_init:")) {
      // NOTE: 以gap_init:开头，初始化NativeToJsMessageQueue类实例中的NativeToJs mode
      if (pluginManager.shouldAllowBridgeAccess(origin)) {
          // Enable the bridge
          int bridgeMode = Integer.parseInt(defaultValue.substring(9));
          // 设置NativeToJsMessageQueue类实例中的NativeToJs mode
          jsMessageQueue.setBridgeMode(bridgeMode);
          // Tell JS the bridge secret.
          int secret = generateBridgeSecret();
          return ""+secret;
      } else {
          LOG.e(LOG_TAG, "gap_init called from restricted origin: " + origin);
      }
      return "";
    }
    return null;
}
```

## Cordova Android中一些重要的类
cordova android中有一些类扮演了比较重要的角色，下面是这些类的剖析

CordovaActivity类：  
功能：在onCreate生命周期中实现了cordova插件和cordovaWebView的初始化逻辑，是cordova应用启动的入口。  
以下是一些重要属性和重要方法：
```java
public class CordovaActivity extends Activity {
  // 加载html页面的webview实例
  protected CordovaWebView appView;
  // If true, then the JavaScript and native code continue to run in the background（default = true）
  protected boolean keepRunning = true;
  // 存储从config.xml中加载到的preference信息
  protected CordovaPreferences preferences;
  // webview需要加载html url，从config.xml中获取
  protected String launchUrl;
  // 存储从config.xml中加载到的插件信息
  protected ArrayList<PluginEntry> pluginEntries;

 /**
  * 调用loadConfig函数，实现从config.xml配置文件加载cordova插件、launchUrl、preference等重要信息
  */
  @Override
  public void onCreate(Bundle savedInstanceState) {
    loadConfig();
    ...
  }
 
 /**
  * 通过ConfigXmlParser实现对config.xml文件的解析
  */
  protected void loadConfig() {
    ....
  }
 /**
  * 创建webview实例，调用webview的init方法初始化webview，并webview的init方法内完成cordova插件的
  * 初始化和window._cordovaNative原生对象的注册
  */
  protected void init() {
    ...
  }
}
```

CordovaActivity类里面的appView非常重要，这个实例属于CordovaWebViewImpl类，CordovaActivity将关于JS和Native的交互，以及html页面的管理都委托给这个实例进行。

```java
public class CordovaWebViewImpl implements CordovaWebView {
  // 插件管理器，其内部有一个以插件名为key，插件实例为value的map实例，用来获取、维护、初始化插件实例
  private PluginManager pluginManager;

  // 一个内置的cordova插件实例，实现了对返回键、菜单键、音量键、以及提供了一个稳定的js callback通道来向js传递各种事件
  private CoreAndroid appPlugin;
  // Native向js发送消息的队列
  private NativeToJsMessageQueue nativeToJsMessageQueue;
  // SystemWebViewEngine的实例，webview会将Native与JS之间的通讯委托给这个实例进行
  protected final CordovaWebViewEngine engine;
  /**
    * 在这个方法中完成了pluginManager的初始化并且注册了内置插件CoreAndroid、
    * 完成了engine实例的初始化，通过engine.init方法，完成了cordova native一侧的jsbridge初始化
    * 完成了NativeToJsMessageQueue的初始化
    */ 
  public void init() {
    ...
  }

  // 该方法由CallbackContent（后续会讲到）实例调用，传入一个插件运行结果，和callbackId
  @Override
  public void sendPluginResult(PluginResult cr, String callbackId) {
    nativeToJsMessageQueue.addPluginResult(cr, callbackId);
  }
}
```

PluginManager类内部有一个以插件名为key，插件实例为value的map实例，该类用来获取、维护、初始化插件实例。    
PluginManager类有几个非常重要的方法：
1. **exec**方法，作为外部调用指定插件的指定action的接口方法
2. addService方法，外部向PluginManager注册插件
```java
public class PluginManager {
  // 插件实例map
  private final LinkedHashMap<String, CordovaPlugin> pluginMap = new LinkedHashMap<String, CordovaPlugin>();
  // 插件信息map
  private final LinkedHashMap<String, PluginEntry> entryMap = new LinkedHashMap<String, PluginEntry>();
  // 构造函数，传入CordovaActivity内的pluginEntries，用以初始化PluginManager
  public PluginManager(CordovaWebView cordovaWebView, CordovaInterface cordova, Collection<PluginEntry> pluginEntries) {
    this.ctx = cordova;
    this.app = cordovaWebView;
    setPluginEntries(pluginEntries);
  }

  public void setPluginEntries(Collection<PluginEntry> pluginEntries) {
    if (isInitialized) {
        this.onPause(false);
        this.onDestroy();
        pluginMap.clear();
        entryMap.clear();
    }
    // 初始化时isInitialized为false，只运行for循环
    for (PluginEntry entry : pluginEntries) {
        // 遍历pluginEntries，一个一个添加
        addService(entry);
    }
    if (isInitialized) {
        startupPlugins();
    }
  }

  // 将插件添加到两个相应的map内
  public void addService(PluginEntry entry) {
    // 添加到entryMap
    this.entryMap.put(entry.service, entry);
    if (entry.plugin != null) {
        entry.plugin.privateInitialize(entry.service, ctx, app, app.getPreferences());
        // 将插件实例添加到pluginMap
        pluginMap.put(entry.service, entry.plugin);
    }
  }

  // 供外部实例根据插件名来获取插件实例
  public CordovaPlugin getPlugin(String service) {
    CordovaPlugin ret = pluginMap.get(service);
    if (ret == null) {
      // 如果插件实例还没有初始化，
      PluginEntry pe = entryMap.get(service);
      if (pe == null) {
          return null;
      }
      if (pe.plugin != null) {
          ret = pe.plugin;
      } else {
          // 通过反射初始化插件实例
          ret = instantiatePlugin(pe.pluginClass);
      }
      ret.privateInitialize(service, ctx, app, app.getPreferences());
      // 保存插件实例
      pluginMap.put(service, ret);
    }
    return ret;
  }

  // NOTE: 根据传入的插件名，action，callbackId，json字符串参数来执行相关插件。该方法是执行插件的直接方法，由CordovaBridge实例调用
  public void exec(final String service, final String action, final String callbackId, final String rawArgs) {
    // 按照插件名获取插件实例
    CordovaPlugin plugin = getPlugin(service);
    if (plugin == null) {
      LOG.d(TAG, "exec() call to unknown plugin: " + service);
      PluginResult cr = new PluginResult(PluginResult.Status.CLASS_NOT_FOUND_EXCEPTION);
      app.sendPluginResult(cr, callbackId);
      return;
    }
    // NOTE: 初始化一个CallbackContext 实例，CallbackContext类负责处理NativeToJS的方法调用。
    CallbackContext callbackContext = new CallbackContext(callbackId, app);
    try {
      long pluginStartTime = System.currentTimeMillis();
      // 执行插件的execute方法，wasValidAction为false说明是一个非法action调用
      // rawArgs此时任然是一个json格式字符串，CordovaPlugin类的execute方法内对其进行了反序列化
      boolean wasValidAction = plugin.execute(action, rawArgs, callbackContext);
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
}
```

SystemWebViewEngine类：cordovawebview将所有Native和js的通讯都委托这个类进行。该类完成了cordova native一侧的jsbridge的初始化（无论是基于js Prompt的jsToNative方式还是基于javascriptinterface的原生对象注册，都是在该类的init方法中完成）。该类还初始化了CordovaBridge实例，这个实例非常重要，SystemWebViewEngine类会将Native和js通讯都委托给CordovaBridge来实现。

```java
public class SystemWebViewEngine implements CordovaWebViewEngine {
  // 加载html的webview实例，其内部的chromeClient实例实现了对js Prompt弹窗的拦截，使得JS可以通过Prompt弹窗的方式对Native进行调用
  protected final SystemWebView webView;
  // js调用native方式时，总是经由CordovaBridge进行处理
  protected CordovaBridge bridge;
 /**
  * 完成了Prompt base的js to native交互方式
  * 初始化CordovaBridge实例，通过javascriptInterface向window注册_cordovaNative对象
  */
  @Override
  public void init(CordovaWebView parentWebView, CordovaInterface cordova, CordovaWebViewEngine.Client client,
            CordovaResourceApi resourceApi, PluginManager pluginManager,
            NativeToJsMessageQueue nativeToJsMessageQueue) {
    if (this.cordova != null) {
        throw new IllegalStateException();
    }
    // Needed when prefs are not passed by the constructor
    if (preferences == null) {
        preferences = parentWebView.getPreferences();
    }
    this.parentWebView = parentWebView;
    this.cordova = cordova;
    this.client = client;
    this.resourceApi = resourceApi;
    this.pluginManager = pluginManager;
    this.nativeToJsMessageQueue = nativeToJsMessageQueue;
    // 调用SystemWebView的init方法，其内部完成了SystemWebChromeClient类的初始化
    // SystemWebChromeClient类实现了onJsPrompt用于拦截js Prompt弹窗，实现JS通过Prompt来调用Native的方式
    webView.init(this, cordova);

    initWebViewSettings();

    // 继续注册两种调用js的方式
    nativeToJsMessageQueue.addBridgeMode(new NativeToJsMessageQueue.OnlineEventsBridgeMode(new NativeToJsMessageQueue.OnlineEventsBridgeMode.OnlineEventsBridgeModeDelegate() {
        @Override
        public void setNetworkAvailable(boolean value) {
            //sometimes this can be called after calling webview.destroy() on destroy()
            //thus resulting in a NullPointerException
            if(webView!=null) {
                webView.setNetworkAvailable(value); 
            }
        }
        @Override
        public void runOnUiThread(Runnable r) {
            SystemWebViewEngine.this.cordova.getActivity().runOnUiThread(r);
        }
    }));
    nativeToJsMessageQueue.addBridgeMode(new NativeToJsMessageQueue.EvalBridgeMode(this, cordova));
    // NOTE: 关键步骤,创建CordovaBridge实例，CordovaWebViewEngine类中关于Native和JS的通讯委托给这个类负责
    bridge = new CordovaBridge(pluginManager, nativeToJsMessageQueue);
    // NOTE: 关键步骤，在JS中暴露原生对象
    exposeJsInterface(webView, bridge);
  }

 /**
  * 初始化SystemExposedJsApi实例，注册window._cordovaNative原生对象
  */
  @SuppressLint("AddJavascriptInterface")
  private static void exposeJsInterface(WebView webView, CordovaBridge bridge) {
    SystemExposedJsApi exposedJsApi = new SystemExposedJsApi(bridge);
    webView.addJavascriptInterface(exposedJsApi, "_cordovaNative");
  }
}
```

SystemExposedJsApi类，SystemExposedJsApi内部通过@javascriptInterface注解的方式，向webview暴露了三个原生方法exec、setNativeToJsBridgeMode、retrieveJsMessages
```java
class SystemExposedJsApi implements ExposedJsApi {
  // 将jsToNative的方法调用委托给bridge实例
  private final CordovaBridge bridge;

  SystemExposedJsApi(CordovaBridge bridge) {
      this.bridge = bridge;
  }

  // JS调用原生插件时使用
  @JavascriptInterface
  public String exec(int bridgeSecret, String service, String action, String callbackId, String arguments) throws JSONException, IllegalAccessException {
    // 委托给bridge实例进行处理
    return bridge.jsExec(bridgeSecret, service, action, callbackId, arguments);
  }

  // JS中设置NativeToJS的通讯方式
  @JavascriptInterface
  public void setNativeToJsBridgeMode(int bridgeSecret, int value) throws IllegalAccessException {
    bridge.jsSetNativeToJsBridgeMode(bridgeSecret, value);
  }

  // ？？
  @JavascriptInterface
  public String retrieveJsMessages(int bridgeSecret, boolean fromOnlineEvent) throws IllegalAccessException {
    return bridge.jsRetrieveJsMessages(bridgeSecret, fromOnlineEvent);
  }
}
```

CordovaBridge类，js对Native的调用，不论是基于window._cordovaNative还是基于prompt的调用，都会经由CordovaBridge实例处理。其中的jsExec方法非常重要，就是在这个方法中实现了对cordova插件的调用，该方法内涉及到的pluginManager.exec()方法，已经在pluginManager类中说明
```java
public class CordovaBridge {
    private static final String LOG_TAG = "CordovaBridge";
    // 插件管理器
    private PluginManager pluginManager;
    // 消息队列
    private NativeToJsMessageQueue jsMessageQueue;
    // 通讯凭证，由UI线程产生，js线程读取
    // 在prompt base的交互过程中，会产生一个随机数作为Secret，但是jsinterface base的交互中貌似这个值始终是-1
    private volatile int expectedBridgeSecret = -1; // written by UI thread, read by JS thread.

    public CordovaBridge(PluginManager pluginManager, NativeToJsMessageQueue jsMessageQueue) {
        this.pluginManager = pluginManager;
        this.jsMessageQueue = jsMessageQueue;
    }
    // 执行window._nativeCordova.exec方法后，进入到该方法
    // 执行window.prompt后，也进入到该方法
    // 该方法是执行Cordova插件的统一入口
    public String jsExec(int bridgeSecret, String service, String action, String callbackId, String arguments) throws JSONException, IllegalAccessException {
        if (!verifySecret("exec()", bridgeSecret)) {
            return null;
        }
        // If the arguments weren't received, send a message back to JS.  It will switch bridge modes and try again.  See CB-2666.
        // We send a message meant specifically for this case.  It starts with "@" so no other message can be encoded into the same string.
        if (arguments == null) {
            return "@Null arguments.";
        }
        // 调用native插件期间暂停nativeToJS的消息传递
        jsMessageQueue.setPaused(true);
        try {
            // Tell the resourceApi what thread the JS is running on.
            CordovaResourceApi.jsThread = Thread.currentThread();
            // 利用pluginManager获取指定插件名插件，然后执行
            pluginManager.exec(service, action, callbackId, arguments);
            String ret = null;
            if (!NativeToJsMessageQueue.DISABLE_EXEC_CHAINING) {
                ret = jsMessageQueue.popAndEncode(false);
            }
            return ret;
        } catch (Throwable e) {
            e.printStackTrace();
            return "";
        } finally {
            jsMessageQueue.setPaused(false);
        }
    }

    // js调用设置nativeTojs的mode
    public void jsSetNativeToJsBridgeMode(int bridgeSecret, int value) throws IllegalAccessException {
        if (!verifySecret("setNativeToJsBridgeMode()", bridgeSecret)) {
            return;
        }
        jsMessageQueue.setBridgeMode(value);
    }

    public String jsRetrieveJsMessages(int bridgeSecret, boolean fromOnlineEvent) throws IllegalAccessException {
        if (!verifySecret("retrieveJsMessages()", bridgeSecret)) {
            return null;
        }
        return jsMessageQueue.popAndEncode(fromOnlineEvent);
    }

    private boolean verifySecret(String action, int bridgeSecret) throws IllegalAccessException {
        if (!jsMessageQueue.isBridgeEnabled()) {
            if (bridgeSecret == -1) {
                LOG.d(LOG_TAG, action + " call made before bridge was enabled.");
            } else {
                LOG.d(LOG_TAG, "Ignoring " + action + " from previous page load.");
            }
            return false;
        }
        // Bridge secret wrong and bridge not due to it being from the previous page.
        if (expectedBridgeSecret < 0 || bridgeSecret != expectedBridgeSecret) {
            LOG.e(LOG_TAG, "Bridge access attempt with wrong secret token, possibly from malicious code. Disabling exec() bridge!");
            clearBridgeSecret();
            throw new IllegalAccessException();
        }
        return true;
    }

    /** Called on page transitions */
    void clearBridgeSecret() {
        expectedBridgeSecret = -1;
    }

    public boolean isSecretEstablished() {
        return expectedBridgeSecret != -1;
    }

    /** Called by cordova.js to initialize the bridge. */
    //On old Androids SecureRandom isn't really secure, this is the least of your problems if
    //you're running Android 4.3 and below in 2017
    @SuppressLint("TrulyRandom")
    int generateBridgeSecret() {
        SecureRandom randGen = new SecureRandom();
        expectedBridgeSecret = randGen.nextInt(Integer.MAX_VALUE);
        return expectedBridgeSecret;
    }

    public void reset() {
        jsMessageQueue.reset();
        clearBridgeSecret();
    }

    // 处理prompt base的交互，当js 使用prompt调用native时，由这个方法来解析js传来的value前缀，进而执行相应的动作
    public String promptOnJsPrompt(String origin, String message, String defaultValue) {
        if (defaultValue != null && defaultValue.length() > 3 && defaultValue.startsWith("gap:")) {
            // 以gap:开头则尝试调用Cordova 插件
            JSONArray array;
            try {
                array = new JSONArray(defaultValue.substring(4));
                int bridgeSecret = array.getInt(0);
                String service = array.getString(1);
                String action = array.getString(2);
                String callbackId = array.getString(3);
                // 调用插件
                String r = jsExec(bridgeSecret, service, action, callbackId, message);
                return r == null ? "" : r;
            } catch (JSONException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
            return "";
        }
        // Sets the native->JS bridge mode.
        else if (defaultValue != null && defaultValue.startsWith("gap_bridge_mode:")) {
            // gap_bridge_mode:开头则尝试设置NativeToJs的BridgeMode
            try {
                int bridgeSecret = Integer.parseInt(defaultValue.substring(16));
                // 设置NativeToJs mode
                jsSetNativeToJsBridgeMode(bridgeSecret, Integer.parseInt(message));
            } catch (NumberFormatException e){
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
            return "";
        }
        // Polling for JavaScript messages
        else if (defaultValue != null && defaultValue.startsWith("gap_poll:")) {
            int bridgeSecret = Integer.parseInt(defaultValue.substring(9));
            try {
                String r = jsRetrieveJsMessages(bridgeSecret, "1".equals(message));
                return r == null ? "" : r;
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
            return "";
        }
        else if (defaultValue != null && defaultValue.startsWith("gap_init:")) {
            // Protect against random iframes being able to talk through the bridge.
            // Trust only pages which the app would have been allowed to navigate to anyway.
            // cordova.js在启动时使用window.prompt方法，传入gap_init:3 来设置jsMessageQueue的mode为EvalBridgeMode
            if (pluginManager.shouldAllowBridgeAccess(origin)) {
                // Enable the bridge
                int bridgeMode = Integer.parseInt(defaultValue.substring(9));
                jsMessageQueue.setBridgeMode(bridgeMode);
                // Tell JS the bridge secret.
                int secret = generateBridgeSecret();
                return ""+secret;
            } else {
                LOG.e(LOG_TAG, "gap_init called from restricted origin: " + origin);
            }
            return "";
        }
        return null;
    }
}
```

PluginResult类：每一个PluginResult实例都代表一个插件运行结果，插件执行业务完毕后需要构建PluginResult类实例来存储运行结果和状态。    
其内部有三个非常重要的属性：
1. status：代表插件运行结果的状态码，cordova.js中根据这个状态码判断插件是否调用成功
2. keepCallback：是否需要保持当前callbackID，如果为false，cordova.js中回调完函数会将该callbackID删除
3. encodedMessage：需要传递给js的插件运行结果，最常见是json格式字符串
```java
// 代表插件运行结果
public class PluginResult {
  // 插件结果状态码
  private final int status;
  // 回传给js的结果的类型
  private final int messageType;
  // 是否保持callbackID，如果为false，则js中回调完这次callbackID就会删掉
  private boolean keepCallback = false;
  // 需要回传给js的结果
  private String encodedMessage;

  // 创建实例的时候传入结果状态和json格式结果
  public PluginResult(Status status, JSONObject message) {
    this.status = status.ordinal();
    this.messageType = MESSAGE_TYPE_JSON;
    encodedMessage = message.toString();
  }

  // 插件结果状态枚举类，其中NO_RESULT和OK两种状态在js中会被认为是调用插件成功
  // cordova.js中有对应的状态定义
  public enum Status {
      NO_RESULT, // NOTE: 结果为NO_RESULT时，cordova.js中会直接删除callbackid对应的回调函数
      OK,
      CLASS_NOT_FOUND_EXCEPTION,
      ILLEGAL_ACCESS_EXCEPTION,
      INSTANTIATION_EXCEPTION,
      MALFORMED_URL_EXCEPTION,
      IO_EXCEPTION,
      INVALID_ACTION,
      JSON_EXCEPTION,
      ERROR
  }
}
```
CallbackContext类的每一个实例代表着一个Native对Js的callback的引用。该类主要有三个作用：
1. 负责将PluginResult实例传递给webView，每发送一次会在js一侧引发一次对应callbackId的回调
2. 提供快捷方法来快速创建指定状态的PluginResult实例，并将实例发送给webView。
3. 在内部保存callbackId

```java
// 该类负责将PluginResult实例传递给webView
public class CallbackContext {
  private static final String LOG_TAG = "CordovaPlugin";

  // 回调id
  private String callbackId;
  private CordovaWebView webView;
  // 该回调id是否已经finish
  protected boolean finished;

  // 构建实例的时候传入callbackId和webview
  public CallbackContext(String callbackId, CordovaWebView webView) {
    this.callbackId = callbackId;
    this.webView = webView;
  }

   // 插件运行完毕，设置结果，同时将结果实例传入webview
  public void sendPluginResult(PluginResult pluginResult) {
    synchronized (this) {
      if (finished) {
        // 如果该callbackContent已经finished，则不会在js侧触发回调
        LOG.w(LOG_TAG, "Attempted to send a second callback for ID: " + callbackId + "\nResult was: " + pluginResult.getMessage());
        return;
      } else {
        // 如果keepCallback为true，则finished为false。finished是对pluginResult实例内部的keepcallback取反
        finished = !pluginResult.getKeepCallback();
      }
    }
    // 调用webview的方法，传入插件结果实例，webView会直接将pluginResult和callbackId交给nativeToJsMessageQueue实例进行处理
    webView.sendPluginResult(pluginResult, callbackId);
  }

  // 快速创建一个OK状态的结果，然后发送结果
  public void success(JSONObject message) {
    sendPluginResult(new PluginResult(PluginResult.Status.OK, message));
  }
}
```

NativeToJsMessageQueue类维护着NativeToJs的消息队列，并且负责执行这些消息。它承担了三个重要作用：
1. 内部维护了一个Native发向js的消息队列LinkedList<JsMessage>
2. 内部定义了抽象类BridgeMode和四个实现子类。BridgeMode代表着Native调用js的模式，其四个实现类分别实现了四种不同的方式来进行NativeToJs的调用
3. 内部提供了一些拼接**js方法调用字符串**的辅助方法，如：popAndEncodeAsJs

```java
/**
 * Holds the list of messages to be sent to the WebView.
 * 存储Native到js的消息队列，并且提供一系列方法来对传递给JS的消息进行加工
 * 其内部定义了BridgeMode的抽象类及其子类和JsMessage类
 */
public class NativeToJsMessageQueue {
  // 当前队列是否暂停运行
  private boolean paused;
  // Native to js消息队列
  private final LinkedList<JsMessage> queue = new LinkedList<JsMessage>();
  // nativeTojs的模式的数组。启动后注册的顺序为NoOpBridgeMode， LoadUrlBridgeMode，OnlineEventsBridgeMode，EvalBridgeMode
  private ArrayList<BridgeMode> bridgeModes = new ArrayList<BridgeMode>();
  // 当前使用的nativeToJs模式
  private BridgeMode activeBridgeMode;
  // 添加一个NativeToJs模式
  public void addBridgeMode(BridgeMode bridgeMode) {
    bridgeModes.add(bridgeMode);
  }
  // 设置当前使用mode，该方法只被CordovaBridge类调用，因此只能由js侧来调用，所以js侧决定了mode应该选哪个
  // cordova.js运行后，会使用prompt的方式通知Native，调用该方法，将bridgeMode设置为EvalBridgeMode
  public void setBridgeMode(int value) {
      if (value < -1 || value >= bridgeModes.size()) {
          LOG.d(LOG_TAG, "Invalid NativeToJsBridgeMode: " + value);
      } else {
          BridgeMode newMode = value < 0 ? null : bridgeModes.get(value);
          if (newMode != activeBridgeMode) {
              LOG.d(LOG_TAG, "Set native->JS mode to " + (newMode == null ? "null" : newMode.getClass().getSimpleName()));
              synchronized (this) {
                  activeBridgeMode = newMode;
                  if (newMode != null) {
                      newMode.reset();
                      if (!paused && !queue.isEmpty()) {
                          // 如果当前队列不为空，则马上进行处理
                          newMode.onNativeToJsMessageAvailable(this);
                      }
                  }
              }
          }
      }
  }
  /**
    * Same as popAndEncode(), except encodes in a form that can be executed as JS.
    * 根据队列内的jsMessage实例拼接并返回出相关的js方法调用字符串
    * 拼接JS方法调用字符串的时候又借助了JsMessage类的encodeAsJsMessage方法
    */
  public String popAndEncodeAsJs() {
      synchronized (this) {
          int length = queue.size();
          if (length == 0) {
              return null;
          }
          // 总的回传js的消息长度
          int totalPayloadLen = 0;
          // 需要回传给js的消息个数
          int numMessagesToSend = 0;
          // 遍历队列内的jsMessage
          for (JsMessage message : queue) {
              // 计算消息长度
              int messageSize = message.calculateEncodedLength() + 50; // overestimate.
              // 查看回传js的消息长度是否超过限制
              if (numMessagesToSend > 0 && totalPayloadLen + messageSize > MAX_PAYLOAD_SIZE && MAX_PAYLOAD_SIZE > 0) {
                  break;
              }
              totalPayloadLen += messageSize;
              numMessagesToSend += 1;
          }
          // 是否一次性发送队列内所有的消息，如果为false，则说明队列内还有剩余消息需要发送到js
          boolean willSendAllMessages = numMessagesToSend == queue.size();
          StringBuilder sb = new StringBuilder(totalPayloadLen + (willSendAllMessages ? 0 : 100));
          // Wrap each statement in a try/finally so that if one throws it does
          // not affect the next.
          for (int i = 0; i < numMessagesToSend; ++i) {
              JsMessage message = queue.removeFirst();
              if (willSendAllMessages && (i + 1 == numMessagesToSend)) {
                  message.encodeAsJsMessage(sb);
              } else {
                  sb.append("try{");
                  // 拼接出来的js调用：cordova.callbackFromNative(callbackId, isSuccess, status, [ encodeData ], keepCallback); 根据插件结果的类型不同，有些差异，详见encodeAsJsMessage方法内解析
                  message.encodeAsJsMessage(sb);
                  sb.append("}finally{");
              }
          }
          // 以上如果有三个jsMessage，则拼接出的js如下
          /**
            * try {
            *   cordova.callbackFromNative(callbackId, isSuccess, status, [ encodeData ], keepCallback);
            *  } finally {
            *   try {
            *      cordova.callbackFromNative(callbackId, isSuccess, status, [ encodeData ], keepCallback);
            *    } finally {
            *      cordova.callbackFromNative(callbackId, isSuccess, status, [ encodeData ], keepCallback);
            */
          if (!willSendAllMessages) {
              // 如果不是一次发送全部消息，则添加以下语句，让js在下一个event loop对队列中剩余的消息进行查询
              // NOTE: cordova/plugin/android/polling这个模块在cordova.js中并没有定义，非常奇怪，如果运行了这一句，js侧会报错
              sb.append("window.setTimeout(function(){cordova.require('cordova/plugin/android/polling').pollOnce();},0);");
          }
          // 补齐之前缺失的 }
          for (int i = willSendAllMessages ? 1 : 0; i < numMessagesToSend; ++i) {
              sb.append('}');
          }
          String ret = sb.toString();
          return ret;
      }
  }
   /**
     * Add a JavaScript statement to the list.
     * 添加一个插件运行结果，在CordovaWebViewImpl类中被调用
     */
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
}
```
遗留问题：NativeToJsMessageQueue类的popAndEncodeAsJs方法拼接出来的js字符串中，有可能会出现 'cordova.require('cordova/plugin/android/polling').pollOnce()' 这句js代码的调用，但是cordova.js中并没有注册这个module，一旦执行这句话就会报错。


BridgeMode类是一个抽象类，代表Native调用Js的方式。它定义了一个抽象方法onNativeToJsMessageAvailable，这个方法代表Native实际上如何调用Js，不同的子类会有不同的实现机制。
```java
// BridgeMode抽象类，代表native如何调用js
public static abstract class BridgeMode {
    // Native调用js的抽象方法
    public abstract void onNativeToJsMessageAvailable(NativeToJsMessageQueue queue);
    public void notifyOfFlush(NativeToJsMessageQueue queue, boolean fromOnlineEvent) {}
    public void reset() {}
}
```

BridgeMode类有四个实现类，它们使用了不同的机制来进行NativeToJs的调用，这些差异都被onNativeToJsMessageAvailable方法统一了对外接口。
1. NoOpBridgeMode类：Native端不做操作，依靠JS侧使用定时器进行轮询获取消息
2. LoadUrlBridgeMode类：先根据JsMessage实例拼接出JS代码字符串，大概类似于'**cordova.callbackFromNative(callbackId, isSuccess, status, [ encodeData ], keepCallback)**'。然后使用webView.loadUrl('javascript:')来调用js方法
3. OnlineEventsBridgeMode类：通过online/offline事件来告诉JS什么时候去查询信息
4. EvalBridgeMode类：先根据JsMessage实例拼接出JS代码字符串（跟LoadUrlBridgeMode一样），然后使用webView.evaluateJavascript方法来最终执行js

其中LoadUrlBridgeMode类和EvalBridgeMode类都是借助NativeToJsMessageQueue类的popAndEncodeAsJs方法来进行js代码字符串拼接的

```java
/** Uses JS polls for messages on a timer..  */
// 在js一侧使用定时器轮询来获取消息，Native端不做操作
public static class NoOpBridgeMode extends BridgeMode {
    @Override public void onNativeToJsMessageAvailable(NativeToJsMessageQueue queue) {
    }
}
// 使用webView.loadUrl方法来调用JS
/** Uses webView.loadUrl("javascript:") to execute messages. */
public static class LoadUrlBridgeMode extends BridgeMode {
    private final CordovaWebViewEngine engine;
    private final CordovaInterface cordova;

    public LoadUrlBridgeMode(CordovaWebViewEngine engine, CordovaInterface cordova) {
        this.engine = engine;
        this.cordova = cordova;
    }

    // 该方法负责调用js方法
    @Override
    public void onNativeToJsMessageAvailable(final NativeToJsMessageQueue queue) {
        cordova.getActivity().runOnUiThread(new Runnable() {
            public void run() {
                // NOTE: 借助queue的popAndEncodeAsJs方法拼接出js代码字符串
                String js = queue.popAndEncodeAsJs();
                if (js != null) {
                    // 通过engine实例来执行拼接出的js语句
                    engine.loadUrl("javascript:" + js, false);
                }
            }
        });
    }
}
/** Uses online/offline events to tell the JS when to poll for messages. */
// 通过online/offline事件来告诉JS什么时候去查询信息
public static class OnlineEventsBridgeMode extends BridgeMode {
    private final OnlineEventsBridgeModeDelegate delegate;
    private boolean online;
    private boolean ignoreNextFlush;

    public interface OnlineEventsBridgeModeDelegate {
        void setNetworkAvailable(boolean value);
        void runOnUiThread(Runnable r);
    }

    public OnlineEventsBridgeMode(OnlineEventsBridgeModeDelegate delegate) {
        this.delegate = delegate;
    }

    @Override
    public void reset() {
        delegate.runOnUiThread(new Runnable() {
            public void run() {
                online = false;
                // If the following call triggers a notifyOfFlush, then ignore it.
                ignoreNextFlush = true;
                delegate.setNetworkAvailable(true);
            }
        });
    }

    @Override
    public void onNativeToJsMessageAvailable(final NativeToJsMessageQueue queue) {
        delegate.runOnUiThread(new Runnable() {
            public void run() {
                if (!queue.isEmpty()) {
                    ignoreNextFlush = false;
                    delegate.setNetworkAvailable(online);
                }
            }
        });
    }
    // Track when online/offline events are fired so that we don't fire excess events.
    @Override
    public void notifyOfFlush(final NativeToJsMessageQueue queue, boolean fromOnlineEvent) {
        if (fromOnlineEvent && !ignoreNextFlush) {
            online = !online;
        }
    }
}
/** Uses webView.evaluateJavascript to execute messages. */
// 通过webView.evaluateJavascript方法来最终执行js
public static class EvalBridgeMode extends BridgeMode {
    private final CordovaWebViewEngine engine;
    private final CordovaInterface cordova;

    public EvalBridgeMode(CordovaWebViewEngine engine, CordovaInterface cordova) {
        this.engine = engine;
        this.cordova = cordova;
    }

    @Override
    public void onNativeToJsMessageAvailable(final NativeToJsMessageQueue queue) {
        cordova.getActivity().runOnUiThread(new Runnable() {
            public void run() {
                // NOTE: 借助queue的popAndEncodeAsJs方法拼接出js代码字符串
                String js = queue.popAndEncodeAsJs();
                if (js != null) {
                    engine.evaluateJavascript(js, null);
                }
            }
        });
    }
}
```

JsMessage类：这个类代表着Native发送给Js的消息，主要功能有两个：
1. 提供函数在EvalBridgeMode和LoadUrlBridgeMode下将PluginResult实例转换成js代码字符串
2. 提供一些辅助函数来获取js消息的相关信息

```java
private static class JsMessage {
    // 回调id
    final String jsPayloadOrCallbackId;
    // 存储的插件运行结果
    final PluginResult pluginResult;

    JsMessage(PluginResult pluginResult, String callbackId) {
        if (callbackId == null || pluginResult == null) {
            throw new NullPointerException();
        }
        jsPayloadOrCallbackId = callbackId;
        this.pluginResult = pluginResult;
    }

    // 拼接js方法调用字符串，后续会通过webview.loadUrl('javascript:' ...)或者webview.evelJavaScript的方式来调用js方法
    void encodeAsJsMessage(StringBuilder sb) {
        if (pluginResult == null) {
            sb.append(jsPayloadOrCallbackId);
        } else {
            int status = pluginResult.getStatus();
            boolean success = (status == PluginResult.Status.OK.ordinal()) || (status == PluginResult.Status.NO_RESULT.ordinal());
            // 注意，调用的是js方法是cordova.callbackFromNative(callbackId, isSuccess, status, [ params ], keepCallback);
            sb.append("cordova.callbackFromNative('")
                    .append(jsPayloadOrCallbackId)
                    .append("',")
                    .append(success)
                    .append(",")
                    .append(status)
                    .append(",[");
            buildJsMessage(sb);
            sb.append("],")
                    .append(pluginResult.getKeepCallback())
                    .append(");");
        }
    }
}
```