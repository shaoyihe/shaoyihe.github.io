---
title:  "zookeeper服务注册例子"
date:   2016-05-08 16:50:00 +0800
tags: [zookeeper]
---

# zookeeper服务注册例子
---
#### 解决的问题

​	随着项目扩大，根据不同的维度（业务，访问频率等）对项目进行拆分为不同的模块。不同的模块进行单独部署，目前的问题是不同的模块需要通信，比如把用户权限模块单独部署，那么所有的其他模块都需要访问该模块进行鉴权。如图所示：

![服务模块](/assets/images/server-communication.png)

内部模块通讯也可以使用`Nginx`或配置文件配置哪台服务器提供服务（`ip:port`），但是明显ip不容易记忆，而且这些服务器明显可以动态增加，删除。

所以目前都是服务注册机制。即服务提供方发布自己的`ip:port`作为一个服务名， 服务调用方根据服务名加载全部的服务，然后直接调用提供方。如图示：

![服务注册模块](/assets/images/server-communication-with-register.png)

基本步骤是：

* 用户权限服务器向服务注册模块注册自己的服务（`ip:port`）为`user-auth`。
* 商品信息服务器从注册模块下周提供`user-auth`的所有服务。
* 商品信息服务器直接从备选`user-auth`服务器轮训调用服务。

#### 注册机制

目前提供该类似服务的有[zookeeper](http://zookeeper.apache.org/)，[consul](https://www.consul.io/)，[etcd](https://github.com/coreos/etcd)等。

不管哪种机制，都需要满足：

* 动态添加/删除当前服务
* 心跳检查，如果一段时间不能服务提供方不能响应或发起，则删除该提供方
* 集群共享

下面以zookeeper说明服务注册。

#### 样例

先说样例。

必备条件：

* 安装[Jdk8](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
* 安装[maven](http://maven.apache.org/)
* 安装[node](https://nodejs.org/)
* 安装[zookeeper](http://zookeeper.apache.org/releases.html)

我用的Mac，上面软件都是用[brew](http://brew.sh/)自动安装的。windows环境需要单独下载，linux查看对应的包管理工具。测试步骤如下：

* 启动zookeeper(zoo.cfg文件路径不同环境可能可能不同)：

  ```
  ○ → zkServer start  /usr/local/etc/zookeeper/zoo.cfg
  ZooKeeper JMX enabled by default
  Using config: /usr/local/etc/zookeeper/zoo.cfg
  Starting zookeeper ... STARTED
  ```

  ​


* 下载样例代码[zookeeper example](https://github.com/shaoyihe/zookeeper-example)

* 命令行窗口打开样例目录，然后启动服务提供方（可以打开多个窗口，模拟多个提供方）：

  ```
  ± |master U:2 ?:2 ✗| → cd zookeeper-server

   2016-05-08 16:03:13 ☆  heshaoyideMacBook-Pro in ~/IdeaProjects/zookeeper-server/zookeeper-server
  ± |master U:2 ?:2 ✗| → mvn clean package && java -jar target/zookeeper-server-1.0-SNAPSHOT.jar
  [WARNING]
  [WARNING] Some problems were encountered while building the effective settings
  ...
  2016-05-08 16:03:25.022  INFO 7920 --- [           main] com.he.Application                       : Started Application in 3.842 seconds (JVM running for 4.332)
  register {"protrol":"http","host":"100.78.143.183","port":59491} to /servers/test-me/0000000825
  ```


* 启动客户端

  ```
   2016-05-08 16:05:39 ☆  heshaoyideMacBook-Pro in ~/IdeaProjects/zookeeper-server
  ± |master U:2 ?:2 ✗| → cd zookeeper-client/

   2016-05-08 16:05:42 ☆  heshaoyideMacBook-Pro in ~/IdeaProjects/zookeeper-server/zookeeper-client
  ± |master U:2 ?:2 ✗| → npm start

  > zookeeper-client@1.0.0 start /Users/heshaoyi/IdeaProjects/zookeeper-server/zookeeper-client
  > node ./app.js

  listening 3000
  find new serverDatas http://100.78.143.183:54487,http://100.78.143.183:59491,http://100.78.143.183:54490
  ```


* 浏览器打开`http://localhost:3000/`，效果如下：

  ![服务发现效果](/assets/images/zookeeper-example-efect.gif))

  每次调用客户端服务，就会就会轮训调用不同的服务提供方。

  ​



