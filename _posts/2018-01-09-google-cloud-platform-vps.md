---
layout:     post
title:      "利用 Google Cloud Platform 获得免费的 VPS 服务器"
subtitle:   ""
date:       2018-01-09 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - common
    - linux
    - vps
---

>Google Cloud Platform （后文称 GCP）对新用户提供了 1 年的免费使用期限和 300 美元的赠送金额进行体验。 他是按配置进行收费，可以自定义 CPU、内存、硬盘来调整到合适的价格，因此比购买套餐定制更灵活。但如果是作为梯子使用的话，GCP 会按照流量进行计费的，如果流量大的话还是不太合算。我是用来熟悉一下各个 Linux 系统，部署点小工具。


### 1. 创建一个虚拟机实例

- ##### 1.1 需要一张 Visa 或者 JCB 的信用卡，用来在 [cloud.google.com](https://cloud.google.com/) 激活你的 Google 账号

- ##### 1.2 地址随便填就好了，但是电话留真的以备账号找回等需要使用

- ##### 1.3 跟随 Compute Engine 版块的入门教程，创建一个项目。

- ##### 1.4 在项目中创建 VM 实例。

    可以选择最低等级的 5 美元档做试验，1 共享核 + 0.6 GB 内存 + 10GB HDD。入门教程要求 Ubuntu 系统，不喜欢的话可以跳过教程更换别的。

    * 1.4.1 服务器位置选择 Asia-east 或者 Asia-northeast 都挺快的 （香港或日本）

    * 1.4.2 自定义机器类型，拖到最低

    * 1.4.3 启动磁盘 可以更改操作系统镜像，基本能满足需求，不行的话可以自定义

    * 1.4.4 防火墙 允许 http 和 https 流量

### 2. 使用 SSH 客户端访问虚拟机

在实例创建后可以点击 SSH 按钮，通过网页终端发送一个临时秘钥 SSH 到虚拟机上，但是网页终端比较慢，所以我们要本地生产一对秘钥并放到虚拟机上，通过 SSH 软件来访问。

- ##### 2.1 在本地生成一对 rsa 秘钥，Windows 系统下默认不会有 ssh-agent 这个工具，不过 git bash 里有。打开 git bash 输入下面的命令，秘钥默认会以 id_rsa 的名字生成到~/.ssh/ 文件夹下， 但会询问你是否更改文件名，如果改的话就要把路径输入全，否则生成的文件会跑到 git bash 所在目录。然后回车两次生成完毕。

    ```
    # -C 后面的字符串是秘钥描述，GCP 要求填 Gmail 邮箱 不允许空格
    $ ssh-keygen -t rsa -b 4096 -C "xxxx@gmail.com"
    ```
- ##### 2.2 将私钥添加到 `ssh-agent`

    ```
    # start the ssh-agent in the background
    $ eval $(ssh-agent -s)
    # add private key
    $ ssh-add ~/.ssh/your_key_name
    ```

- ##### 2.3 将公钥上传

    将公钥添加到虚拟机上的~/.ssh/authorized_keys 文件中才能使服务器用它来加密，GCP 上默认禁止了密码登陆方式，不过提供了界面方便进行这一步。

    * 2.3.1 使用编辑器打开 xxx.pub 文件， 复制内容

    * 2.3.2 回到 GCP，进入 Compute Engine 》 元数据 》 SSH 秘钥 界面 》 修改 添加一项 》 黏贴内容并保存。 添加的公钥会在所有实例中生效，因此新建了虚拟机就直接可以用现有秘钥了。

- ##### 2.4 使用 SSH 客户端访问虚拟机

    * 2.4.1 在 SSH 客户端中发起一个链接，输入外部 ip 地址，端口 22 先不用改，链接。

    * 2.4.2 当询问用户名的时候，GCP 建立虚拟机的时候默认会按邮箱名字帮你创建好用户，不确定的话可以先网页 SSH 到虚拟机看下

    * 2.4.3 浏览并选择导入你的私钥， 密码生成的时候没输的话就不用填

    ![add_private_key](https://ws1.sinaimg.cn/large/006tNc79gy1foo0k5y3kej30p80cf0ts.jpg)

### 3. 更改 SSH 默认监听的 22 端口

上一步确定之后应该已经连上了虚拟机，那么为了安全性我们把 ssh 程序默认端口给改了。

- ##### 3.1 修改 sshd_config 文件，添加一个监听端口

    ```
    $ sudo vi /etc/ssh/sshd_config
    ```

    使用 vim 修改，输入 i 进入编辑状态，就可以修改内容，完了使用 `:w` 保存 `:q` 退出，也可以连用 `:wq`。不同系统下这个文件内容不太一样，不过总之确保几点, 下面这几条都找得到并且前面没有 `#`

    ```
    Port 22
    Port xxx // 加一个监听端口 1024 ~ 65535 里选个喜欢的数字
    PermitRootLogin no
    PasswordAuthentication no
    PermitEmptyPassword no
    ```

- ##### 3.2 重新运行 sshd 服务使修改生效

    ```
    # 查看服务状态
    $ sudo service sshd status
    # 重启服务
    $ sudo service ssh restart
    # 再看一遍状态 进程号改了就成功了
    ```

- ##### 3.3 在 GCP 防火墙设置中放行刚刚监听的 xxx 端口。

    GCP 导航栏 》 VPC 网络 》 防火墙规则 》 入站， 参照 default-allow-ssh 的设置，添加一条规则，协议 / 端口这部分写的应该是 tcp:xxx 其他都一样 （不要直接修改 default-allow-ssh 这条规则，22 端口还是留着，否则新建的实例连不上）。这个设置同样对项目中的所有实例生效。

- ##### 3.4 修改 SSH 客户端中的连接端口，重新连接成功之后。再次进入 sshd_config，把 `Port 22` 这一行前面加个 `#` 注释掉

### 后话

开始玩耍吧，300美元扣完之后不会扣你信用卡里的钱，但不要手贱点了申请续费什么的，类似的选项都注意一下。

* 云端启动器中有许多现成的配置方案可以试试

* App Engine可以直接部署应用

* 时常看看结算界面，保证没有预料外的费用发生
