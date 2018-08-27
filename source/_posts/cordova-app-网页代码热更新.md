---
title: cordova app 网页代码热更新
date: 2018-07-19 19:05:18
categories:
  - cordova系列
tags: 
  - 前端
  - cordova
  - 移动端
comments: false
---
<!-- post5 -->
cordova app由h5提供界面功能，在实现网页代码热更新后，开发者就可以随时发布新版app页面，而用户无需重新安装app^^。以下教程基于cordova andorid平台。
cordova 网页代码热更新的实现主要依赖以下几个插件：   
1. [**cordova-hot-code-push**](https://github.com/nordnet/cordova-hot-code-push) 代码热更新主要由此插件完成
2. [**cordova-hot-code-push-local-dev-addon**](https://github.com/nordnet/cordova-hot-code-push-local-dev-addon) 本地部署测试时需要安装该插件，正式发布时最好将此插件移除
3. [**cordova-hot-code-push-cli**](https://github.com/nordnet/cordova-hot-code-push-cli) 命令行工具，用来生成代码热更新必须的相关文件（网页文件清单文件以及热更新配置文件）

首先介绍代码热更新里面的两个重要配置文件：   
1. chcp.manifest：该文件是需要部署的网页文件的清单，包含了需要部署的网页文件的相对路径以及hash码。[**官方文档点此**](https://github.com/nordnet/cordova-hot-code-push/wiki/Content-manifest)，文件截图如下：
> <img src="/img/post/post5/chcp.manifest.png" alt="chcp.manifest截图">
2. chcp.json：该文件包含了代码热更新服务的一些基础配置，其中的配置项意义可参考截图注释。[**官方文档点此**](https://github.com/nordnet/cordova-hot-code-push/wiki/Application-config)，截图如下：
> <img src="/img/post/post5/chcp.json.png" alt="chcp.json截图">

除了上述两个重要配置文件，在cordova项目中的配置文件config.xml上（如果不知道这个文件，需要补习一下cordova的文档）需要配置chcp标签，[**官方文档点此**](https://github.com/nordnet/cordova-hot-code-push/wiki/Cordova-config-preferences)，截图如下：
> <img src="/img/post/post5/config.xml.png" alt="config.xml.json截图">

cordova-hot-code-push插件实现网页文件热更新的原理和过程如下，[**官方文档点此**](https://github.com/nordnet/cordova-hot-code-push/wiki/Update-workflow)：
> <img src="/img/post/post5/update-workflow.png" alt="更新流程图">
>
> 1. 用户打开app
> 2. cordova加载cordova-hot-code-push插件，该插件启动upload-loader
> 3. upload-loader从config.xml中的chcp标签的config-file配置中获取到服务器中chcp.json的url，并从这个url下载到最新的chcp.json文件，然后对比app本地的chcp.json文件和最新的chcp.json文件中的release项的value。如果不同，则认为服务器中发布了新的网页文件，进入下一步
> 4. upload-loader比较服务器中chcp.json文件中的[**min_native_interface**](https://github.com/nordnet/cordova-hot-code-push/wiki/Application-config)项的value与本地config.xml文件中chcp标签的native-interface的value。如果native-interface >= min_native_interface 则进入下一步。（min_native_interface的意思是允许本地版本最低是多少的客户端进行网页热更新，低于这个版本，只能更新apk）
> 5. upload-loader从新的chcp.json文件中获取content_url 的value，然后从这个url下载chcp.manifest文件。通过对比新的chcp.manifest文件与本地旧的chcp.manifest文件（比较文件名与hash码），得出需要更新那些网页文件。
> 6. upload-loader将需要更新的网页文件从content_url下载下来
> 7. upload-loader在应用内发送一个notification，提醒插件更新准备完毕
> 8. 插件根据chcp.json文件中的update项设置的时机进行网页文件更新。

在整个流程中，可见chcp.json与chcp.manifest文件的重要性。这两个文件的生成，特别是chcp.manifest文件中还需要每个文件的hash码，当然不可能人工手写。而是要借助cordova-hot-code-push-cli命令行工具。其安装和几个关键的命令如下，[**官方文档点此**](https://github.com/nordnet/cordova-hot-code-push-cli)：
* 安装
```
npm install -g cordova-hot-code-push-cli
```
* 命令行简介：
```bash
cordova-hcp init 
# 运行后按照提示输入一些项目关键信息，比如app update的时机，项目名称等，其中有关于Amazon服务器配置，如果不是使用Amazon服务器部署的话，可以跳过不填。输入关键信息后，会生成一个cordova-hcp.json文件

cordova-hcp build [www]
# 可携带一个文件夹的路径参数，用来指定哪个是需要部署的网页文件夹，默认是当前目录下的www目录。该命令扫描指定的网页文件夹内的网页文件，产生chcp.manifest清单文件。也根据cordova-hcp.json文件生成chcp.json文件。

cordova-hcp server
# 启动本地部署服务器，在做本地部署测试时可以使用到。

cordova-hcp login
# 使用Amazon服务器部署的时候，使用这个命令生成登录凭证。不使用Amazon服务器部署，可以不使用这个命令

cordova-hcp deploy
# 使用Amazon服务器部署的时候，可以使用这个命令将网页文件部署到Amazon服务器。不使用Amazon服务器部署，可以不使用这个命令
```


知道大概的原理后，下面是官方提供的本地部署网页代码热更新体验流程，[**官方文档点此**](https://github.com/nordnet/cordova-hot-code-push#quick-start-guide)：
>1. 创建一个新的cordova project，添加ios or android 平台，之后在cordova项目的根目录里面运行以下流程
```
cordova create TestProject com.example.testproject TestProject
cd ./TestProject
cordova platform add android
cordova platform add ios
```
>2. 安装cordova-hot-code-push插件
```
cordova plugin add cordova-hot-code-push-plugin
```
>3. 添加cordova-hot-code-push-local-dev-addon插件，本地部署测试需要使用
```
cordova plugin add cordova-hot-code-push-local-dev-addon
```
>4. 安装cordova-hot-code-push-cli 命令行工具：
```
npm install -g cordova-hot-code-push-cli
```
>5. 打开本地服务器。实际上就是开了一个本地服务器，然后使用ngrok反向代理在公网暴露本地服务器，充当网页热更新的部署服务器。对外暴露的网页文件夹是cordova项目根目录的www文件夹。此时已经自动产生chcp.manifest和chcp.json文件放在www文件夹中。
```
cordova-hcp server
```
运行完这一步后，可以看到命令行内输出：
```
Running server
Checking:  /Cordova/TestProject/www
local_url http://localhost:31284
Warning: .chcpignore does not exist.
Build 2015.09.02-10.17.48 created in /Cordova/TestProject/www
cordova-hcp local server available at: http://localhost:31284
cordova-hcp public server available at: https://5027caf9.ngrok.com
```
>6. 保持cordova-hcp server运行，新开一个命令行运行cordova run，启动一个app。这里android平台需要注意以下细节：  
目前的cordova-hot-code-push-plugin与cordova-hot-code-push-local-dev-addon插件，在执行cordova run命令时，会自动尝试向android平台内的config.xml添加chcp标签配置。但是这两个插件在cordova run hook中按照cordova-android-6.0+的项目模板结构来搜索config.xml文件。如果cordova的android项目模板用的是cordova-android-7.0+，cordova run过程中会报错，无法找到config.xml文件。有以下两个方法解决这个问题:
> * 在添加android平台的时候，指定使用6.0+的模板，放弃使用最新的7.0+模板
> * 坚持使用7.0+模板，但是不适用cordova run命令。手动在android平台的config.xml文件中添加chcp标签配置，具体的config-file url参考第五步运行 cordova-hcp server 对外暴露的url。手动添加配置后，使用android studio手动打包apk运行。
```
cordova run
```
>7. 最后一步，体验网页热更新。修改cordova项目根路径的www文件夹下的html网页文件，观察app中的实时更新。



经过本地部署体验，加上了解了插件更新的工作流程和原理后，实际的项目部署也就清晰明了，基本按照以下几个步骤准备：
>1. 在nginx中开辟一个文件夹专门存放app更新网页文件，并且对外提供一个url映射。
>2. 在cordova项目中安装好 cordova-hot-code-push-plugin 插件和 cordova-hot-code-push-local-dev-addon插件（发布时最好卸载掉该插件）
>3. 配置好cordova平台内的config.xml的chcp标签
>4. 在前端工程中运行cordova-hcp init命令，输入关键配置，产生cordova-hcp.json文件。
>5. 每次前端工程编译后，记得运行cordova-hcp build命令，在编译后的网页文件中产生chcp.manifest文件和chcp.json文件。然后部署到nginx的更新部署文件夹。

## 最后附上我的工程目录截图：
><img src="/img/post/post5/project.png" alt="工程目录截图">

所有官方文档参考：
* [**cordova-hot-code-push-cl github主页**](https://github.com/nordnet/cordova-hot-code-push-cli)
* [**cordova-hot-code-push github主页**](https://github.com/nordnet/cordova-hot-code-push)
* [**cordova-hot-code-push的wiki document**](https://github.com/nordnet/cordova-hot-code-push/wiki)