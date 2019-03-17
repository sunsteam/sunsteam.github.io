---
layout:     post
title:      "Debian 9 中使用 Docker Cli"
subtitle:   ""
date:       2018-11-27 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Tools
---


### 准备工作

- 安装Docker参考 [官方文档](https://docs.docker-cn.com/engine/installation/linux/docker-ce/debian/)
- Docker命令参考 [Docker 命令大全](http://www.runoob.com/docker/docker-command-manual.html)

### 运行Jenkins

1. 获取长期支持版Jenkins镜像
    `$ docker pull jenkinsci/jenkins:lts`
        
2. 创建`/usr/local/work/jenkins`存放Jenkins生产文件的目录，否则容器停止后文件将丢失
3. 赋予访问权限
    `$ chmod 777 /usr/local/work/jenkins`
4. 创建镜像的容器

    **--restart=on-failure:1** 错误关闭后重启一次，正常关闭不会重启
    **--name=jenkins** 设置容器别名
    **-idt** i:可交互，d:后台运行，t:创建伪命令行
    **-v /usr/local/work/jenkins:/var/jenkins_home** 将容器目录映射到本地
    **-p 9123:8080 -p 50000:50000** 将容器暴露的端口映射到本地端口（冒号前的）
    **jenkinsci/jenkins:lts** 容器所用的镜像名称，冒号后为tag
    
    ```shell
    $ docker run --restart=on-failure:1 --name=jenkins -idt -v /usr/local/work/jenkins:/var/jenkins_home -p 9123:8080 -p 50000:50000 jenkinsci/jenkins:lts
    ```
    
5. 进入系统 `/usr/local/work/jenkins` 目录，根据提示找到初始化秘钥进行初始化

6. 假设需要更改配置的参数，那么需要复制一个新容器出来
    ```shell
    $ docker run --restart=on-failure:1 --name=jenkins -idt --volumes-from jenkinsold -v /usr/share/android:/var/android_home -p 9123:8080 -p 50000:50000 jenkinsci/jenkins:lts
    ```

7. 注意：在安装jenkins时候，挂载给Jenkins容器的目录的归属用户id必须是1000，否则会抛出无操作权限异常，这是Jenkins官方DockerFile中要求的。

    ![](media/15426212748961/15427887417266.jpg)

    ```shell
    # 查看目录所属id
    $ ls -al /usr/share/android
    # 更改目录所属id,group
    $ chown -R 1000:1000 /usr/share/android
    # 重启容器
    $ docker restart jenkins
    ```

### 运行 MySQL
1. 运行 mysql 容器
    ```shell
    # 获取 mysql 镜像
    $ docker pull mysql
    # 创建 mysql 挂载数据的数据卷
    $ docker volume create mysql_home
    # 运行容器
    $ docker run --name mysqlserver --restart=always -v mysql_home:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=xxxxx -p 3306:3306 -d mysql
    ```
    
1. 运行 phpmyadmin 容器，管理 mysql
    ```shell
    $ docker run --name phpmyadmin --restart=always --link mysqlserver:db -p 9090:80 -d phpmyadmin/phpmyadmin
    ```
   