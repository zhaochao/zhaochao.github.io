---
layout: post
title: OpenStack Foreman Installer —— staypuft
date: 2014-09-28 18:53:43
tags: OpenStack Foreman
---

在[前一篇博客](/2014-09-26-OpenStack-Foreman-installer.html)中对 astapor 的基本安装和使用进行了一些说明，但是只有 astapor 是实现不了 RedHat OpenStack Platform Installer 文档中提到的“部署过程的统筹和排序”这个特性的，本来准备过段时间再回来调查下，不过稍微搜索了一下，就发现了这个项目：

[https://github.com/theforeman/staypuft](https://github.com/theforeman/staypuft)

相关的介绍可以看下面这些链接：

* [Project Staypuft: An Introduction to the Foreman OpenStack Installer](https://www.openstack.org/summit/openstack-summit-atlanta-2014/session-videos/presentation/project-staypuft-an-introduction-to-the-foreman-openstack-installer)
* [Public OpenStack Foreman Installer](http://allthingsopen.com/2014/06/10/openstack-foreman-installer/)

    _这个是 RedHat 的演示文档，不得不说他们的文档做得不错，产品本身先不说，光看演示文档就比较能吸引和打动目标客户。。。_

staypuft 是 foreman 自行维护的插件，但是目前没有 for el7 的 rpm 安装包，不过安装不会太麻烦：

1. [Foreman pakcaging](https://github.com/theforeman/foreman-packaging)，提供了打包所需要的东西，可以自行打包；
2. [foreman-installer-staypuft](https://github.com/theforeman/foreman-installer-staypuft)，这个应该是社区比较推荐的方式，可以处理好一些数据初始化相关的工作；
3. 使用 gem 安装。

由于目前主要目的是简单测试和体验一下 starpuft，因此直接使用 gem 安装。这种方式安装不是比较方便，安装过程出现文档的可能性比较大。下面是这次测试的一些记录：

#### 1、安装（依赖和 staybuf 本身）
具体的依赖没有仔细研究，不过使用 gem 安装 starpuft 时有依赖的 gem 包需要对 c 和 c++ 代码进行编译，因此首先需要保证系统具备开发编译环境。

    # yum install gcc gcc-c++ glibc-headers libxml2-devel zlib-devel \
                  ruby193-ruby-devel ruby193-rubygems-devel \
                  ruby193-rubygem-sass-rails ruby193-rubygem-uglifier \
                  ruby193-rubygem-therubyracer
    # scl enable ruby193 'gem install staypuft'

### 2、配置 foreman
首先配置 foreman 调用 starpuft，使用的方法是在 foreman 目录下的 bundler.d 增加配置文件：

/usr/share/foreman/bundler.d/staypuft.rb

    gem 'staypuft'

/usr/share/foreman/bundler.d/assets.rb

    gem 'sass-rails'
    gem 'therubyracer'
    gem 'bootstrap-sass'
    gem 'jquery-rails'
    gem 'jquery-ui-rails'
    gem 'uglifier'

### 3、初始化数据
执行命令：

    sudo -u foreman scl enable ruby193 'cd /usr/share/foreman; rake db:migrate RAILS_ENV=production'

或者：

    foreman-rake db:migrate

不过目前 softwarecollections 提供的 therubyracer 有点问题，会提供无法加载 therubyracer 库，解决的方法是通过 gem 重新安装更新版本：

    scl enable ruby193 'gem install therubyracer'

### 4、添加 SELinux 策略
上述的问题解决完了之后，foreman Web 界面的 Openstack Installer 页面仍然会出现问题，foreman 日志中会出现这样的信息：

    No such file or directory - "/usr/share/foreman/tmp/sockets/dynflow_socket" (Errno::ENOENT)

查了很久不能确认问题，还是怀疑是 SELinux 的问题，看了下 /var/log/audit/audit.log ，的确有相关报错：

    # ausearch -m avc -c ruby -ts 18:01:00 | audit2allow -R

根据上述命令生成的 SELinux 策略，进行调整精简，可以得到这样的策略文件：

    require {
                type passenger_t;
                        class process execmem;
    }

    #============= passenger_t ==============
    allow passenger_t self:process execmem;

可以手动编译成 SELinux 模块，或者直接使用

    # ausearch -m avc -c ruby -ts 18:01:00 | audit2allow -M passenger-execmem

最后加载 SELinux 模块：

    # semodule -i passenger-execmem.pp

### 5、设置 base_hostgroup
Foreman Web 界面的 “OpenStack Installer” 可以访问之后，可以通过 “New Deployment” 来配置自定义的 OpenStack 部署流程或者方式，正是我们需要的“部署过程的统筹和排序”这一特性。但是点击 “New Deployment” 会提示：

    missing base_hostgroup

原因可能是 staypuft 数据初始化的过程仍然存在问题，不过可以手动做一些处理，比如自己定义 “base_hostgroup” ：

    Foreman Web 界面 --> “Administre” 菜单 --> “settings”
      --> “StaypuftProvisioning” --> 修改 “base_hostgroup” 的值

至此，staypuft 基本可用了（当然仍然需要保证基本的 puppet 模块是可用的），由于能够自定义部署架构和流程，所以相对来说是目前最理想的 OpenStack 部署方式。
