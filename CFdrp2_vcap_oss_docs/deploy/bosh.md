#什么是BOSH#

##概述##

Cloud Foundry 是业界首款开源 PaaS。它支持多种框架、多项服务和多家云提供商。BOSH 原本是在 Cloud Foundry 项目的背景下产生的。不过，它已成为一个通用的工具链，用于对大规模的分布式服务进行部署和生命周期管理。

Cloud Foundry 包含多个组件。其中最重要的组件是 Cloud Controller、NATS、Router、 HealthMonitor 和 DEA。可通过下面的链接可找到对这些组件的介绍：《开源 PaaS Cloud Foundry 深度解析》[http://cnblog.cloudfoundry.com/?p=46](http://cnblog.cloudfoundry.com/?p=46)和《新版Cloud Foundry揭秘》[http://cnblog.cloudfoundry.com/?p=31](http://cnblog.cloudfoundry.com/?p=31)。这些组件的设计可以使系统能水平的扩展。也就是说，一个Cloud Foundry 实例可以包含每个组件的一份或多份副本，以满足云所需的负载。这些组件可以分散地部署在多个节点上。

BOSH 是一款供我们用来将 Cloud Foundry 组件部署到分布式节点上的工具。（在虚拟化环境中， “节点”一词可以与“虚拟机”或 VM 互换使用）。在开始详细阐述真实部署前，我们先来简要介绍一下 BOSH 自动化部署系统的工作原理。我们建议您阅读下面的 BOSH 官方文档。[https://github.com/cloudfoundry/oss-docs/blob/master/bosh/documentation/documentation.md](https://github.com/cloudfoundry/oss-docs/blob/master/bosh/documentation/documentation.md)，文档的中文翻译可以在这里获得，[http://cndocs.cloudfoundry.com/deploy/bosh-docs.html](http://cndocs.cloudfoundry.com/deploy/bosh-docs.html)。

BOSH 是 Bosh OutterSHell（Bosh 外壳）的递归缩写词。相对于“外壳”（Outter Shell），由 BOSH 部署和管理的系统则称作“内壳”（Inner Shell）。下图显示了一个简化的 BOSH 模型。

可以将 BOSH 视作一台负责协调分布式系统部署过程的服务器或机器人。有个Ruby 工具可以与 BOSH 命令行界面 (CLI) 进行交互。BOSH 需要以下三个必备项才能开始部署系统：一个stemcell、一个Release（要安装的软件）和一个部署清单(Deployment Manifest)。我们来更详细地了解一下这三项内容。

Stemcell：在云平台中，虚拟机通常是从模板克隆而来的。一个stemcell就是一个包含标准 Ubuntu  Linux的虚拟机模板。该模板中还嵌入了一个 BOSH 代理，以便 BOSH 可以控制从该stemcell克隆出来的虚拟机。“stemcell”这个名字源自于“stem cell”（干细胞）这个生物学术语，该术语指的是能够生成各种细胞的未分化细胞。同样，一个 BOSH stemcell所创建的各个虚拟机起初也是完全相同的。初始化后，这些虚拟机便配置了不同的 CPU、内存、存储和网络参数，并装有不同的软件包。因此，基于同一个stemcell模板构建的虚拟机会表现出不同的行为。

Release：Release包含若干组将要安装到目标系统上的软件代码和配置。每个虚拟机上都要部署一组软件，这组软件称作一个作业（job）。配置通常是包含诸如 IP 地址、端口号、用户名、密码、域名等参数的模板。这些参数在部署时将被部署清单文件中定义的属性所替代。

部署：部署就是使静态的Release变成虚拟机上可运行的软件的过程。部署清单定义了部署所需的实际参数值。在部署过程中，BOSH 会替换掉Release中的参数，从而使软件按照我们规划的配置来运行。

当上述 3 项内容都准备好后，BOSH CLI 工具会将它们上传到 BOSH。接着，用BOSH 来安装分布式系统包括以下主要步骤：

1) 如果Release中的某些包需要编译，BOSH 首先会创建几个临时虚拟机（worker，工作者虚拟机）来编译它们。编译完后，BOSH 便会销毁这些工作者虚拟机，将所产生的二进制代码存储在其内部blobstore中。
		
2) BOSH 创建一个虚拟机池，池中的虚拟机将成为该Release要部署到的节点。这些虚拟机是从装有 BOSH 代理的stemcell克隆而来的。BOSH使用CPI接口调用vSphere的虚拟机创建操作API，来自动化的完成虚拟机的创建和配置工作。CPI接口同样适合于OpenStack、AWS等其他IaaS管理平台。
		
3) 对于该Release的每个作业，BOSH 会从该池中选取一个虚拟机，然后根据部署清单更新该虚拟机的配置。具体的配置可以包括 IP 地址、持久磁盘的大小等。
		
4) 重新配置完该虚拟机后，BOSH 会向每个虚拟机内的代理发送命令。这些命令通知该代理安装软件包。在安装期间，该代理可能会从 BOSH 下载包并安装它们。安装完毕后，该代理会运行启动脚本来启动该虚拟机的作业。
		
5) BOSH 重复执行第 3 步至第 4 步，直至所有作业都部署完毕并启动为止。这些作业可以同时部署，也可以按顺序部署。清单文件中的“max_in_flight”值用于控制并行部署的作业数。当该值为 1 时，表示这些作业逐一按顺序部署。对于较慢的系统，该值有助于避免因资源拥塞而造成的超时。该值大于 1 时，表示这些作业可以并行部署。
