---
title: vscode debugging
date: 2018-06-20 18:34:34
categories: 
  - 翻译系列
tags: 
  - vscode
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

vscode debugger支持两大类的调试模式：
* launch： debugger以debug模式直接加载(launch)运行一个程序，然后进行debug。
* attah：事先以debug模式运行被调试程序，然后debugger通过被调试程序开放的一个调试端口连接到被调试程序进行debug。远程调试只能通过这种方式，因为远程调试debugger无法直接加载运行被调试程序。

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
vscode内置的nodejs debugger 与 nodejs runtime之间通过 [wire protocols](https://en.wikipedia.org/wiki/Wire_protocol)协议连接，具体使用哪种协议则由runtime支持哪种协议决定。当前的wire protocols存在两种类型的协议：

>* legacy: 旧版本的nodejs(node version < 6.3)支持的协议
>* inspector: 新版本nodejs开始支持的协议(node version >= 6.3)    

以下是其他runtime对两种协议的支持情况：
<img src="/img/rumtime.png">

>下面是一些重要的launch configuration选项：
> * protocol - "auto" or "inspector" or "legacy"; auto是默认值，vscode会尝试自动判断使用哪种协议;"inspector"强制使用新版的inspector协议，在确认runtime支持的时候可以使用；"legacy"强制使用旧版的协议，在确认runtime支持的时候可以使用。（应该尽量使用新版协议）
>* port - 9229(示例：9229端口); 使用attach模式或者进行远程调试时的调试端口，如果使用的runtime使用的协议是inspector，则默认端口是9229，如果使用的协议是legacy，则默认端口是5858。
>* address - 127.0.0.1(示例：本机ip); 使用attach模式或者进行远程调试时被调试的runtime的IP地址
>* sourceMaps - true(default) or false，是否开启scourceMaps
>* outFiles - ["path to generated JavaScript files"]; 数组对象，每个元素都是一个路径。路径是编译出的js文件的所在路径，当使用例如ts这种可以编译为js的语言时通常需要配置这一项。如果编译出的js文件不在源文件旁边而是在某一个单独的目录，则需要使用这个选项来帮助vscode debugger找到编译后js文件。
>* restart - true or false。whether the Node.js debugger automatically restarts after the debug session has ended(当调试会话结束时，debugger是否自动重启。其实就是当被调试程序重启时，debugger也自动重启，重新进行调试)。这个选项配合上nodemon这种可以自动检测文件变化，然后自动重启应用的工具来进行调试时，非常好用。当被调试的应用重启时，debug会话就会结束，如果restart设置为true，debugger此时会自动重启。
>* timeout - 10000 （示例：10s）;连接一个debug session 的timeout时间
>* stopOnEntry - true or false；是否在进入调试程序后立刻断点暂停。
>* localRoot - "path to root dir"; 指明vscode项目的根目录，在[**远程调试**](https://code.visualstudio.com/docs/nodejs/nodejs-debugging#_remote-debugging)的时候有用。
>* remoteRoot - "Node's root directory"; 在[**远程调试**](https://code.visualstudio.com/docs/nodejs/nodejs-debugging#_remote-debugging)的时候有用。

> 以下配置只适合 debugger以 launch模式进行调试时使用：
>* program - "path to progrom to debug"; 指明需要加载进行调试的node应用路径。
>* args - ["arg1", "arg2", ... ]; 需要传递给program 的参数。
>* cwd - "path";  launch the program to debug in this directory。不知道跟program有什么区别。
>* [runtimeExecutable](https://code.visualstudio.com/docs/nodejs/nodejs-debugging#_launch-configuration-support-for-npm-and-other-tools) - 'node'(default) or 'npm' or 'mocha' or 'gulp' ...; 如果不想使用默认的node runtime直接launch被debug的程序，则可以使用runtimeExecutable选项。任何在path环境变量中可以获取的program（例如npm、mocha等等）都可以被设置。使用runtimeArgs选项可以传递参数。如果runtimeExecutable指定了需要加载的程序，比如npm run 中通常已经指定了需要加载哪些js文件，此时可以不需要指定program选项。
>* runtimeArgs - ['arg1', 'arg2'，... ]，需要传递给runtimeExecutable的参数。
>* env - { key: "value", ... }; 为程序设置环境变量
>* [**console**](https://code.visualstudio.com/docs/nodejs/nodejs-debugging#_node-console) - "internalConsole"(default) or "integratedTerminal" or "externalTerminal"; 定义调试代码的输入输出终端。internalConsole是VS Code Debugger内置的终端，但是该终端不支持输入交互，所以当调试一些需要输入的程序时通常需要切换到其他终端。integratedTerminal是vs code编辑器的集成终端，可以支持输入交互。externalTerminal是启用外部终端，具体配置这里不是很清楚。


### Remote debugging
node version>= 4.x时支持远程调试。默认情况下，vscode会获取远程node上的源码文件并在vscode上以只读的方式打开（具体原理不明确）。如果想要修改代码则需要配置 localRoot 和 remoteRoot 这两个选项。remoteRoot 用来指明远程node的项目根目录，localRoot用来指明获取的源码文件放在本地的哪个目录作为根目录。

# 各种类型的debug的官方demo
官方有一个展示不同类型和场景的调试demo仓库，非常具有参考价值，强烈建议进入围观。[**点击此处进入**](https://github.com/Microsoft/vscode-recipes)