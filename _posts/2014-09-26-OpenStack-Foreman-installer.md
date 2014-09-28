---
layout: post
title: OpenStack Foreman Installer 安装及基本使用
date: 2014-09-26 12:10:33
tags: OpenStack Foreman
---

OpenStack 的部署方式有很多，基础文档可以从这里获取：

[http://docs.openstack.org](http://docs.openstack.org)

RedHat 有一篇文档对不同的部署方式做了整理和对比，不过需要订阅访问：

[Evaluating OpenStack: Deployment Types and Tools](https://access.redhat.com/articles/1140323)

不过 RedHat 在其他的 Deploying OpenStack 系列文档里面大概说明了一下他们提供的3种部署方式的区别：

1. 手动部署

    纯学习目的，了解 OpenStack 的结构以及各组件的安装和基本配置。

2. PackStack

    验证性质的部署方式（ proof-of-concept ）。PackStack 提供了命令行工具，使用 Puppet 模块完成（在已经安装了操作系统的服务器上） OpenStack 的快速部署，除了使用命令行工具，也可以使用 annswer 文件这种形式完成非交互式的自动部署。

    RedHat 的建议是不应使用 PackStack 来进行生产环境的 OpenStack 部署，原因是 PackStack 为了简化部署流程，对 OpenStack 各组件的配置项做了很多预设定，导致无法实现 HA 、 Load Balance 等结构，配置复杂网络拓扑也会比较麻烦。

    不过说起来，PackStack 是使用 Puppet 的，对于有经验的用户来说，自行修改相关模块的配置，应该不会特别困难。

3. RHEL OpenStack Platform Installer

    这是 RedHat OpenStack Platform 产品的部署工具，提供了 Web 界面、向导式的自动化部署方式（基于 Foreman 和 Puppet ）。包含了一些更好的特性，比如 LiveCD ，部署过程的统筹、排序，裸机自动发现等。

    不过 RedHat 的文档里面说明， RHEL OpenStack Platform Installer 目前只能在 RHEL 6.5 上使用（用来部署 RHEL 7 以及 在 RHEL 7 上部署 RHEL OpenStack Platform 5 则是没问题的）。

    关于 RHEL OpenStack Platform 5 的文档参见：

    [Red Hat Enterprise Linux OpenStack Platform 5](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux_OpenStack_Platform/5/index.html)

从特性上看，当然是最后一种比较吸引人，不过更详细的文档可能不太容易找到了。我们能找到的是这个：

[https://github.com/redhat-openstack/astapor](https://github.com/redhat-openstack/astapor)

我并没有做深入的调查，所以不能确认 RHEL OpenStack Installer 的上游项目就是这个 astapor，但是从项目的 README 、脚本以及 Puppet 模块可以看出，两者重合度还是很高。

astapor 的 README 文件写的比较清楚，主要的步骤如下：

1. 执行 foreman_server.sh ；
2. 安装完成之后，访问 https://FQDN 就可打开 foreman 的 Web 界面；
3. 将 foreman_server.sh 生成的 /tmp/foreman_client.sh 复制到的准备部署为 OpenStack Controller 的服务器并执行，这个脚本会安装 puppet 并主动向 foreman 服务器注册；
4. 在 foreman 的 Web 管理界面上，可以看到新注册的服务器，编辑该服务器，选择“主机组”；
5. 回到准备安装 OpenStack 组件的服务器，执行 puppet agent -tv，会自动开始部署 OpenStack Controller 相关软件并进行配置；
6. 对其他 OpenStack 节点服务器，重复上述第 3 - 5 步，根据服务器角色不同选择主机组。

具体的操作流程这里不在一一说明，说一下在 CentOS 7 中遇到的问题：

#### 1、安装依赖的软件包和配置 YUM 仓库
要使用 astapor ，首先还是需要安装一些依赖的软件包的：

    # yum install puppet openstack-puppet-modules foreman foreman-mysql \
                  foreman-installer ruby193-rubygem-foreman_simplify \
                  mysql-server augeas

不过 foreman 安装过程会有不少包不在 foreman 自己的 YUM 仓库，也不在 CentOS 和 EPEL 的仓库里面，需要从 SCL 软件集获取，这些包目录可以从下面的网站获取：

[https://www.softwarecollections.org](https://www.softwarecollections.org)

在我的测试过程需要安装的软件包都在 Ruby193 和 V8314 这两个集合中，给个YUM repo 文件的例子：

    [softwarecollections-ruby193]
    name=Ruby192 packages from softwarecollections.org
    baseurl=https://www.softwarecollections.org/repos/rhscl/ruby193/epel-7-x86_64/
    gpgcheck=0

    [softwarecollections-v8314]
    name=V8314 packates from softwarecollections.org
    baseurl=https://www.softwarecollections.org/repos/rhscl/v8314/epel-7-x86_64/
    gpgcheck=0



#### 2、跳过系统 release 信息检测
`foreman_server.sh` 中以下几行：

    if [ ! -f /etc/redhat-release ] || \
        cat /etc/redhat-release | grep -v -q -P 'release 6.[456789]'; then
        echo "This installer is only supported on RHEL 6.4 or greater."
        exit 1
    fi

对于 CentOS 7 来说就没有什么意义了，直接可以去掉。

#### 3、禁止自动添加外部 YUM 仓库
`foreman-installer` 默认会自动添加 EPEL 和 SCL 的 YUM 仓库，不过在获取 repodata 可能会出错，导致 `foreman_server.sh` 无法完成，因此在 `foreman_server.sh` 生成 `install.pp` 的地方加上 2 行配置：

将

    class { 'foreman':
        db_type => 'mysql',
        custom_repo => true,
    }

修改为

    class { 'foreman':
        db_type => 'mysql',
        custom_repo => true,
        configure_epel_repo => false,
        configure_scl_repo => false
    }

#### 4、修改 seeds.rb
seeds.rb 中包含了在 foreman 的数据库初始化和 astapor 提供的 puppet 模块信息，因此比较重要。但是由于 foreman 给 el7 打的rpm 包还是使用 ruby193，在 seeds.rb 中调用 facter 时很多系统信息无法获取，会导致执行 `rake db:seed` 时出错。

下面是我拿系统基本命令替换 facter 的 patch ，可能比较难看，但是可以用：

    @@ -183,8 +183,13 @@ if ENV["FOREMAN_PROVISIONING"] == "true" then
         
    # Subnets - use Import Subnet code
    s=Subnet.find_or_create_by_name "OpenStack"
    -  s.network=Facter.value("network_#{secondary_int}")
    -  s.mask=Facter.value("netmask_#{secondary_int}")
    +  #s.network=Facter.value("network_#{secondary_int}")
    +  #s.mask=Facter.value("netmask_#{secondary_int}")
    +  t_ip=`ifconfig #{secondary_int} | awk '/netmask/ {print $2}'`.chomp
    +  t_mask=`ifconfig #{secondary_int} | awk '/netmask/ {print $4}'`.chomp
    +  t_network=`ipcalc -n #{t_ip} #{t_mask} | awk -F'=' '{print $2}'`.chomp
    +  s.network=t_network
    +  s.mask=t_mask
    s.dhcp = Feature.find_by_name("DHCP").smart_proxies.first
    s.dns = Feature.find_by_name("DNS").smart_proxies.first
    s.tftp = Feature.find_by_name("TFTP").smart_proxies.first
    @@ -237,10 +242,12 @@ if ENV["FOREMAN_PROVISIONING"] == "true" then
    
    # Override all the puppet class params for quickstack
    primary_int=`route|grep default|awk ' { print ( $(NF) ) }'`.chomp
    -  primary_prefix=Facter.value("network_#{primary_int}").split('.')[0..2].join('.')
    -  sec_int_hash=Facter.to_hash.reject { |k| k !~ /^ipaddress_/ }.reject { |k| k =~ /lo|#{primary_int}/ }.first
    -  secondary_int=sec_int_hash[0].split('_').last
    -  secondary_prefix=sec_int_hash[1].split('.')[0..2].join('.')
    +  #primary_prefix=Facter.value("network_#{primary_int}").split('.')[0..2].join('.')
    +  primary_ip=`ifconfig #{primary_int} | awk '/netmask/ {print $2}'`.chomp
    +  primary_mask=`ifconfig #{primary_int} | awk '/netmask/ {print $4}'`.chomp
    +  primary_network=`ipcalc -n #{primary_ip} #{primary_mask} | awk -F'=' '{print $2}'`.chomp
    +  primary_prefix=primary_network.split('.')[0..2].join('.')
    +  secondary_prefix=t_network.split('.')[0..2].join('.')
    end

#### 5、完善 foreman-installer 中对系供 release 版本的检测
foreman-installer 中对于 CentOS 等 RHEL 克隆发行版的检测还是不太完善，比如

/usr/share/foreman-installer/modules/mysql/manifests/params.pp，需要将：

    'RedHat': {
        if $::operatingsystemlease >= 7 {
            $provider = 'mariadb'
        } else {
            $provider = 'mysql'
        }
    }

修改为：

    /^(RedHat|CentOS|Scientific)$/: {
        if $::operatingsystemmajrelease >= 7 {
            $provider = 'mariadb'
        } else {
            $provider = 'mysql'
        }
    }

其他的模块，碰到类似问题的话，也需要进行修改。


#### 6、执行 foreman_server.sh
astapor 提供了 quickstack （ 部署 OpenStack 平台的 puppet 模块），其路径最后会写入到 puppet 的系统配置中，因此建议把 git clone 下来的代码放到比较固定、不容易干扰的目录，比如：

    # mv astapor /usr/share/openstack-foreman-installer

另外由于 CentOS 7 中 SELinux 默认是启用的，需要恢复下目录的安全上下文：

    # restorecon -RFv /usr/share/openstack-foreman-installer

还需要做的是，要为主机设置好主机名并正确配置 /etc/hosts ：

    # hostnamectl set-hostname foreman.zhaochao.eayunstack.eayun.com
    # echo "xxx.xxx.xxx.xxx foreman.zhaochao.eayunstack.eayun.com" >> /etc/hosts

最后，执行 foreman_server.sh 时，需要为 FOREMAN_GATEWAY 环境变量赋值（指定 foreman 服务器的 IP 地址即可，脚本会检测系统的网络配置，并且使用非默认路由对应的网卡给后续的 privison 使用，因此 FOREMAN_GATEWAY 可以使用前期配置好的内部 IP 地址）：

    # FOREMAN_GATEWAY=xxx.xxx.xxx.xxx ./foreman_server.sh

#### 7、foreman_proxy 注册不成功
`foreman_server.sh` 脚本如果由于前期配置和环境准备不完善，多次执行之后仍然可能会碰到一些问题，比如

    Info: Class[Foreman_proxy::Register]: Scheduling refresh of Foreman_smartproxy[foreman.zhaochao.eayunstack.eayun.com]
    E, [2014-09-26T18:04:53.419648 #26850] ERROR -- : 404 Resource Not Found

跟踪脚本执行，可以看到是下面这条命令出错：

    puppet apply --verbose installer.pp --modulepath=modules

而从后面的详细输出可以看到，原因是，访问

    https://foreman.zhaochao.eayunstack.eayun.com/apidoc/v2.json

时返回了 HTTP 404。

而根据 foreman 的日志提示，可能需要执行 `rake apipie:cache`，参考 `foreman_server.sh` 脚本中的方式执行：

    sudo -u foreman scl enable ruby193 \
        'cd /usr/share/foreman; rake apipie:cache RAILS_ENV=production'

之后，可以重新执行 `foreman_server.sh` 。

#### 8、无法使用 admin 登录 foreman
`foreman_server.sh` 执行成功之后，使用 admin/changeme 总是提示验证失败，具体原因不明，应该是 foreman installer 的 puppet 模块还存在一些问题，没有具体研究。可以使用下面的命令，重置 admin 用户的密码：

    # foreman-rake permissions:reset

#### 9、foreman 中没有 qickstack 相关的模块和主机组信息
最开始由于对 puppet 完全没概念，以为是配置问题，所以直接的解决方法时，把 astapor 提供的 puppet 模块全部复制一份到 puppet 的默认目录中去，比如 `/etc/puppet/environments/production/modules/` ，的确重新执行 `foreman_server.sh` 或者使用 `rake` 命令手动导入 puppet 模块，在 foreman 中就可以看到 quickstack 的模块和主机组了。

但是问题依然存在，在 provision 服务器节点时，puppet agent 总是提供找不到对应的 qickstack 模块中的组件，最后无奈使用了 `strace`，确认原因是 puppet-master 对于 quickstack 模块的目录和文件没有权限访问。

而基本 rwx 权限是没有问题，/var/log/audit/audit.log 竟然没有 AVC 的报错，但是使用 `restorecon` 恢复这些文件默认的安全上下文之后，puppet agent 就可以正常运行了。

#### 10、对裸机进行部署
foreman 安装完整之后，PXE 、DHCPD 、 TFTPD 等服务已经配置好了，需要做的事情比较简单，准备安装界面和 pxelinux 配置文件。

不过 RHEL OpenStack Platform Installer 中提到 bare metal provision 则是使用了 `foreman_discovery` 插件，仍然是基于上述几个服务，但是在 foreman 界面上多了类似“主机扫描”的页面，而且对于 PXE 提供的安装介质有一定的要求，需要在 initrd 中使用 `foreman_discovery` 提供的脚本向 foreman 服务器提交注册请求。


### 写这篇博客的过程中，搜到的一些 slide ：

* [http://www.slideshare.net/domcleal/cfgmgmt-2014-rdo](http://www.slideshare.net/domcleal/cfgmgmt-2014-rdo)
* [http://rhsummit.files.wordpress.com/2014/04/summit2014_scale_cloud_infra-20140416_wfoster_kambiz.pdf](http://rhsummit.files.wordpress.com/2014/04/summit2014_scale_cloud_infra-20140416_wfoster_kambiz.pdf)
