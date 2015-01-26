---
layout: post
title: 手动安装 Staypuft
date: 2015-01-21 16:33:53
tags: OpenStack Foreman Staypuft
---

一晃，上一篇博客已经是四个月之前的事了，时间的魔力正是这样，一切的一切，当反转头再一次往后看的时候，都变得似乎没有丁点重量，不留一丝痕迹……

言归正传，通过前面两篇博客“[OpenStack Foreman Installer 安装及基本使用](/2014-09-26-OpenStack-Foreman-installer.html)”和“[OpenStack Foreman Installer —— staypuft](/2014-09-29-OpenStack-Foreman-installer-staypuft.html)”只能搭建一个基本可用的基于 Foreman 的 OpenStack 布署工具，但是对 Foreman 和 Staypuft 其实并没有多少了解，更不用说上手做些调整和修改了，所以便有这第三篇，“手动安装 Staypuft”，只是过程记录（包括遇到问题的处理和说明）：

#### 1、准备 Foreman 服务器环境
一个 CentOS 7.0.1406 的系统，配置好 YUM REPO，最少需要有 CentOS 、EPEL 、RDO 、Foreman（包括 release 和 plugins 目录）。

其它的：

* 防火墙，firewalld 这个服务还是关掉（当然也许只是因为我还没研究过，所以还不会用……）
* SELinux，如果怕麻烦，也可设置 permissive。我自己测试时和上一篇博客类似，都是顺手生成了规则。

#### 2、安装软件包
* 安装 SCL Repo 软件包

        # yum install rhscl-ruby193-epel-7-x86_64 rhscl-v8314-epel-7-x86_64

* 安装 foreman 、foreman-installer 、 foreman-proxy 、 foreman-discovery

        # yum install foreman foreman-installer foreman-proxy \
                      ruby193-rubygem-foreman_discovery

#### 3、配置 foreman
执行 `foreman-installer`：

    # foreman-installer -i

需要改动的配置有：

*  在 **Configure foreman** 中将 **configure\_epel\_repo** 和 **configure\_scl\_repo** 设置为 `false` ；
* 启用 **Configure foreman\_plugin\_discovery**，不过其中关于 discovery 镜像的下载地址、 initrd 和 vmlinuz 的文件名都不对，所以暂时不用设置 **install_images** （默认是 `false` ），后面手动下载新版本的 discovery 镜像；
* **Configure foreman\_plugin\_tasks** 对应 **foreman-tasks** 软件包，是 Staypuft 的依赖之一，不过由于目前 CentOS 、 EPEL 和 Foreman 上游 YUM 源中都没有 rpm 包，所以只能留待后面手动安装。


`foreman-installer` 配置成之后，会提示 foreman Web 界面的访问地址，以及 `admin` 用户的密码（自动生动的随机字符串，建议第一次登陆之后修改密码）。

#### 4、手动安装 Puppet 模块
Staypuft 需要的 Puppet 模块来源包括 foreman-installer 、 astapor 、 foreman-installer-staypuft：

* openstack-puppet-modules 中的模块：

        # yum install openstack-puppet-modules
        # for dir in /usr/share/openstack-puppet/modules/*
        > do ln -sf "$dir" /etc/puppet/environments/production/modules/
        > done

* quickstack 模块（来自 astapor 项目）：

        # git clone https://github.com/redhat-openstack/astapor.git
        # cd astapor
        # cp -r puppet/modules/quickstack /usr/share/foreman-installer/modules/
        # ln -sf /usr/share/foreman-installer/modules/quickstack \
                 /etc/puppet/environments/production/modules/

* foreman-installer 和 foreman-installer-staypuft 中的部份模块：

        # git clone https://github.com/theforeman/foreman-installer-staypuft.git
        # cd foreman-installer-staypuft
        # rsync -az modules/foreman/ /usr/share/foreman-installer/modules/foreman/
        # ln -sf /usr/share/foreman-installer/modules/foreman \
                 /etc/puppet/environments/production/modules/

将这些 puppet 模块导入到 foreman 中，有两种方法：

* 命令行：

        # foreman-rake puppet:import:puppet_classes[batch]

* 通过 Foreman Web 界面：

    点击 “Configure”（菜单） --> “Puppetclass”（菜单） --> “Import from ...”（按钮）。

#### 5、配置 foreman-discovery
foreman-discovery 是 foreman 用于实现主机发现的插件，可以使 foreman OS provision 的过程自动化程序更高。

* 首先，在 foreman 中创建一个新的 `environment`，名字是 `discovery` （这是 Staypuft 的要求，目前应该是固定的）；

* 下载 discovery 引导镜像，并建立 PXE 引导会用到的 tftp 目录：

        # wget http://downloads.theforeman.org/discovery/releases/latest/fdi-image-latest.tar \
                -O - | tar x --overwrite -C /var/lib/tftpboot/boot
        # cd /var/lib/tftpboot/boot/fdi-image
        # sha256sum -c SHA256SUM

* 在 foreman 中创建新的 `installation media` ，最简单的方法是将 CentOS 安装光盘挂载到一个目录，将这个目录通过 HTTP 分享，然后基于这个 HTTP url 创建 `installation media`。不过这种方法有一个明显的不足： CentOS 安装光盘中没有 puppet 软件包，导致 foreman 在 OS provision 过程中基本系统装完之后，需要手动进行 Kickstart 脚本中的一些步骤，包括通知 foreman 系统安统完成。当然解决方法也很多，比如复制一份 CentOS 安装光盘的目录结构，加上 puppet 包，然后更新目录的 YUM repo metadata；或者在 kickstart 或者配置 staypuft deployment 是指定外部软件源；…… （参见后文 **“遇到的其它问题”**）

* 配置 `Provisioning Template`。在 foreman Web 界面的 `Provisioning Template` 页面中找到 **PXELinux global default** ，增加以下内容：

        LABEL discovery
        MENU LABEL Foreman Discovery
        MENU DEFAULT
        KERNEL boot/fdi-image/vmlinuz0
        APPEND initrd=boot/fdi-image/initrd0.img rootflags=loop root=live:/fdi.iso rootfstype=auto ro rd.live.image acpi=force rd.luks=0 rd.md=0 rd.dm=0 rd.lvm=0 rd.bootif=0 rd.neednet=0 nomodeset foreman.url=http://xyz
        IPAPPEND 2

    其中，`foreman.url=http://xyz` 使用 foreman 服务器的真实地址。

    上述配置无误之后，再加上下面这一行，以保证新服务器默认从 foreman 引导项启动：

        ONTIMEOUT discovery

    最后，在 `Provisioning Template` 页面中点击 ”**Build PXE Default**” 生成默认的 PXE 引导选项。

#### 6、配置 `Provision`
由于 foreman 的 bug ，新主机向 foreman 发送 facts 时会出错，下面是该 bug 的具体信息：

[http://projects.theforeman.org/issues/8377](http://projects.theforeman.org/issues/8377)

为 foreman 打上补丁之后，记得重启 httpd 服务。

_这里可以把 /etc/puppet/autosign.conf 也修改一下，参见后文 **“遇到的其它问题”**；_

点击 “Architecture” --> “Provisioning Setup” ，然后根据向导创建新的 Provision 。

一个比较容易出现的问题是，dhcpd 服务启动不成功，原因是我的 foreman OS provision 网络会对应 OpenStack 的管理网络，不会设置网关，所以 dhcpd.conf 里面的 “option routers” 配置项没有具体的值，删除这一行，重启 dhcpd 就行。

最后，启动 OS Provision 所在网络上的一台服务器，确认从网络启动时，是否进入 foreman 引导项。

#### 7、安装 Staypuft

    # yum install gcc gcc-c++ glibc-headers libxml2-devel zlib-devel \
    ruby193-ruby-devel ruby193-rubygems-devel \
    ruby193-rubygem-jquery-rails ruby193-rubygem-jquery-ui-rails \
    ruby193-rubygem-bootstrap-sass ruby193-rubygem-sass-rails \
    ruby193-rubygem-uglifier ruby193-rubygem-therubyracer \
    ruby193-rubygem-apipie-params ruby193-rubygem-algebrick \
    ruby193-rubygem-sequel ruby193-rubygem-sinatra \
    ruby193-rubygem-daemons ruby193-rubygem-wicked \
    ruby193-rubygem-ipaddress
    # scl enable ruby193 v8314 "gem install --ignore-dependencies foreman-tasks staypuft dynflow"

在 /usr/share/foreman/bundler.d/ 下新建一个文件，如 `staypuft.rb` ，内容如下：

    gem 'staypuft'
    gem 'sass-rails'
    gem 'bootstrap-sass'
    gem 'jquery-rails'
    gem 'jquery-ui-rails'
    gem 'uglifier'

最后执行：

    # foreman-rake db:migrate
    # foreman-rake db:seed

_检查 Staypuft 数据是否初始化成功，可以参考后文 **“遇到的其它问题”**；_

#### 8、配置 foreman-tasks

    # cp /opt/rh/ruby193/root/usr/local/share/gems/gems/foreman-tasks-x.x.x/deploy/foreman-tasks.sysconfig /etc/sysconfig/foreman-tasks
    # cp /opt/rh/ruby193/root/usr/local/share/gems/gems/foreman-tasks-x.x.x/deploy/foreman-tasks.service /usr/lib/systemd/system/
    # cp /opt/rh/ruby193/root/usr/local/share/gems/gems/foreman-tasks-x.x.x/bin/foreman-tasks /usr/bin/

编辑 /usr/lib/systemd/system/foreman-tasks.service ，在 ExecStart 和 ExecStop 的命令中加上 scl 前缀，即

    ExecStart=/usr/bin/scl enable ruby193 v8314 '/usr/bin/foreman-tasks start' 
    ExecStop=/usr/bin/scl enable ruby193 v8314 '/usr/bin/foreman-tasks stop'

然后执行：

    # systemctl daemon-reload

手动测试 foreman-tasks 能否启动：

    # /usr/bin/scl enable ruby193 v8314 '/usr/bin/foreman-tasks start'

如果启动正常，则配置为随系统启动，并重新通过 systemctl 启动：

    # /usr/bin/scl enable ruby193 v8314 '/usr/bin/foreman-tasks stop'
    # systemctl enable foreman-tasks
    # systemctl start foreman-tasks

这里如果之前手动启动成功，但是 systemctl 启动失败，可能有 2 个原因：

* SELinux ：这里重新贴一下顺手生成的规则，供参考

        require {
            type passenger_t;
            type puppet_etc_t;
            class process execmem;
            class lnk_file read;
        }
        
        #============= passenger_t ==============
        allow passenger_t puppet_etc_t:lnk_file read;
        allow passenger_t self:process execmem;
        init_stream_connect_script(passenger_t)

* foreman-tasks pid 目录和文件的权限，保证 foreman 用户对 `/var/run/foreman` 目录及其下的文件和子目录有读写权限。

#### 9、创建 Staypuft deployment

* 首先，在 foreman 界面上，点击 “Administer” --> “Setting” --> “StayPuftProvioning” ，将 `base_hostgroup` 设置为前面创建 `Provision` 时自动生成的 `hostgroup` ；

* 在 “Architecture” --> “Subnets” 页面中为 OpenStack 创建必要的网络；我测试时，OS provison 对应的网络是 OpenStack 管理网络，还需要建立外部网络和租户网络；

* 点击 “OpenStack Installer” --> “New deployment” ，按装向导创建新的 Staypuft 布署方案；

* 在 “Configure” --> “Hostgroups” 页面检查默认的 base\_hostgroup 的 `environment` 参数是 `production` ，并为 OS provision 配置默认的系统 root 用户密码；

* 在 OS provions 对应的网络中，启动一台服务器，被 foreman 发现后，在 Staypuft 布署方案中添加并分配角色，尝试布署。

#### 10、遇到的其它问题

* OS provision 完成之后，主机无法从 foreman 服务器获取 puppet catalog ：

    foreman 服务器上，puppet 服务应该配置自动为新的 puppet agent 作证书签名：

    编辑 /etc/puppet/autosign.conf ，在其中加上一行：

        *

    重启 httpd 服务。

    如果以前出现节点证书没有签名的情况，可以通过下面的命令手动签名（在 foreman 服务器上执行）：

        # puppet cert list
        # puppet cert sign <puppet agent 所在服务器的 fqdn>

* Staypuft 布署方案中的 puppet 模块不正确

    foreman 是 rails 应用，staypuft 是 foreman 的一个插件，布署方案中服务器角色和 OpenStack 服务 、Puppet 模块之间的关系也保在 foreman 数据库中，初始数据是在装完 Staypuft 之后，执行 `foreman-rake db:migrate` 和 `foreman-rake db:seed` 写入数据库的。如果 Staypuft 布署方案中 puppet 模块不正确，说明数据初始化出现问题，必须检查 Staypuft 安装过程是否出现问题，然后再次执行上述 2 条 `foreman-rake` 命令（多次执行这两条命令不会影响 foreman 数据库中已有数据）。

    另外也可以手动检查 foreman 数据库，确认 staypuft 相关数据已正确初始化：

        # sudo -u postgres psql foreman
        foreman=# select * from staypuft_role_services;
        foreman=# select * from staypuft_service_classes;

    如果 role （服务器角色） 和 services （ OpenStack 服务） 、 service 和 classes （ Puppet 模块）的数据都存在，说明 Staypuft 数据库数据初始化没有问题。

* 查看服务器 OS Provision 的具体配置，以及手动通知 Foreman 服务器 OS Provision 已经完成

    服务器在 OS Provision 阶段的 Kickstart 配置可以通过访问下面的 url 查看：

        http://foreman/unattended/provision?spoof=xxx.xxx.xxx.xxx （ xxx.xxx.xxx.xxx 是该服务器的 ip 地址）

    如果由于 `installation media` 中没有 puppet 软件包，基本系统装完可手动完成 Kickstart 配置没有完成的操作，然后通知 Foreman 服务器 OS Provision 已经完成，该命令也在 Kickstart 配置的最后：

        wget -q -O /dev/null --no-check-certificate http://foreman/unattended/built?token=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

    url 后面的 token 可以省略。


#### 相关链接
* [apporc](http://blog.apporc.org) 的 2 篇博客

    * [openstack部署实践-foreman(rhel-osp)硬件部分](http://blog.apporc.org/?p=660)
    * [openstack部署实践-foreman(rhel-osp)软件部分](http://blog.apporc.org/?p=655)
