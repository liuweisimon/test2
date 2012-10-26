#在vSphere上部署#

##本文根据Cloud Foundry中国架构师团队的实际部署经验总结而成，共分三个部分。##

		-	准备IaaS环境
		-	安装BOSH
		-	部署Cloud Foundry

##作者介绍##

张轩宁，有多年软件工程的从业经历，在移动应用开发、云应用的架构设计以及软件开发周期管理等方面有着丰富的经验。目前，张轩宁是VMware中国公司负责云应用生态系统的资深架构师，他帮助许多国内的运营商实施了公有云和私有云的解决方案。作为开发者关系团队的成员，他还负责开放云计算平台Cloud Foundry的开发者社区的技术支持工作。在加入VMware之前，张轩宁曾在IBM和SUN等跨国公司任职，作为解决方案架构师，帮助国内外金融、电信和医疗等行业的企业客户成功的实施了许多大型项目。

陈实，VMware公司Cloud Foundry部门实习生，复旦大学计算机科学技术学院硕士研究生，研究兴趣包括但不限于：云计算与云计算安全，PaaS架构与应用，网络与信息安全。现在VMware公司从事CloudFoundry的相关工作，主要工作领域为：通过BOSH大规模部署CloudFoundry实例，CloudFoundry在不同IaaS层的部署、集成与测试，对CloudFoundry进行自动化测试等。

陈威，VMware公司Cloud Foundry部门实习生，研究方向为Cloud Foundry生态系统的搭建，使用BOSH和Dev-Setup搭建Cloud Foundry，研究添加服务和框架与Cloud Foundry平台的集成，以及周边环境的建设，构建起本地健康的Cloud Foundry生态系统。目前是南京大学计算机系研究生三年级学生，云计算技术爱好者，关注移动互联网，喜欢捣鼓新奇玩意儿，偶尔也会拿起相机拍拍照片。

