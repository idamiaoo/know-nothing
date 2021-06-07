---
title: 利用 drone 和 gogs 搭建部署平台 
date: 2016-11-15 16:55:23
category: 工具
tags:
    - drone
    - CI,CD
    - gogs
---

[Gogs](https://github.com/gogits/gogs)是一个使用go语言开发的自助git服务，支持所有平台  
[Docker](https://github.com/docker/docker)是使用go开发的开源容器引擎  
[Drone](https://github.com/drone/drone)是一个基于容器技术的持续集成平台。每个构建都在一个临时的Docker容器中执行，使开发人员能够完全控制其构建环境并保证隔离。drone易于安装和使用，其目标是替代jenkins 

本文所实现的的功能为当你push代码到gogs时，自动更新您测试环境的二进制文件并重启，实现自动部署（以go开发api服务为例,测试环境为ubuntu）

整个流程为:

1. push code
2. drone搭建临时容器拉取最新代码编译，在临时容器内通过scp拷贝编译好的二进制文件至测试服务器，然后通过ssh控制测试环境应用服务重启(supervisorctl)

### 步骤(ubuntu)

本文默认您已经安装好gogs和docker,以及使用supervisor部署应用服务(可选为其他部署方式)

1. 安装docker
    具体安装步骤可见[官网文档](https://docs.docker.com/engine/installation/)

2. 安装gogs
    [官网安装文档](https://gogs.io/docs/installation)(需翻墙,也可自行搜索相关安装文档)

3. 安装drone(v0.5)
    通过docker安装
    1. 下载drone镜像

        ```bash
        docker pull drone/drone:0.5
        ```

    2. 启动drone server
        ```bash
         docker run -d \
            -e DRONE_DEBUG=true \
            -e DRONE_GOGS=true \
            -e DRONE_GOGS_URL=http://cmkj.lunarhalo.cn \
            -e DRONE_SECRET=... \
            -e DRONE_OPEN=true \
            -v /var/lib/drone:/var/lib/drone \
            -p 8000:8000 \
            --restart=always \
            --name=drone \
            drone/drone:0.5
        ```
        该命令启动的是一个以sqlite做为存储数据库，可选配mysql,postgres可根据自己情况进型配置，见[文档](http://readme.drone.io/0.5/)
    drone启动成功，可以通过网页访问，使用gogs账号登录，找到项目开启管理。

    3. 启动drone agent

        ```bash
         docker run -d \
            -e DRONE_SERVER=ws://172.17.0.1/ws/broker \
            -e DRONE_SECRET=... \
            -v /var/run/docker.sock:/var/run/docker.sock \
            --restart=always \
            --name=drone-agent \
            drone/drone:0.5 agent
        ```

4. 生成定制golang镜像(在.drone.yml配置置该镜像作为构建镜像)

    1. pull一个base镜像  

        docker pull goang:latest 可选择版本
    
    2. 定制镜像

        1. 创建并启动golang容器
        
        ```bash
        docker run --ti golang:latest /bin/bash
        ```
        2. 生成ssh公钥，并设置ssh免密登录测试服务器  
            **容器内**:  
            - 执行ssh-keygen -t rsa   
            - 会在$HOME/.ssh目录下生成id_rsa和id_rsa.pub  
            - 将id_rsa.pub通过scp拷贝至测试服务器  
            **测试服务器**:    
            - 在home目录下建立.ssh文件夹
            - 并cat id_rsa.pub >> .ssh/authorized_keys
            - chmod 600 .ssh/authorized_keys  
            ssh免密密登录已配置好

            下载自己项目需要的依赖包 go get ...(官方golang镜像的GOPATH为/go)  
            准备好之后退出容器，并把在容器里面的修改保存为一个新的镜像  
            如：docker commit [容器id] golang:dev

5. 在项目根路径添加.drone.yml文件
    配置示例:
    ```yml
    workspace:
        base: /root/go
        path: src/projectname


    pipeline:
        build:
            image: golang:dev //指定构建镜像
        environment: 
            - GOPATH=/go:/root/go
            - SSH_ARGS=-p 22 -o StrictHostKeyChecking=no（设置第一次登录时不需要输入yes）
            - SCP_ARGS=-P 22 -o StrictHostKeyChecking=no
            - BUILD_NAME=buildname
            - APP_NAME=appname
            - TEST_SERVER=root@172.17.0.1
            - RUN_PATH=/data/go/project(配置自己测试环境应用保存运行地址)
        commands:
            - go build -o $BUILD_NAME
            - eval $(ssh-agent -s)
            - ssh-add /root/.ssh/id_rsa
            - scp $SCP_ARGS "$BUILD_NAME" "$TEST_SERVER":"$RUN_PATH"/"$BUILD_NAME"_"$(date '+%Y%m%d')"_"$(git rev-parse HEAD| cut -c1-10)"  //拷贝文件
            - ssh $SSH_ARGS "$TEST_SERVER" "ln -s -f $RUN_PATH/$BUILD_NAME\_$(date '+%Y%m%d')_$(git rev-parse HEAD| cut -c1-10) $RUN_PATH/$BUILD_NAME && supervisorctl restart $APP_NAME" //重启，利用软连接实现备份
    ```
    - workspace: 工作路径，根据如上配置，会把你的项目克隆到 /root/go/projectname， 且$PWD=/root/go/projectname
    - image: 指定构建镜像                  
    - environment: 构建临时容器的环境变量，相当于 docker run -e .....
    - commands: 在容器内shell上执行的命令
    
    上面配置文件中设置了两个GOPATH是因为在之前的测试中，如果我设置  
        base/root  
        GOPATH=/go   
    会把之前镜像下载的依赖包给清空了，不知是哪里配错了还是什么原因，暂时只找到设置另个GOPATH的方法来解决
    
    编写好.drone.yml后加入仓库push代码便会自动构建部署了