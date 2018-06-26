---
title: centos7下docker安装指南
date: 2018-06-26 17:28:14
categories: 
  - docker
tags: 
  - docker
  - centos
comments: false
---
今天在百度云vps上面安装了docker和docker compose，操作系统是centos7，觉着应该把安装过程大致记录下，防止以后忘记。整个安装并不复杂，主要有以下几个步骤：  
* 根据[**官网的安装文档**](https://docs.docker.com/install/linux/docker-ce/centos/)安装docker。主要是安装一些必要软件以及docker的仓库源，最后安装docker，有以下命令需要运行:
 ```
 # 安装必要软件
 $sudo yum install -y yum-utils \
 device-mapper-persistent-data \
 lvm2

 # 安装docker仓库源
 $sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

 # 安装docker
 sudo yum install docker-ce

 # 启动docker服务
 $sudo systemctl start docker

 # 将docker设置为开机启动
 sudo systemctl enable docker
```

* 设置[**Docker中国官方镜像加速**](https://www.docker-cn.com/registry-mirror)。修改修改 /etc/docker/daemon.json 文件并添加上 registry-mirrors 键值，然后重启docker。
```
{
   "registry-mirrors": ["https://registry.docker-cn.com"]
}

# 修改后重启docker
$sudo systemctl start docker
```

* [**使用非root用户管理docker**](https://docs.docker.com/install/linux/linux-postinstall/#manage-docker-as-a-non-root-user)。主要是添加docker用户组，然后把当前用户加入docker用户组：
```
$ sudo groupadd docker
$ sudo usermod -aG docker $USER
# 运行上面两句后，logout再重新login。再运行hello world来检查非root用户是否可以运行docker
$docker run hello-world
```

* 安装docker compose。安装之前，要先完成上一步（使用非root用户管理docker）
```
$sudo curl -L https://github.com/docker/compose/releases/download/1.21.2/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose

$sudo chmod +x /usr/local/bin/docker-compose

# 验证docker compose 已安装完毕
$docker-compose --version
```


上个周末在daocloud里面体验了一把基于docker的持续集成与部署。demo比较简单，没有太多参考价值。自己有空再思考跟找一下持续集成与部署跟git flow流程如何结合。然后再写一个demo出来。