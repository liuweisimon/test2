#什么是BOSH（deploy/bosh.MD）#

##概述##

Cloud Foundry 是业界首款开源 PaaS。它支持多种框架、多项服务和多家云提供商。BOSH 原本是在 Cloud Foundry 项目的背景下产生的。不过，它已成为一个通用的工具链，用于对大规模的分布式服务进行部署和生命周期管理。

Cloud Foundry 包含多个组件。其中最重要的组件是Cloud Controller、NATS、Router、 HealthMonitor和 DEA。可通过下面的链接可找到对这些组件的介绍：《开源 PaaS Cloud Foundry 深度解析》http://cnblog.cloudfoundry.com/?p=46和《新版Cloud Foundry揭秘》http://cnblog.cloudfoundry.com/?p=31。这些组件的设计可以使系统能水平的扩展。也就是说，一个Cloud Foundry 实例可以包含每个组件的一份或多份副本，以满足云所需的负载。这些组件可以分散地部署在多个节点上。

BOSH 是一款供我们用来将 Cloud Foundry 组件部署到分布式节点上的工具。（ 在虚拟化环境中， “节点”一词可以与“虚拟机”或 VM 互换使用）。在开始详细阐述真实部署前，我们先来简要介绍一下 BOSH 自动化部署系统的工作原理。我们建议您阅读下面的 BOSH 官方文档。https://github.com/cloudfoundry/oss-docs/blob/master/bosh/documentation/documentation.md，文档的中文翻译可以在这里获得，http://cndocs.cloudfoundry.com/deploy/bosh-docs.html。

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

#在vSphere上部署（deploy/vSphere.MD）#

##本文根据Cloud Foundry中国架构师团队的实际部署经验总结而成，共分三个部分。##

		-	准备IaaS环境
		-	安装BOSH
		-	部署Cloud Foundry

##作者介绍##

张轩宁，有多年软件工程的从业经历，在移动应用开发、云应用的架构设计以及软件开发周期管理等方面有着丰富的经验。目前，张轩宁是VMware中国公司负责云应用生态系统的资深架构师，他帮助许多国内的运营商实施了公有云和私有云的解决方案。作为开发者关系团队的成员，他还负责开放云计算平台Cloud Foundry的开发者社区的技术支持工作。在加入VMware之前，张轩宁曾在IBM和SUN等跨国公司任职，作为解决方案架构师，帮助国内外金融、电信和医疗等行业的企业客户成功的实施了许多大型项目。

陈实，VMware公司Cloud Foundry部门实习生，复旦大学计算机科学技术学院硕士研究生，研究兴趣包括但不限于：云计算与云计算安全，PaaS架构与应用，网络与信息安全。现在VMware公司从事CloudFoundry的相关工作，主要工作领域为：通过BOSH大规模部署CloudFoundry实例，CloudFoundry在不同IaaS层的部署、集成与测试，对CloudFoundry进行自动化测试等。

陈威，VMware公司Cloud Foundry部门实习生，研究方向为Cloud Foundry生态系统的搭建，使用BOSH和Dev-Setup搭建Cloud Foundry，研究添加服务和框架与Cloud Foundry平台的集成，以及周边环境的建设，构建起本地健康的Cloud Foundry生态系统。目前是南京大学计算机系研究生三年级学生，云计算技术爱好者，关注移动互联网，喜欢捣鼓新奇玩意儿，偶尔也会拿起相机拍拍照片。

#准备IaaS环境（deploy/vSphere-IaaS.html）#

用BOSH部署 Cloud Foundry 的完全手册

VMWare中国研发中心

Henry Zhang、Victor Chen、Wei Chen

##第I 部分##

###概述###

Cloud Foundry 是业界首款开源 PaaS。它支持多种框架、多项服务和多家云提供商。BOSH 原本是在 Cloud Foundry 项目的背景下产生的。不过，它已成为一个通用的工具链，用于对大规模的分布式服务进行部署和生命周期管理。本系列由 4 部分组成，将一步步地向您介绍使用 BOSH安装 Cloud Foundry 平台的过程。

Cloud Foundry 包含多个组件。其中最重要的组件是云控制器、NATS、router、运行 状况管理器和 DEA。可通过下面的链接可找到对这些组件的介绍：http://blog.cloudfoundry.com/2011/04/19/cloud-foundry-open-paas-deep-dive/。这些组件的设计可以使系统能水平的扩展。也就是说，一个Cloud Foundry 实例可以包含每个组件的一份或多份副本，以满足云所需的负载。这些组件可以分散地部署在多个节点上。

BOSH 是一款供我们用来将 Cloud Foundry 组件部署到分布式节点上的工具。（ 在虚拟化环境中，“节点”一词可以与“虚拟机”或 VM 互换使用）。在开始详细阐述真实部署前，我们先来简要介绍一下 BOSH 自动化部署系统的工作原理。我们建议您阅读下面的 BOSH 官方文档。()

BOSH 是 Bosh Outter SHell（Bosh 外壳）的递归缩写词。相对于“外壳”，由 BOSH 部署和管理的系统则称作“内壳”（Inner Shell）。下图显示了一个简化的 BOSH 模型。

可以将 BOSH 视作一台负责协调分布式系统部署过程的服务器或机器人。有个Ruby 工具可以与 BOSH 命令行界面 (CLI) 进行交互。BOSH 需要以下三个必备项才能开始部署系统：一个 stemcell、一个Release（要安装的软件）和一个部署清单(Deployment Manifest)。我们来更详细地了解一下这三项内容。

Stemcell：在云平台中，虚拟机通常是从模板克隆而来的。一个 stemcell 就是一个包含标准 Ubuntu  Linux的虚拟机模板。该模板中还嵌入了一个 BOSH 代理，以便 BOSH 可以控制从该 stemcell 克隆出来的虚拟机。“stemcell”这个名字源自于“stem cell”（干细胞）这个生物学术语，该术语指的是能够生成各种细胞的未分化细胞。同样，一个 BOSH stemcell 所创建的各个虚拟机起初也是完全相同的。初始化后，这些虚拟机便配置了不同的 CPU/内存/存储/网络，并装有不同的软件包。因此，基于同一stemcell 模板构建的虚拟机会表现出不同的行为。

Release：Release包含若干组将要安装到目标系统上的软件代码和配置。每个虚拟机上都要部署一组软件，这组软件称作一个作业(job)。配置通常是包含诸如 IP 地址、端口号、用户名、密码、域名等参数的模板。这些参数在部署时将被部署清单文件中定义的属性所替代。

部署：部署就是使静态的Release变成虚拟机上可运行的软件的过程。部署清单定义了部署所需的实际参数值。在部署过程中，BOSH 会替换掉Release中的参数，从而使软件按照我们规划的配置来运行。

当上述 3 项内容都准备好后，BOSH CLI 工具会将它们上传到 BOSH。接着，用BOSH 来安装分布式系统包括以下主要步骤：

		1) 如果Release中的某些包需要编译，BOSH 首先会创建几个临时虚拟机（worker,工作者虚拟机）来编译它们。编译完后，BOSH 便会销毁这些工作者虚拟机，将所产生的二进制代码存储在其内部 blobstore 中。
		2) BOSH 创建一个虚拟机池，池中的虚拟机将成为该Release要部署到的节点。这些虚拟机是从装有 BOSH 代理的 stemcell 克隆而来的。
		3) 对于该Release的每个作业，BOSH 会从该池中选取一个虚拟机，然后根据部署清单更新该虚拟机的配置。具体的配置可以包括 IP 地址、持久磁盘的大小等。
		4) 重新配置完该虚拟机后，BOSH 会向每个虚拟机内的代理发送命令。这些命令通知该代理安装软件包。在安装期间，该代理可能会从 BOSH 下载包并安装它们。安装完毕后，该代理会运行启动脚本来启动该虚拟机的作业。
		5) BOSH 重复执行第 3 步至第 4 步，直至所有作业都部署完毕并启动为止。这些作业可以同时部署，也可以按顺序部署。清单文件中的“max_in_flight”值用于控制并行部署的作业数。当该值为 1 时，表示这些作业逐一按顺序部署。对于较慢的系统，该值有助于避免因资源拥塞而造成的超时。该值大于 1 时，表示这些作业可以并行部署。

我们将在后文中对上述步骤进行详细解释。开始安装部署前，我们先讨论一下硬件和软件方面的前提条件。

###软件：###

		1) 64 位 Ubuntu 10.04 LTS，最好是 ISO 格式。
		2) vSphere V4.1 或 V5.x（本文采用vSphere 作为hypervisor）
		3) vSphere Client
		4) vCenter（安装在 Windows 2008 R2 64 位或 Windows 2003 服务器上，物理机或虚拟机皆可）

上述软件的60天或90天评估使用版本，均可以在对应公司的官方网站下载获得。

###硬件：###

假设所有节点都是虚拟机，下表显示了所需的虚拟机数目：

| 	|节点数目   |操作系统   |可否是物理机|
|-------|-----------|-----------|------------|
|BOSH CLI|   1      |	Ubuntu	|可以|
|vCenter+vSphere Client|   1  |	Win2008	|可以安装在一起，也可划分成两个节点|
|Micro BOSH|	1   |	Ubuntu |不可以|
|BOSH|	6  |	Ubuntu|	不可以|
|Cloud Foundry|	34   |	Ubuntu|	不可以，见下文|
|-------|-----------|-----------|------------|
|合计：|	43| | |	 	 

注意：上表中 Cloud Foundry 的节点数目是所需的最少节点数目。此数目可能会因实际的 Cloud Foundry 部署规模而异。选择硬件配置时通常要考虑两个原则：

		1) vCPU总数不应超过物理核心总数的两倍。在生产系统中，两者之比应该接近于 1。
		2) 所有虚拟机的总内存应小于所有hHypervisor的物理内存。

下面例子是假设每个虚拟机有 4 GB 内存和 1 个vCPU时的硬件配置：

		6 台服务器，每台服务器有 8 核 CPU和 32GB RAM。

就实验系统而言，我们曾在一台配置如下的服务器上成功部署（假定每个虚拟机有 256 MB 内存）：

		1 台物理服务器：8 核 CPU，16 GB RAM。

对于生产环境，我们建议选择CPU核数和内存容量都比较大的机型，这样在同一台物理机，可以运行更多的虚拟机。同时需要考虑有比较高吞吐量的网卡和存储设备。

除了服务器之外，存储也是云平台中的一个关键要素。存储最好应有 200 GB 或更大的可用空间，以便保存所有虚拟机的映像。在生产系统中，建议采用快速的共享存储。NFS 是用来在hHypervisor间共享存储的最常用协议。在试验环境中，可以使用基于 Linux 的 NFS 服务器来代替专用存储。尽管hHypervisor中的本地磁盘在POC 测试型环境中可以使用，但通常不建议将本地磁盘用于生产系统中。

我们最后应规划的是网络。在实验室环境中，我们可以直接将所有节点都放在同一网络中。不过，在生产系统中，出于安全和管理需要，应将 Cloud Foundry 的各个组件正确分配到VLAN中。在本系列文章中，我们不讨论网络连接方面的细节。作为例子，我们在部署期间将采用四个VLAN：

|VLAN      |节点       |
|----------|-----------|
|Management VLAN |hHypervisor和 NFS 存储|
|CF VLAN |BOSH 虚拟机以及 Cloud Foundry 的虚拟机|
|Service VLAN |LB ，双宿 (Dual-Homed) router|
|Public VLAN |LB，外网请求|

Cloud Foundry 实例的安装过程分为以下四个部分：

		1) 在 Ubuntu 10.04 操作系统中安装 BOSH CLI 工具。此操作系统的主机可以是物理机，也可以是虚拟机。
		2) 安装Micro BOSH。Micro BOSH是一个包含 BOSH 所有组件的虚拟机。它所具备标准 BOSH 的所有功能。不过，它用来存储多个Release的磁盘空间十分有限。部署Micro BOSH的目的是为了安装 BOSH，因为BOSH 本身就是一个分布式系统。
		3) 通过Micro BOSH来安装 BOSH。BOSH 通常包含 6 个结点，每个节点部署一个组件。其中一个称作blobstore的节点具有较大的磁盘，可以保存较大的Release。
		4) 通过 BOSH 安装 Cloud Foundry 实例。

##第 II 部分##

###安装 BOSH CLI###

###重要前提条件：###
在 BOSH 和 Cloud Foundry 的整个安装过程中，都需要直接的 Internet 连接。这一点非常重要，因为部分软件代码是直接从 Internet 下载的，例如 Ruby gGem 以及一些开源软件。在虚拟机与 Internet 之间设置 Web 代理服务器将导致安装失败。注：NAT是允许的。

另一项前提条件是要有稳定的 Internet 连接。如果您的网络在从 Internet 下载文件时速度缓慢或者不可靠，安装可能会因出现超时或连接错误而失败。

我们遇到过很多因为 Internet 连接问题而安装失败的情况。强烈建议您先咨询网络管理员，再开始安装 BOSH。

###在 vCenter 中创建一个群集###

假定所有节点都是虚拟机，那么我们首先在所有裸机服务器上安装 vSphere（在本篇文章中我们采用 V5.x）。各 vSphere 服务器通过Management VLAN相连。安装完毕后，我们需在其中一个Hhypervisor上创建一个虚拟机以安装 64 位 Windows 2008 R2。随后，我们需在此 Windows 2008 虚拟机上安装 vCenter。下一步是使用 vSphere Client 连接到 vCenter，以便我们可以管理这些服务器。

有关vSphere，vCenter和vSphere Client的安装使用细节，请参考VMware官方网站中的文档介绍http://www.vmware.com/cn/products/

我们可以在任意 Windows机器（甚至是虚拟机）上安装 vSphere Client。之后，我们便可以通过 vSphere Client 以远程方式连接到 vCenter。首先，我们来在 vCenter 中创建一个数据中心。为此，请右键单击左窗格中的 vCenter 节点，然后选择““新建数据中心””(（New Datacenter)）以添加一个新的数据中心。

接下来，请右键单击新创建的数据中心节点，然后选择“新建群集...”(New Cluster...)。

在“新建群集向导”(New Cluster Wizard) 执行期间，如果您启用了 vSphere DRS 功能，系统将要求您配置 VMware DRS。请确保“自动化级别”(Automation Level) 设置为“半自动”(Partially Automated) 或“全自动”(Fully Automated)，如下所示。如果您选择“手动”(Manual)，将会弹出一个窗口提示您输入您的选择。这种行为可能会阻止 BOSH 自动化安装。

然后，请选中“启用主机监控”(Enable Host Monitoring) 复选框，再选中“禁用:启动违反可用性限制的虚拟机”(Disable: Power on VMs that violate availability constraints)：

接着，转到“虚拟机监控”(VM Monitoring) 子部分，选择“已禁用”(Disabled)：

单击“下一步”(Next)，按如下所示做出选择：

###添加 vSphere 主机###

下一步是将hypervisor放入我们刚创建的群集中。为此，请右键单击该群集节点，然后选择“添加主机...”(Add Host...)。对于每台 vSphere 服务器，输入其 IP 地址、管理员用户名和密码，然后确认您进行的配置：

添加完所有主机后，这些主机将在“数据中心”(Datacenter) ->“主机”(Hosts) 选项卡中列出：

###将数据存储挂接到主机###

该群集中的所有主机都应共享同一 NFS 存储。对于每个主机，我们将该存储以数据存储的形式添加进来。为此，请在 vCenter 中单击相应的 vSphere 主机，然后选择“配置”(Configuration) 选项卡。选择“硬件”(Hardware) ->“存储”(Storage)。单击右上方的“Add Storage...”(添加存储...) 

在“选择存储类型”(Select Storage Type) 对话框中，选择“网络文件系统”(Network File System)。

输入 NFS 存储的 IP 地址、相应的文件夹名称以及相应的数据存储名称。请务必对该群集内的所有主机都采用完全相同的数据存储名称，这一点非常重要。在本例中，我们采用“NFSdatastore”这一名称。

###为虚拟机和模板创建文件夹###

从 vCenter 的导航栏中，选择“主页”(Home) ->“清单”(Inventory) ->“虚拟机和模板”(VMs and Templates) 视图，然后按如下所示创建文件夹：

这些文件夹随后将用来对 BOSH 和 Cloud Foundry 的虚拟机进行分组。在上例中，“template_folder_bosh”用来存放 BOSH stemcell。“vm_folder_bosh”用来存放 BOSH 节点。“template_folder”用来存放 Cloud Foundry stemcell。“vm_folder”用来存放 Cloud Foundry 节点。随后将在部署清单文件中用到这些名称。

###网络配置###

Cloud Foundry 的虚拟机将部署到一个或多个网络中。在部署前，我们需要在 vSphere 中创建一些网络。下图显示了 Cloud Foundry 所需的网络连接。

在每个 vSphere 主机上，我们创建以下两个网络：

		1)	CF Network：映射到 CF VLAN
		2)	Service Network：映射到Service VLAN。

大多数虚拟机都位于 CF Network上。只有router虚拟机是双宿在Service Network和 CF Network上。

注意：在试验环境中，您可以将所有虚拟机都放在同一网络上，以便简化安装过程。因此，hHypervisor上的只有1个网络可能就绰绰有余。

要创建网络，请选择“主机和群集”(Hosts and Clusters) 视图。选择一个主机，然后切换到“配置”(Configuration) 选项卡。然后选择“网络”(Networking)，再单击“添加网络”(Add Networking)：

连接类型应为“虚拟机”(Virtual Machine)：

请使用现有的虚拟交换机：

在下一步中，将网络标签重命名为“CF Network”。如果网络管理员已经指定了VLAN ID，请相应地输入 CF VLAN ID。

然后单击“完成”(Finish)，这样便完成了网络创建。重复上述步骤创建“Service Network”，直接将网络标签命名为“Service Network”即可。
务必要让同一群集内所有主机上的网络名称都保持完全相同。下图显示已经为某个主机创建了两个网络。我们将这两个网络分别命名为“CF Network”和“Service Network”。稍后在 BOSH 和 Cloud Foundry 的 YML 文件中将会用到这两个名称。

此外，如果您从“数据中心”(Datacenter) ->“清单”(Inventory) ->“网络”(Networking) 视图中进行查看的话，显示如下：

###为 BOSH CLI 创建一个虚拟机###

从 vCenter 中，我们选择该群集中的主机之一来创建一个虚拟机。为此，请单击“创建新虚拟机”(Create a new virtual machine)。我们将在此虚拟机上安装 64 位 Ubuntu 10.04 操作系统。为此虚拟机分配 2 个虚拟 CPU、2 GB 内存、20 GB 磁盘空间（或者更多）。在安装期间，一定要手动设置网络。

我们现在开始在该虚拟机上安装 BOSH CLI。为此，请登录到 Ubuntu，然后遵照以下步骤操作。（请注意，这些步骤大都摘自 BOSH 官方文档。为了您方便起见，在此将它们列出来。）
1.通过 rbenv 安装 Ruby：
		1.BOSH 是用 Ruby 编写的。我们来安装 Ruby 的依赖项：
		`$ sudo apt-get install git-core build-essential libsqlite3-dev curl libmysqlclient-dev libxml2-dev libxslt-devlibpq-devgenisoimage`
		2.获取最新版本的rbenv
		`$ git clone git://github.com/sstephenson/rbenv.git .rbenv`
		3.将 ~/.rbenv/bin 添加到您的 $PATH 以便能够访问rbenv命令行实用程序 
		`$ echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile`
		4.将rbenvinit添加到您的 shell 以启用填充程序 (Shim) 和自动完成
		`$ echo 'eval "$(rbenvinit -)"' >> ~/.bash_profile`
		5.下载 Ruby 1.9.2
		注意：您也可以使用适用于 rbenv 的 ruby-build 插件来构建 ruby。请参见https://github.com/sstephenson/ruby-build
		`$ wgethttp://ftp.ruby-lang.org/pub/ruby/1.9/ruby-1.9.2-p290.tar.gz`
		6.将 Ruby 解包并安装
		`$ tar xvfz ruby-1.9.2-p290.tar.gz`
		`$ cd ruby-1.9.2-p290`
		`$ ./configure --prefix=$HOME/.rbenv/versions/1.9.2-p290`
		`$ make`
		`$ make install`
		7.重新启动您的 shell 以使路径更改生效
		`$ source ~/.bash_profile`
		8.将您的默认 Ruby 设置为 1.9.2 版本
		`$ rbenv global 1.9.2-p290`
		注意：使用此方法时可能需要重新安装 rake 0.8.7 gem
		`$ gem pristine rake`
		9.更新rubygem并安装捆绑包。
		注意：安装 gem（gem install 或 bundle install）后，请运行rbenv rehash 以添加新的填充程序
		`$ rbenv rehash`
		`$ gem update –system`
		`$ gem install bundler`
		`$ rbenv rehash`

1.安装 BOSH CLI：
		1.在http://reviews.cloudfoundry.org注册 Cloud Foundry Gerrit服务器
		2.设置您的ssh公钥（接受所有默认值）：
		`$ ssh-keygen –t rsa`
		3.将您的密钥从 ~/.ssh/id_rsa.pub 复制到您的Gerrit帐户中。
		4.在您的Gerrit帐户配置文件中创建并上传自己的 SSH 公钥。
		5.设置您的姓名和电子邮件：
		`$ git config --global user.name "FirstnameLastname"`
		`$ gitconfig --global user.emailyour_email@youremail.com`
		6.安装 gerrit-cli gem：
		`$ gem install gerrit-cli`
		7.运行一些 rake 任务来安装 BOSH CLI：
		`$ gem install bosh_cli`
		`$ rbenv rehash`
		`$ bosh –version`

如果一切运行顺利，最后一个命令将会显示您刚刚创建的 BOSH 版本。这表明 BOSH CLI 已安装成功。

##第 III 部分##

###安装Micro BOSH和 BOSH（deploy/vSphere-BOSH.MD）###

BOSH CLI 安装完毕后，我们现在便开始安装Micro BOSH。如前文所述，可以将Micro BOSH视作袖珍版的 BOSH。尽管标准的 BOSH各个组件分布在 6 个虚拟机上 ，但Micro BOSH却恰恰相反，它在单个虚拟机中包含了所有组件。它可以轻易的设置，通常用于部署小型的Release，如 BOSH。从这个意义上讲，BOSH 是自部署的。用 BOSH 团队的话说，这叫做“Inception”。

以下步骤是通过在 BOSH 官方文档基础上添加更多操作细节改编成的。
1.在 BOSH CLI 虚拟机中，安装 BOSH 部署器 ruby gem。

`$ gem install bosh_deployer`

一旦安装了该部署器，那么您在命令行中键入 bosh 后将会看到一些额外的命令显示出来。

`$ bosh help`

		...
    		Micro
        	micro deployment [<name>]	选择要使用的微部署
        	micro status			显示Micro BOSH部署的状态
       	 	micro deployments		显示部署列表
        	micro deploy <stemcell>	将Micro BOSH实例部署到当前选择的部署
                        --update		更新现有实例
        	micro delete			删除Micro BOSH实例（包括持久磁盘）
        	micro agent <args>		发送代理消息
        	micro apply <spec>		应用规范

注意：上述 bosh micro 命令必须在Micro BOSH部署目录中运行

2.在 vCenter 中的“主页”(Home) ->“清单”(Inventory) ->“虚拟机和模板”(VMs and Templates) 视图下，确保用来存放虚拟机和模板的文件夹已经创建（见第 II 部分）。这些文件夹将在部署配置中使用。

3.从“主页”(Home) ->“清单”(Inventory) ->“数据存储”(Datastores) 视图中，选择我们创建的NFSdatastore数据存储，并浏览该存储。

右键单击根文件夹，创建一个用来存储虚拟机的子文件夹。在本例中，我们将此子文件夹命名为“boshdeployer”。此文件夹名称将成为我们部署清单中的“disk_path”参数的值。
 
注意：如果您没有共享的 NFS 存储，您可以使用hypervisor的本地磁盘作为数据存储。（注意: 仅建议在试验系统采用本地磁盘。）您可以按如下方式命名这些数据存储：主机 1 的数据存储命名为“localstore1”、主机 2 的数据存储命名为“localstore2”，依此类推。随后在清单文件中，您可以使用诸如“localstore*”之类的通配符模式来指定所有主机的数据存储。应在所有本地数据存储上都创建“boshdeployer”文件夹。

4.下载公共 stemcell  

`$ mkdir -p ~/stemcells`

`$ cd stemcells`

`$ bosh public stemcells`

输出大致如下 ：

	+---------------------------------+-------------------------------------------------------+
	| Name                            | Url                                                   |
	+---------------------------------+-------------------------------------------------------+
	| bosh-stemcell-0.5.2.tgz         | https://blob.cfblob.com/rest/objects/4e4e78bca31e1... |
	| bosh-stemcell-aws-0.5.1.tgz     | https://blob.cfblob.com/rest/objects/4e4e78bca21e1... |
	| bosh-stemcell-vsphere-0.6.4.tgz | https://blob.cfblob.com/rest/objects/4e4e78bca31e1... |
	| micro-bosh-stemcell-0.1.0.tgz   | https://blob.cfblob.com/rest/objects/4e4e78bca51e1... |
	+---------------------------------+-------------------------------------------------------+
	To download use 'bosh download public stemcell<stemcell_name>'.For full url use --full.

使用下面的命令下载Micro BOSH的 stemcell：

`$ bosh download public stemcell micro-bosh-stemcell-0.1.0.tgz`

注意：此stemcell的大小为 400 至 500 MB。在速度缓慢的网络中可能需要耗费很长时间才能将它下载下来。在这种情况下，您可以使用任何能在出断点续传的工具（如 Firefox 浏览器）进行下载。使用 --full 参数可显示完整的下载 URL。

5.配置部署 (.yml) 文件，然后将它保存在micro1文件夹下，该文件夹名称在 .yml文件中定义。

```
$ cd ~
    $ mkdir deployments
    $ cd deployments 
    $ mkdir micro01
```

在 yml 文件中，有一节内容是关于 vCenter 的。请在此节中输入我们在第 II 部分中创建的文件夹的名称。“disk_path”应为我们刚刚在数据存储 (NFSdatastore) 中创建的文件夹。datastore_pattern 和persistent_datastore_pattern 的值是共享数据存储的名称 (NFSdatastore)。如果您采用本地磁盘，此值可以是像“localstore*”这样的通配字符串。

		datacenters:
		- name:vDataCenter	
		vm_folder:vm_folder	
		template_folder:template	
		disk_path:boshdeployer		
		datastore_pattern:NFSdatastore								
		persistent_datastore_pattern:NFSdatastore		
		allow_mixed_datastores:true 

下面是Micro BOSH的一个示例yml文件的链接：

https://github.com/vmware-china-se/bosh_doc/blob/master/micro.yml

6.使用以下命令设置此Micro BOSH部署：

```
    $ cd deployments 
$ bosh micro deployment micro01
```

部署设置为“~/deployments/micro01/micro_bosh.yml”	部署设置为“~/deployments/micro01/micro_bosh.yml”

`$ bosh micro deploy ~/stemcells/micro-bosh-stemcell-0.1.0.tgz`

如果一切都运行顺利，Micro BOSH将在几分钟内部署完毕。您可以通过下面的命令查看部署状态：

`$ bosh micro deployments`

您将会看到您的Micro BOSH部署已列出 ：

		+---------+-----------------------------------------+-----------------------------------------+
		| Name    | VM name                                 | Stemcell name                           |
		+---------+-----------------------------------------+-----------------------------------------+
		| micro01 | vm-a51a9ba4-8e8f-4b69-ace2-8f2d190cb5c3 | sc-689a8c4e-63a6-421b-ba1a-			  400287d8d805 |
		+---------+-----------------------------------------+-----------------------------------------+

###安装 BOSH###

Micro BOSH准备就绪后，我们就可用它来部署 BOSH。BOSH 是一个包含 6 个虚机的分布式系统。正如上一节所提到的那样，我们需要有三项内容：一个作为虚拟机模板的stemcell、一个作为待部署软件的 BOSH Release，以及一个用来定义部署配置的部署清单文件。我们来逐一准备。

1.首先，我们将 BOSH CLI 的目标设为Micro BOSH的dDirector。可以将 BOSH diDirector视作 BOSH 的控制者或协调者。所有 BOSH CLI 命令均发往该dDirector加以执行。该dDirector的 IP 地址在我们用来创建Micro BOSH的 yml 文件中定义。BOSH dDirector的默认用户/密码为 admin/admin。在我们的示例中，我们使用下面的命令来设定Micro BOSH的目标和进行身份验证：

```
 $ bosh target 10.60.98.124:25555
$ bosh login
```

1.接下来，我们下载 BOSH stemcell并将其上传到Micro BOSH。这一步与下载Micro BOSH的 stemcell 类似。唯一的差别在于，我们选择的是 BOSH 而非Micro BOSH的stemcell。

```
$ cd ~/stemcells
$ bosh public stemcells
```

随即便会显示一列stemcell；请选择最新的stemcell进行下载：

```
$ bosh download public stemcell bosh-stemcell-vsphere-0.6.4.tgz
    $ bosh upload stemcell bosh-stemcell-vsphere-0.6.4.tgz
```

如果您已在第 II 部分中创建了Gerrit帐户，请跳过第 3 步至第 7 步。

1.在以下位置注册 Cloud Foundry Gerrit服务器：http://reviews.cloudfoundry.org

1.设置您的 ssh 公钥（接受所有默认值）

`$ ssh-keygen -t rsa`

将您的密钥从 ~/.ssh/id_rsa.pub 复制到您的Gerrit帐户中

1.在您的Gerrit帐户配置文件中创建并上传自己的 SSH 公钥

1.设置您的姓名和电子邮件

```
$ git config --global user.name "FirstnameLastname"
$ gitconfig --global user.emailyour_email@youremail.com
```

1.安装gerrit-cli gem

1.使用Gerrit从 Cloud Foundry 代码库中克隆Release代码。以下命令分别获取 BOSH 和 Cloud Foundry 的代码。

```
$ gerrit clone ssh://<yourusername>@reviews.cloudfoundry.org:29418/bosh.git 
    $ gerrit clone ssh://<your username>@reviews.cloudfoundry.org:29418/cf-release.git
```

然后我们创建我们自己的 BOSH Release：

```
$ cd bosh-release
$ ./update 
$ bosh create release  --with-tarball
```

如果存在本地代码冲突，您可以添加“--force”选项：

`$ bosh create release  --with-tarball --force`

这一步可能需要一些时间才能完成，具体取决于您的网络速度。它首先会从一个 Blob 服务器下载二进制文件。然后它会构建包并生成清单文件。该命令的输出大致如下：

	         Syncing blobs…
	          ...
	          Building DEV release
	          Please enter development release name:bosh-dev1
	           ---------------------------------
	          Building packages
	          …
	          Generating manifest...
	          …
	          Copying jobs...

最后，Release创建完毕后，您将看到大致如下的内容。请注意，最后两行指出了清单文件和Release文件。

		Generated /home/boshcli/bosh-release/dev_releases/bosh-dev1-6.1-dev.tgz
		Release summary
		---------------
		Packages
		+----------------+---------+-------+------------------------------------------+
		| Name           | Version | Notes | Fingerprint                              |
		+----------------+---------+-------+------------------------------------------+
		| nginx          | 1       |       | 1f01e008099b9baa06d9a4ad1fcccc55840cf56e |
		……
		| ruby           | 1       |       | c79b76fcb9bdda122ad2c52c3604ba950b482681 |
		+----------------+---------+-------+------------------------------------------+

		Jobs
		+----------------+---------+-------------+------------------------------------------+
		| Name           | Version | Notes       | Fingerprint                              |
		+----------------+---------+-------------+------------------------------------------+
		| micro_aws      | 1.1-dev | new version | fadbedb276f918beba514c69c9e77978eadb65ac |
		……
		| redis          | 2       |             | 3d5767e5deec579dee61f2c131982a97975d405e |
		+----------------+---------+-------------+------------------------------------------+
		Release version:6.1-dev
		Release manifest:/home/boshcli/bosh-release/dev_releases/bosh-dev1-6.1-dev.yml
		Release tarball (88.8M):/home/boshcli/bosh-release/dev_releases/bosh-dev1-6.1-dev.tgz

1.将创建好的Release上传到Micro BOSH的director。

`$ bosh upload release dev_releases/bosh-dev1-6.1-dev.tgz `

1.配置 BOSH 部署清单。首先，我们通过执行以下命令获取该director的 UUID 信息：

`$ bosh status`

	Updating director data... done
	Target         micro01 (http://10.60.98.124:25555) Ver: 0.4 (00000000)
	UUID           7d72eb71-9a98-4081-9857-ad7c7ff4ee33
	User           admin
	Deployment     /home/boshcli/bosh-dev1.yml

现在我们将进入此安装过程中最为复杂的环节：修改部署清单文件。由于大多数 BOSH 部署错误都是因为清单文件中的设置不正确而造成的，因此我们详细地对此环节进行说明。

首先，我们从以下位置获取清单模板：https://github.com/cloudfoundry/oss-docs/blob/master/bosh/tutorial/examples/bosh_manifest.yml

由于 BOSH 官方文档提供了该清单文件的规范，因此我们假定您在阅读本篇文章前，已经通读了此文档。我们将不会对该文件进行全面的介绍；而是讨论该清单文件中的一些重要项目。

**网络（networks）**

下面是网络一节的一个示例。

		networks:		#定义网络
		- name:default
		  subnets:
		  - reserved:	#您不希望分配的 IP
		    - 10.60.98.121 - 10.60.98.254
		    static:		#您将使用的 IP
		    - 10.60.98.115 - 10.60.98.120
		    range: 10.60.98.0/24
		    gateway: 10.60.98.1
		dns:
		    - 10.40.62.11
		    - 10.135.12.101
		cloud_properties:#与所有其他虚拟机都相同的网络。
		      name:VM Network

static：包含 BOSH 虚拟机的 IP 地址。

reserved：BOSH 不应使用的 IP 地址。请务必要排除所有已经分配给同一网络中其他设备（例如存储设备、网络设备、Micro BOSH和 vCenter 主机）的 IP 地址。在安装期间，Micro BOSH可能会创建一些临时虚拟机（工作者虚拟机）来执行编译。如果我们不指定预留的地址，这些临时虚拟机可能会与现有设备或主机存在 IP 地址冲突。

cloud_properties：name 是我们在 vSphere 中定义的网络名称（见第 II 部分）。

**Resource Pool (资源池)（resource_pools）**

此节定义作业使用的虚拟机配置（CPU、内存、磁盘和网络）。通常，应用程序的各个作业在资源使用方面各异。例如，有些作业需要较多的内存量，而有些作业则需要更多的vCPU来执行计算密集型任务。根据实际需要，我们应创建一个或多个资源池。需要注意的是，所有池的总规模应等于在清单文件中定义的作业实例的总数。部署 BOSH 时，由于总共有 6 个虚拟机（6 个作业），因此所有池的规模加起来应等于 6。

在我们的清单文件中，我们有 3 个资源池：

|Resource Pool|	Size   |VM Configuration   |Job  |
|-------------|--------|-------------------|-----|
|small	|3	|RAM：512 MB；CPU：1 个；磁盘：2 GB	|nats、redis、health_monitor|
|medium	|2	|RAM：1 GB；CPU：1 个；磁盘：8 GB	|postgres、blobstore|
|director |1	|RAM：2 GB；CPU：2 个；磁盘：8 GB	|director|

**编译（compilation）**

此节定义为编译包而创建的工作者虚拟机。在资源有限的系统中，我们应减少并发工作者虚拟机的数目，以确保编译成功。在我们的示例中，我们定义 4 个工作者虚拟机。

**更新（update）**

此节包含一个非常有用的参数：max_in_flight。此参数用于向 BOSH 告知最多可以并行安装的作业数。在运行速度缓慢的系统中，请设法减小此数目。如果将此数目设置为 1，则表示作业将按顺序部署。对于 BOSH 部署，我们建议将此数目设置为 1，以确保 BOSH 可以成功安装。

**作业（jobs）**

在 BOSH Release中有六个作业。每个作业占用一个虚拟机。根据作业的性质和资源使用情况，我们将作业分配给各个资源池。需要注意的一点是，我们需要向以下三个作业分配持久磁盘：postgres、director 和blobstore。若无持久磁盘，这些作业将无法正常运行，因为它们的本地磁盘很快就会被占满。

最好填写一张像下面这样的电子表格来对您的部署进行规划。您可以根据该电子表格修改部署清单。

|作业	|资源池	|IP|
|-------|-------|--|
|nats	|small	|10.60.98.120|
|postgres	|medium	|10.60.98.119|
|redis	|small	|10.60.98.118|
|director	|director	|10.60.98.117|
|blob_store	|medium	|10.60.98.116|
|health_monitor	|small	|10.60.98.115|

我们基于上表创建了一个示例部署清单，您可以从这里下载：

https://github.com/vmware-china-se/bosh_doc/blob/master/bosh.yml

1.更新完部署清单文件后，我们便可以通过运行以下命令开始实际部署：

```
$ bosh deployment bosh_dev1.yml
$ bosh deploy 
```

此过程可能需要等待一段时间才能完成，具体取决于您的网络条件以及可用的硬件资源。您也可以在 vCenter 控制台查看创建、配置并销毁虚拟机的过程。

		Preparing deployment
		…..
		Compiling packages
		……
		Binding instance VMs
		  postgres/0 (00:00:01)                                                                            
		  director/0 (00:00:01)                                                                            
		  redis/0 (00:00:01)                                                                               
		  blobstore/0 (00:00:01)                                                                           
		  nats/0 (00:00:01)                                                                                
		  health_monitor/0 (00:00:01)                                                                      
		Done                    6/6 00:00:01                                                               

		Updating job nats
		  nats/0 (canary) (00:01:14)                                                                       
		Done                    1/1 00:01:14
		……
		Updating job director
		  director/0 (canary) (00:01:10)                                                                   
		Done                    1/1 00:01:10   
		……

如果一切运行顺利，您最终将会看到大致如下的结果：

		Task 14 done
		Started		2012-08-12 03:32:24 UTC
		Finished	2012-08-12 03:52:24 UTC
		Duration	00:20:00
		Deployed `bosh-dev1.yml` to `micro01`
    
这表示您已成功部署 BOSH。您可以通过执行下面的命令来查看您的部署：

$ bosh deployments

		+-------+
		| Name  |
		+-------+
		| bosh1 |
		+-------+

您可以通过执行下面的命令来查看所有虚拟机的状态：

`$ boshvms`

如果一切都未出问题的话，您将看到大致如下的虚拟机状态：

		+------------------+---------+---------------+--------------+
		| Job/index        | State   | Resource Pool | IPs          |
		+------------------+---------+---------------+--------------+
		| blobstore/0      | running | medium        | 10.60.98.116 |
		| director/0       | running | director      | 10.60.98.117 |
		| health_monitor/0 | running | small         | 10.60.98.115 |
		| nats/0           | running | small         | 10.60.98.120 |
		| postgres/0       | running | medium        | 10.60.98.119 |
		| redis/0          | running | small         | 10.60.98.118 |
		+------------------+---------+---------------+--------------+
		VMs total: 6

##第 IV 部分##

###安装部署 Cloud Foundry（/deploy/vSphere-CF.MD）###

在前面的文章中，我们安装了Micro BOSH和 BOSH。如果一切顺利，我们准备已经为好安装 Cloud Foundry做好准备了。首先，我们为Cloud Foundry的部署制定资源计划。

在我们编写本文时，完整的 Cloud Foundry 安装包含大约 34 个不同作业（虚拟机）。其中有些作业为核心组件，必须安装至少一个这类作业的实例；例如，Cloud Controller、NATS 和 DEA 等。有些作业应具有多个实例，具体取决于实际需求；例如 DEA 和router等。有些作业是可选的，例如服务网关和服务节点。因此，我们在安装 Cloud Foundry 前，应决定将哪些组件纳入部署范围。我们制定了要部署的组件的清单后，便可以规划每个作业所需的资源。通常，这些资源包括 IP 地址、CPU、内存和存储。下面是一个部署计划示例。

|作业	|实例数	|IP	|内存	|CPU	|磁盘 (GB)	|是否为必需的？|
|-------|-------|-------|-------|-------|---------------|--------------|
|debian_nfs_server	|1	|xx.xx.xx.xx	|2 GB	|2	|16	|必需|
|nats	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|必需|
|ccdb_postgres	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|必需|
|uaadb	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|必需|
|vcap_redis	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|必需|
|uaa	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|必需|
|acmdb	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|必需|
|acm	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|必需|
|cloud_controller	|1	|xx.xx.xx.xx	|2 GB	|2	|16	|必需|
|stager	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|必需|
|router	|2	|xx.xx.xx.xx	|512 MB	|1	|8	|必需|
|health_manager	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|必需|
|dea	|2	|xx.xx.xx.xx	|2 GB	|2	|16	|必需|
|mysql_node	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|mysql_gateway	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|mongodb_node	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|mongodb_gateway	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|redis_node	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|redis_gateway	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|rabbit_node	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|rabbit_gateway	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|postgresql_node	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|postgresql_gateway	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|vblob_node	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|vblob_gateway	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|backup_manager	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|service_utilities	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|serialization_data_server	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|services_nfs	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|syslog_aggregator	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|services_redis	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|opentsdb	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|collector	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|dashboard	|1	|xx.xx.xx.xx	|1 GB	|1	|8	|可选|
|-------|-------|-------|-------|-------|---------------|--------------|
|合计：	|36	|	|39 GB	|40	|320	| |

根据上表，我们便可以确定所需的资源池：

|池名称	|规模	|配置	|作业|
|-------|-------|-------|----|
|small	|30	|RAM：1 GB；CPU：1 个；磁盘：8 GB	|nats、ccdb_postgres、uaadb、vcap_redis、uaa、acmdb、acm、stager、health_manager、mysql_node、mysql_gateway、mongodb_node、mongodb_gateway、redis_node、redis_gateway、postgresql_node、postgresql_gateway、vblob_node、vblob_gateway、backup_manager、service_utilities、serialization_data_server、services_nfs、syslog_aggregator、services_redis、opentsdb、collector、dashboard|
|medium	|4	|RAM：2 GB；CPU：2 个；磁盘：16 GB	|debian_nfs_server、cloud_controller、dea|
|router	|2	|RAM：512 M；CPU：1 个；磁盘：8 GB	|router|

根据上面两个表，我们可以修改清单文件。

我们将清单文件命名为cf.yml。以下各节详细说明了其中的字段。 完整的Cloud Foundry部署yml文件，请参考：https://github.com/vmware-china-se/bosh_doc/blob/master/cf.yml（注，此yml文件是用来为Cloud Foundry的BOSH部署提供配置信息，与之前介绍的BOSH yml文件不同）

**name**

这是 Cloud Foundry 部署名。我们可以随意对它命名。

**director_uuid**

director UUID 是我们刚刚在第 III 部分中部署的 BOSH dDirector的 UUID。我们可以通过下面的命令来检索此值：

`$ bosh status`

**release**

此Release名称应与您在创建 CF Release时输入的名称相同。版本是在创建Release时自动生成的。

**compilation、update、networks、resource_pools**

这些字段与 bosh.yml 文件中的那些字段类似。有关更多信息，请参考上一部分。

**jobs**

作业是 Cloud Foundry 的组件。每个作业在一个虚拟机上运行。各个作业的说明如下。

|debian_nfs_server、services_nfs	|这两个作业在 Cloud Foundry 中用作 NFS 服务器。由于它们是文件服务器，因此我们应确保“persistent_disk”属性确实存在。|
|syslog_aggregator	|此作业用于收集系统日志并将它们存储在数据库中。|
|nats	|NATS 是 Cloud Foundry 的消息总线。它是 Cloud Foundry 中的核心组件之一。|
|opentsdb	|这是用来存储日志信息的数据库。由于它是数据库，因此它也需要“persistent_disk”属性。|
|collector	|此作业用于收集系统信息并将它们存储在数据库中。|
|dashboard	|这是基于 Web 的控制台工具，用来监视和报告 Cloud Foundry 平台的情况。|
|cloud_controller、ccdb	|cloud_controller负责控制 Cloud Foundry 的所有组件。“ccdb”是Cloud Controller的数据库。“persistent_disk”属性在ccdb中是必需的。|
|uaa、uaadb	|uaa用于进行用户身份验证和授权。uaadb是用来存储用户信息的数据库。“persistent_disk”属性对uaadb而言是必需的。|
|vcap_redis、services_redis	|这两个作业用于存储 Cloud Foundry 的内部键值对。|
|acm、acmdb	|acm是访问控制管理器 (Access Control Manager) 的简写形式。ACM 是一项服务，借助这项服务，Cloud Foundry 组件可以实现访问控制功能。“acmdb”是acm的数据库。“acmdb”也需要“persistent_disk”属性。|
|stager	|stager 这个作业负责将用户应用程序的源代码及所有必需包打包。暂存完成后，便会将该应用程序传递给dea加以执行。|
|router	|用于将用户的请求路由到 Cloud Foundry 中的正确目标位置。|
|health_manager、health_manager_next	|health_manager这个作业负责监视所有用户的应用程序的运行状况。health_manager_next是health_manager的下一代版本。它们将共存一些时日。|
|dea	|“dea”是 Droplet Execution Agent 的简写形式。所有用户的应用程序都在dea中执行。|
|mysql_node、mysql_gateway、mongodb_node、mongodb_gateway、redis_node、redis_gateway、rabbit_node、rabbit_gateway、postgresql_node、postgresql_gateway、vblob_node、vblob_gateway	|这些作业全都是给 Cloud Foundry提供服务的。每项服务都有一个负责置备资源的节点。对应的网关位于cloud_controller与服务节点之间，担当每项服务的管理功能。|
|backup_manager	|用于备份用户的数据和数据库。|
|service_utilities	|服务管理实用程序。|
|serialization_data_server	|用于在 Cloud Foundry 中对数据进行序列化的服务器。|

**properties：**

这是 cf.yml 文件中的另一个重要部分。我们应注意，此节中的 IP 地址应与 jobs 字段中的那些地址要一致。您应将此节中的密码和令牌替换成您自己的安全密码和令牌。

domain：这是供用户访问的域的名称。我们还应创建一个 DNS 服务器来将该域解析为负载均衡器的 IP 地址。在我们的示例中，我们将域名设置为cf.local，以便用户在推送应用程序时可以使用vmc target api.cf.local。

cc.srv_api_uri：此属性通常采用以下格式：http://api.<您的域名>。例如，如果我们将域设置为cf.local，那么srv_api_uri将为 http://api.cf.local。

cc.password：此密码必须包含至少 16 个字符。

cc. allow_registration：如果它为 True，则用户可以使用vmc命令注册帐户。将它设置为 False 则禁止此行为。

cc.admins：管理员用户的名单。即使allow_registration标志设置为 False，管理员用户也可以通过vmc命令进行注册。

这些属性中的大多数“nfs_server”都应设置为“services_nfs”作业的 IP 地址。

mysql_node.production：如果它为 True，则mysql_node的内存必须至少为 4 GB。在试验环境中，我们可以将它设置为 False，这样mysql_node的内存便可以设置为小于 4 GB。

由于yml文件可能会随着 Cloud Foundry 新Release的问世而发生演变，因此提供了使用 BOSH 命令验证yml文件的选项。键入“bosh help”后，您便可以看到“bosh diff”的用法和解释：

`$ bosh diff [<template_file>]`

此命令会将您当前的部署清单与指定的部署清单模板进行对比。它可更新部署配置文件。最新的开发模板可以在deployment repo 中找到。

例如，您可以运行下面的命令来将yml文件与模板文件进行对比。首先，您必须切换到您的 cf.yml 文件和模板文件所在的目录，然后运行下面的命令：

`$ bosh diff dev-template.erb`

此命令将显示 cf.yml 文件中的错误。如果缺少某些字段，此命令将自动帮您填入这些字段。如果有拼写错误或其他错误，此命令将报告存在语法错误。

您可以从以下位置下载部署Cloud Foundry的示例yml文件：

https://github.com/vmware-china-se/bosh_doc/blob/master/cf.yml

此清单文件完成后，我们就可以开始安装 Cloud Foundry 了。

1) 在之前的步骤中第 III 部分中，我们已经通过以下命令从Gerrit克隆了 CF 代码库：

`$ gerrit clone ssh://<your username>@reviews.cloudfoundry.org:29418/cf-release.git`

2) 切换到上述目录，然后创建一个 CF Release。

```
$ cd cf-release
$ bosh create release 
```

这将下载部署所需的所有包、blob 数据及其他资源。下载过程将耗费若干分钟，主要取决于网络速度。

**注意：**
1.如果您编辑了 CF Release中的代码，那么您可能需要在命令 bosh create release 中添加 --force 选项。

2.在运行此命令时系统一定要直接连接Internet。

3.如果您的网络速度慢或者您未与 Internet 建立直接连接，那么您可能需要在一个更好的环境中完成创建Release的操作。您可以在有良好 Internet 连接的机器上使用--with-tarball选项创建此Release。然后，您需要将所生成的 tar 包复制到所需系统。

如果一切都未出问题的话，您可以看到此Release的摘要，大致如下：

		Generating manifest...
		----------------------
		Writing manifest...
		Release summary
		---------------
		Packages
		+---------------------------+---------+-------+------------------------------------------+
		| Name                      | Version | Notes | Fingerprint                              |
		+---------------------------+---------+-------+------------------------------------------+
		| sqlite                    | 3       |       | e3e9b61f8cdc2610480c2aa841e89dc0bb1dc9c9 |
		| ruby                      | 6       |       | b35a5da6c214d9a97fdd664931bf64987053ad4c |
		… …
		| debian_nfs_server         | 3       |       | c1dc860ed6ab2bee68172be996f64f9587e9ac0d |
		+---------------------------+---------+-------+------------------------------------------+
		Jobs
		+---------------------------+----------+-------------+------------------------------------------+
		| Name                      | Version  | Notes       | Fingerprint                              |
		+---------------------------+----------+-------------+------------------------------------------+
		| redis_node                | 19       |             | 61098860eaa8cfb5dc438255ebe28db74a1feabc |
		| rabbit_gateway            | 13       |             | ddcc0901ded1e45fdc4f318ed4ba8d8ccca51f7f |
		… …
		| debian_nfs_server         | 7        |             | 234fabc1a2c5dfbfd7c932882d947fed331b8295 |
		| memcached_gateway         | 4        |             | 998623d86733a633c789849721e00d85cc3ebc20 |
		Jobs affected by changes in this release
		+------------------+----------+
		| Name             | Version  |
		+------------------+----------+
		… …
		| cloud_controller | 45.1-dev |
		+------------------+----------+
		Release version:95.10-dev
		Release manifest:/home/boshcli/cf-release/dev_releases/cf-def-95.10-dev.yml

正如您可以看到的那样，dev-releases 目录包含此Release的 yml 清单文件（如果启用了 --with-tarball 选项，则还包含一个 tar 包文件）。

3) 将 BOSH CLI 的目标设为 BOSH 的director。如果您不记得该director的 IP，您可以在第 III 部分中的 BOSH 部署清单中找到它。

`$ bosh target 10.60.98.117:25555`

`Target set to 'bosh_director (http://10.60.98.117:25555) Ver:0.5.1 (release:abb3e7a4 bosh:2c69ee8c)'`

4) 通过生成的清单(manifest)文件（例如，示例中的 cf-def-95.10-dev.yml）上传 CF Release。

`$ bosh upload release cf-def-95.10-dev.yml`

这一步将复制安装包和作业，并将它们构建成一个 tar 包，然后对此Release进行验证以确保文件和依赖项正确无误。验证完毕后，它会上传Release并创建新的作业。最后，会看到Release已上传的信息：

		Task 317 done
		Started		2012-08-28 05:35:43 UTC
		Finished	2012-08-28 05:36:44 UTC
		Duration	00:01:01
		Release uploaded

您可以通过以下命令验证上传的Release：

`$ bosh releases`

您可以查看列表中所有新上传的Release：

		+--------+---------------------------------------------------------------------------+
		| Name   | Versions                                                                                  |
		+--------+----------------------------------------------------------------------------+
		| cf-def | 95.1-dev, 95.8-dev, 95.9-dev, 95.10-dev |
		| cflab  | 92.1-dev                                                                                  |
		+--------+-------------------------------------------------------------------------+

5) 在我们已经上传了Release和stemcell（就是第 III 部分中的那个stemcell）后，部署清单(deployment manifest)也已准备就绪，那么就将BOSH的部署配置设置为此部署清单吧：

`$ bosh deployment cf-dev.yml`

设置完毕：

`Deployment set to '/home/boshcli/cf-dev.yml'`

现在我们就可以部署 Cloud Foundry 了：

`$ bosh deploy`

这将为每个作业创建虚拟机、编译包并安装依赖项。此过程将耗费十几分钟到几小时，快慢取决于服务器的硬件条件。您可以看到大致如下的输出：

		Preparing deployment
		binding deployment (00:00:00)                                                                     
		binding releases (00:00:01)                                                                       
		  … …
		Preparing package compilation
		  … …
		Compiling packages
		   … …
		Preparing DNS
		binding DNS (00:00:00)    
		Creating bound missing VMs
		  … …                                                             
		Binding instance VMs
		  … …                                                                      
		Preparing configuration
		binding configuration (00:00:03)                                                                  
		  … …
		Creating job cloud_controller
		cloud_controller/0 (canary) (00:02:45)        
		  … …                                                    
		Done                    1/1 00:08:41                                                                

		Task 318 done
		Started		2012-08-28 05:37:52 UTC
		Finished	2012-08-28 05:49:43 UTC
		Duration	00:11:51
		Deployed 'cf-dev.yml' to 'bosh_director'

要查看您的部署，可以使用下面的命令：

`$ bosh deployments`

		+----------+
		| Name     |
		+----------+
		| cf.local |
		+----------+
		Deployments total: 1

您还可以验证每个虚拟机的运行状态：

`$ boshvms`

		+-----------------------------+---------+---------------+-------------+
		| Job/index                   | State   | Resource Pool | IPs         |
		+-----------------------------+---------+---------------+-------------+
		| acm/0                       | running | small         | 10.60.98.58 |
		| acmdb/0                     | running | small         | 10.60.98.57 |
		| cloud_controller/0          | running | medium        | 10.60.98.59 |
		 … … …
		+-----------------------------+---------+---------------+-------------+
		VMs total: 40

此时，Cloud Foundry 已经完全安装好了。如果您迫不及待地想要验证此安装，您可以使用vmc命令将其中一个router的 IP 地址设为目标，然后在该目标上部署一个测试用的 Web 应用程序（见后一节）。由于此时没有配置 DNS 服务，因此，在vmc客户端以及运行浏览器来测试的机器的 hosts 文件需包含至少以下两行内容：

		<router的 IP 地址>  api.yourdomain.com  
		<router的 IP 地址><youapp>.yourdomain.com  

如果上述测试顺利通过，则说明您的 Cloud Foundry 实例正常工作。最后要做的是部署负载均衡器和 DNS服务。它们不属于 Cloud Foundry 的组件，但在生产环境中往往需要正确地设置它们。我们简要地介绍一下如何设置。

您可以部署一个硬件或软件负载均衡器 (LB) 来均匀地向多个router实例分配负载。在我们的示例部署中，我们有两个router。对于软件 LB，您可以使用Stingray 流量管理器。可从以下位置下载该软件：https://support.riverbed.com/download.htm?filename=public/software/stingray/trafficmanager/9.0/ZeusTM_90_Linux-x86_64.tgz

要解析 Cloud Foundry 实例所属的域名，需要有 DNS 服务器。基本而言，DNS 服务器会将像 *.yourdomain.com 这样带通配符的域名解析为负载均衡器的 IP 地址。如果您没有 LB，您可以设置 DNS Rotation，从而以循环方式将域解析为各个router的地址。

LB 和 DNS 设置妥善后，您便可以开始在您的实例上部署应用程序。

VMC 是使用Cloud Foundry的命令行工具。它可以执行 Cloud Foundry 上的大多数操作，例如配置应用程序、将应用部署到 Cloud Foundry 以及监控应用程序的状态。要安装 VMC，需要先安装 Ruby 和RubyGems（ Ruby Gem管理器）。目前支持 Ruby 1.8.7 和 1.9.2。接着，您可以通过下面的命令安装 VMC（有关 VMC 安装的更多信息，请参见http://docs.cloudfoundry.com/tools/vmc/installing-vmc.html）：

`$ sudo gem install vmc`

现在，可指定该 Cloud Foundry 实例的目标，相应的 URL 应形如 api.yourdomain.com，例如：

`$ vmc target api.cf.local`

使用管理员用户和密码（部署清单中指定了该凭据）登录：

`$ vmc login`

起初，系统将要求您为自己的帐户设置密码。登录后，您可以获得自己 Cloud Foundry 实例的信息：

`$ vmc info`

现在，我们来创建并部署一个简单的 Hello World Sinatra 应用程序以验证该实例。

```
$ mkdir ~/hello
$ cd ~/hello
```

创建一个名为hello.rb且包含以下内容的 Ruby 文件：

```
require 'sinatra'
get '/' do
"Hello from Cloud Foundry"
end
```

保存此文件，接下来我们要上传此应用程序：

`$ vmc push`

回答提示问题，如下所示

		Would you like to deploy from the current directory?[Yn]: 
		Application Name:hello
		Detected a Sinatra Application, is this correct?[Yn]: 
		Application Deployed URL [hello.cf.local]: 
		Memory reservation (128M, 256M, 512M, 1G, 2G) [128M]: 
		How many instances? [1]: 
		Bind existing services to 'hello'?[yN]: 
		Create services to bind to 'hello'?[yN]: 
		Would you like to save this configuration?[yN]: 

稍等片刻后，您将看到以下内容：

		Creating Application:OK
		Uploading Application:
		Checking for available resources:OK
		Packing application:OK
		Uploading (0K):OK   
		Push Status:OK
		Staging Application 'hello':OK                                                 
		Starting Application 'hello':OK   

现在，到浏览器中访问该应用程序的 URL：hello.cf.local。如果您可以看到下面的文本，则说明您的应用程序已成功部署。

恭喜！您的 Cloud Foundry 实例已经完全设置好了。它在功能上与 cloudfoundry.com 完全相同。





