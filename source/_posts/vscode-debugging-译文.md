---
title: vscode debugging 译文
date: 2018-06-20 18:34:34
categories: 
  - 翻译系列
tags: 
  - vscode
  - 译文
comments: false
---
> 今天看到vscode调试相关的教程，发现vscode的调试功能比我想象的要强大的多。觉得有必要好好阅读一下vscode的debug文档。以下是[**vscode debug的官方文档**](https://code.visualstudio.com/docs/editor/debugging)的非完全译文，主要包括了launch.json文件选项的一些属性翻译

# Debugging
vscode一个关键特性就是它强大的调试能力，vscode内置了强大的debugger。

<img src="https://code.visualstudio.com/assets/docs/editor/debugging/debugging_hero.png">

## Debugger extensions（debugger拓展）
vscode内置的debugger原生支持nodejs runtime，可以支持调试js，ts和其他一些能转译成js的语言。

如果需要调试其他语言（例如：php, Ruby， Go， C#， python等），则需要安装相应的 debugger extensions。下面是一些流行的debugger extensions。

>[**python调试**](https://marketplace.visualstudio.com/items?itemName=ms-python.python)   
[**c/c++调试**](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools)    
[**Debugger for Chrome**](https://marketplace.visualstudio.com/items?itemName=msjsdiag.debugger-for-chrome)    
[**C#**](https://marketplace.visualstudio.com/items?itemName=ms-vscode.csharp)    

## Launch configurations
最简单的调试：**按下F5**，vscode会进入调试模式，并且尝试调试你当前的文件。

为了更加精确的调试，通常需要配置一个**launch.json**文件，该文件通常位于项目根目录的.vscode文件夹下。

vscode debugger通常支持以debug模式加载(launch)一个程序或者依附(attah)在一个正在以debug模式运行的程序上面这两种调试方式。这两种调试模式是vscode调试的基础。

vscode调试最重要的一步莫过于配置launch.json文件，以下是关于这个文件一些关键配置项的解析：
> launch configuration的必要字段：
>* type - 使用哪种类型的debugger进行debug，可以使内置的"node"，或者是安装其他调试插件后提供的debugger，比如安装debugger for chrome插件后可以使用"chrome"。
>* request - "launch" or "attach"，使用launch，vscode会尝试以debug模式加载运行程序；使用attach，vscode则会依附在一个已经运行的程序上进行调试。
>* name - 当前调试配置的别名

> launch configuration的非必要字段
>* preLaunchTask - 在启动调试之前，运行一个task。task的具体配置定义在[**.vscode/tasks.json**](https://code.visualstudio.com/docs/editor/tasks)文件夹内。
>* postDebugTask - 在调试结束的时候，运行一个task。该task同样定义在上述的tasks.json文件中
>* program - 调试启动时需要需要运行的文件。
>* args - 需要传递给program的参数
>* env - 设置环境变量
>* cwd - 设置当前工作目录
>* port - 当以依附模式(attach)运行时，被调试的程序的调试监听端口。
>* stopOnEntry - true or false。当程序加载完时是否需要立刻断点停止

>一些常用的变量：    
> launch.json里面允许使用一些常用的模板变量：   
> * ${workspaceFolder} 代表当前工作文件夹
> *  ${file}代表当前编辑器正打开激活的文件
> * ${env:Name}代表环境变量名
```
{
    "type": "node",
    "request": "launch",
    "name": "Launch Program",
    "program": "${workspaceFolder}/app.js",
    "cwd": "${workspaceFolder}",
    "args": [ "${env:USERNAME}" ]
}
```

## debug in nodejs
以下文档专门介绍关于[**nodejs的调试**](https://code.visualstudio.com/docs/nodejs/nodejs-debugging)

此外github上面的一些例子值得参考：
* [**在node中调试typescript**](https://github.com/jagreehal/node-typescript-debug-example.git)
* [**在node中调试es6**](https://github.com/katopz/vscode-debug-nodejs-es6.git)
* [**vscode调试不完全指南**](http://jerryzou.com/posts/vscode-debug-guide/)

### Supported Node-like runtimes
nodejs debugger 与 nodejs runtime之间通过 [wire protocols](https://en.wikipedia.org/wiki/Wire_protocol)协议连接，具体使用哪种协议则由runtime支持哪种协议决定。当前的wire protocols存在两种类型的协议：

>* legacy: 旧版本的nodejs(node version < 6.3)支持的协议
>* inspector: 新版本nodejs开始支持的协议(node version >= 6.3)    

以下是其他runtime对两种协议的支持情况：
<img src="/img/rumtime.png">

未完待续...