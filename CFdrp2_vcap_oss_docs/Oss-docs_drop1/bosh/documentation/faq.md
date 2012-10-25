# Cloud Foundry BOSH 常见问题解答



## 什么是 Cloud Foundry BOSH？


Cloud Foundry BOSH 是一条开源工具链，用于对大规模分布式服务进行发行版工程处理、部署和生命周期管理。



## BOSH 发挥什么作用？


BOSH 旨在推动服务的系统性、规范性的演变，可为 Cloud Foundry 生产实例的运行提供方便。

BOSH 实现了各种云基础架构的自动化，可帮助进行有针对性的服务更新，从而产生一致的结果并将停机时间缩至最短。



## BOSH 供哪些人使用？


BOSH 主要是为操控 Cloud Foundry 的大规模生产部署的人员设计的。

虽然 BOSH 不是运行 Cloud Foundry 时所必需的，但仍然建议将 BOSH 用于大规模的 Cloud Foundry 实例。



## CloudFoundry.com 是否使用 BOSH？


是的，CloudFoundry.com 自面世以来便一直使用 BOSH 来创建和更新生产服务，并将它用于数十种构成 CloudFoundry.com 的开发、测试和暂存云。有数千例 Cloud Foundry 部署都是使用 BOSH 完成的。


 
## BOSH 是如何许可的？


BOSH 是开源软件，依照 Apache 2 许可证发行。

  

## VMware 为何开放 BOSH 的源代码？


VMware 之所以开放 BOSH 的源代码，是为了让提供商们能够运行规模更大、质量更高的 Cloud Foundry 实例。



## BOSH 支持何种云基础架构？


BOSH 包含一个云提供商接口 (CPI)，此接口是一个面向多种云基础架构的抽象层。

目前，有适用于 VMware vSphere 的生产级实现。

此外还提供对 Amazon Web Services 的支持，这种支持正在开发之中。

VMware 正在致力于开发将于 2012 年晚些时候交付的 vCloud Director 支持。VMware 鼓励以开源方式推出适合其他云基础架构（如 CloudStack、Eucalyptus、OpenStack）的实现。



## 我可以在何处下载 BOSH 和了解它的相关信息？


可在 http://cloudfoundry.org/ 找到 BOSH 文档和软件


 
## BOSH 表示的含义是什么？


BOSH 是“BOSH Outer Shell”（BOSH 外层）的递归缩写词。


 
## 我是否应使用 BOSH 来完成我的 Cloud Foundry 生产部署？


对于 Cloud Foundry 的大规模生产部署，我们建议使用 BOSH。



## BOSH 与 chef、puppet 和配置管理工具有何关系？


Chef 和 Puppet 大体上属于配置管理工具。

BOSH 在大型分布式系统的配置、连续部署、包更新、监视、虚拟机管理以及总体要求之间架起了桥梁。

BOSH 包含多个组件，其中就有一个可以利用 chef、puppet 及其他解决方案的开放式配置管理层。



