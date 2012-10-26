注意：这是这些脚本的初级发行版。

说明：
此子目录包含在单主机或多主机环境中部署和设置 Cloud Foundry 时
要使用的 chef cookbook。这些 cookbook 假定它们是在
全新安装的 Ubuntu 10.04.4 服务器上运行。

快速入门：
-----------
   部署命令：
      vcap/dev_setup/bin/vcap_dev_setup

         该命令会将 Cloud Foundry 源代码下载到
            $HOME/cloudfoundry/vcap

         它将在下面的目录下创建一个名为“devbox”的默认部署
            $HOME/cloudfoundry/.deployments/devbox

   启动命令：
      vcap/dev_setup/bin/vcap_dev start

   停止命令：
      vcap/dev_setup/bin/vcap_dev stop



更多选项：
------------

o 使用您已经下载的 Cloud Foundry 源代码。
   部署命令：
     例如，如果 Cloud Foundry 源代码已下载到 $HOME/projects/vcap 中。

      $HOME/projects/vcap/dev_setup/bin/vcap_dev_setup -d $HOME/projects

        该命令不会下载 Cloud Foundry 源代码。它只是将配置文件设置为
        使用 $HOME/projects/vcap 中的 Cloud Foundry 源代码。

        它将在下面的目录下创建一个名为“devbox”的默认部署
          $HOME/projects/.deployment/devbox

   启动命令：
      $HOME/projects/vcap/dev_setup/bin/vcap_dev -d $HOME/projects start

   停止命令：
      $HOME/projects/vcap/dev_setup/bin/vcap_dev -d $HOME/projects stop


o 使用自定义部署配置

  例如，如果您要使用 multihost 示例配置文件
  $HOME/projects/vcap/dev_setup/deployments/sample/multihost_mysql/dea.yml

   部署命令：
      $HOME/projects/vcap/dev_setup/bin/vcap_dev_setup -d $HOME/projects -c $HOME/projects/vcap/dev_setup/deployments/sample/multihost_mysql/dea.yml

        该命令不会下载 Cloud Foundry 源代码。它只是将配置文件设置为
        使用 $HOME/projects/vcap 中的 Cloud Foundry 源代码。

        它将在下面的目录下创建一个名为“dea”的部署
          $HOME/projects/.deployment/dea

   启动命令：
      $HOME/projects/vcap/dev_setup/bin/vcap_dev -d $HOME/projects -n dea start

   停止命令：
      $HOME/projects/vcap/dev_setup/bin/vcap_dev -d $HOME/projects -n dea stop


o 使用自定义域
  例如，如果您不希望自己的 CloudFoundry 域是 vcap.me
  $HOME/projects/vcap/dev_setup/bin/vcap_dev_setup -D myowndomain.com

  之后，您可以使用下面的命令来设定 CloudFoundry 安装目标：
  vmc target api.myowndomain.com

o 服务备份：
  默认情况下，备份功能处于禁用状态。如果您要启用此功能，可以运行下面的
  脚本：
    $HOME/projects/vcap/dev_setup/bin/vcap_service_backup_ctl start|stop|restart|status <service_type>

  service_type 是可选的，它可以是支持备份功能的其中一项服务；或者，
  当 service_type 为空时，则为所有这些服务。

  start|restart：向 crontab 添加行
  stop：从 crontab 中删除行
  status：从 crontab 中获取行

  如果您要清除属于未置备的服务实例的备份，
  应使用备份管理器。

  默认情况下，backup_manager 未安装，您应在部署配置文件中添加“backup_manager”，然后
  重新运行 vcap_dev_setup。
    $HOME/projects/vcap/dev_setup/bin/vcap_dev_setup -c <您的配置文件>
    $HOME/projects/vcap/dev_setup/bin/vcap_dev start backup_manager


o 服务生命周期：
  默认情况下，服务生命周期功能处于禁用状态。
  如果要启用此功能，请在部署配置文件中添加以下几项内容。
    - services_redis
    - serialization_data_server
    - snapshot_manager
    - redis_worker
    - mysql_worker
    - mongodb_worker
    - postgresql_worker

  然后，重新运行 vcap_dev_setup 和启动操作。
    $HOME/projects/vcap/dev_setup/bin/vcap_dev_setup -c <您的配置文件>
    $HOME/projects/vcap/dev_setup/bin/vcap_dev restart

  snapshot_manager 用于清除属于未配置的服务实例的快照。

o 部署全部组件：

  默认情况下，vcap_dev_setup 使用“$HOME/projects/vcap/dev_setup/deployments/devbox.yml”作为部署
  配置文件，该文件包含了 cloudfoundry 中的大多数组件，但并未包含全部组件。
  要部署全部组件，请参见下面的命令：
    $HOME/projects/vcap/dev_setup/bin/vcap_dev_setup -c $HOME/projects/vcap/dev_setup/deployments/devbox_all.yml
    $HOME/projects/vcap/dev_setup/bin/vcap_dev start

注意：
要了解有关自定义部署配置文件及多主机安装的更多信息，
请参见 vcap/dev_setup/deployments/README 自述文件。

