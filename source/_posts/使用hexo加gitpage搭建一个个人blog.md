---
title: 使用hexo加gitpage搭建一个个人blog
date: 2018-06-19 14:34:36
categories: 
  - hexo
tags: 
  - hexo
  - blog
  - 教程
comments: false
---
>前几天端午放假，舍友旅游的旅游，回家的回家，省下自己一人呆在深圳。闲来无聊，发现了 [**hexo**](https://github.com/hexojs/hexo) 这个好玩的工具，决定用hexo加gitpage搭建一个个人blog。拥有一个比较稳定的个人blog站点一直我想做的，之前做过几次尝试，都因为没有稳定的服务器而不了了之，这次算是找到了一个行之有效的方法了。搭建blog的步骤如下：
* * *
> * 安装hexo； npm install hexo -g

> * 在本地磁盘创建blog文件夹，作为项目文件夹，然后使用hexo进行初始化；
```bash
cd blog;
hexo init;
npm install
```

> * 为了方便以后操作，在blog/package.json文件中添加scripts。hexo clean是清除部署文件，hexo g是重新编译部署文件，hexo d是进行部署。
```
"scripts": {
  "build": "hexo clean && hexo g",
  "dev": "npm run build && hexo s",
  "deploy": "npm run build && hexo d"
},
```

> * 在github新建仓库，仓库名应该为\<username\>.github.io。详情参见[**gitpage创建指南**](https://pages.github.com/)；

> * 安装hexo的git部署工具 [**hexo-deployer-git**](https://github.com/hexojs/hexo-deployer-git)，然后在配置文件blog/_config.yml中配置好部署方式与仓库路径及其分支
```
deploy:
  type: git
  repo: git@github.com:aasailan/aasailan.github.io.git
  branch: master
```

> * 选择一个顺眼的主题安装，hexo的主题多为tar包，可以直接下载解压放置在blog/themes/文件夹下，也可以使用git命令直接clone主题的仓库进行安装。在github上搜索hexo theme，可以搜出很多不错的主题，我选择了[**hexo-theme-material**](https://github.com/viosey/hexo-theme-material)这款主题，并使用git进行安装。在blog目录下执行：
```
git clone https://github.com/viosey/hexo-theme-material ./themes/material
```

> * 按照 [**material主题的安装指南**](https://material.viosey.com/docs) 进行配置。

> * 在blog目录初始化git仓库，这里使用的管理策略是创建两个分支master与project，master分支专门用来发布编译出的网页部署文件，project分支用来保存整个hexo项目。在blog目录执行以下命令，然后查看github上仓库的project分支，确认工程已经上传。
```git
git init
git remote add origin <repourl>
git checkout -b project
git add .
git commit -m '初始化项目'
git push
```

> * 发布前先开启本地服务器查看网页确认没有异常，在blog目录执行以下命令，然后在 localhost:4000 查看本地网页。确认无误后，进入下一步部署。
```
npm run dev
``` 

> * 使用npm run deploy部署。hexo会自动编译文件，并且将编译出的网页文件push到github仓库的master分支。然后访问仓库gitpage url，确认发布完成。最后来一张截图。
> 
> <img src="/img/blogjietu.png" alt="blog截图">

> 这篇教程省略了很多细节，适合熟悉node、git的用户查看。一些更详细的步骤可以查看以下两位的教程：   
>[使用hexo+github免费搭建个人博客网站超详细教程](https://www.jianshu.com/p/a39573555039)     
>[使用Hexo+Github一步步搭建属于自己的博客](https://www.cnblogs.com/fengxiongZz/p/7707219.html)

> 最后来一发今日记事：今天美国商务部宣布对中国2000亿美元的商品增加关税，中国商务部强硬反击，但是A股已经跌成狗了。哎，真是惨。