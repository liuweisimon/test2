部署配置文件主要包含以下几节内容
1. 部署
2. 作业。


部署
----------
有关您的部署的一些基本信息。


例如
---
deployment:
  # 此部署的名称
  name:"sample_deployment"
  # 将作为 cloudfoundry 运行身份的用户，默认为运行安装脚本的
  # 用户
  # user:"<user>"

  # cloudfoundry 所属的组，默认为运行安装脚本的用户所属的
  # 组
  # group:"<group>"


作业
----
作业表示各种可安装的 cloud foundry 组件
这些作业分成 2 组
- 安装（应在此服务器/主机上安装的作业）
- 已安装（这类作业已安装在其他服务器/主机上且其属性是
             计划在此服务器/主机上安装的作业所需要的）


假设您要在 server_A 上安装“nats”。那么，相应的部署配置文件将包含以下作业。此处我们还为我们要安装的 nats 作业指定了属性


例如
---
deployment:
  name:"nats_server"
jobs:
  install:
    nats:
      host:<server_A>
      port:<nats_port>
      user:<nats_user>
      password:<nats_password>


现在，如果您要在 server_B 上安装“dea”。那么，用于 server_B 的部署配置文件将类似于下面的内容。这种情况下“已安装”的作业 nats 及其属性是 router 所需要的。


例如
----
deployment:
  name:"dea_server"
jobs:
  install:
   - router
  installed:
   - nats:
        host:<server_A>
        port:<nats_port>
        user:<nats_user>
        password:<nats_password>


下面是目前受支持的作业的一个示例
 - all（部署全部 cloudfoundry 组件）
 - nats
 - router
 - cloud_controller
 - health_manager
 - dea
 - mysql_node
 - mysql_gateway
 - redis_node
 - redis_gateway
 - mongodb_node
 - mongodb_gateway
 - neo4j_node
 - neo4j_gateway

在多主机环境中上手使用的最简单方式就是着手自定义
此目录中存在的各种示例配置文件。

注意：受支持的多主机环境
这些脚本目前支持以下多主机环境。DEA、服务和
路由器可以配置为运行在不同的主机上。
不过，以下组件应全部运行在同一主机上。
- cloud_controller
- health_manager
- 服务网关，如 mysql_gateway、mongodb_gateway、redis_gateway 和 neo4j_gateway


