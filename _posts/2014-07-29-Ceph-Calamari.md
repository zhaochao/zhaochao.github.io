---
layout: post
title: （旧文）安装部署Ceph Calamari 
date: '2014-07-29 15:32:01'
tags: tex standalone
---
本文最开始提交在这里:

<http://ovirt-china.org/mediawiki/index.php/安装部署Ceph_Calamari>

## 一、安装

### 1、获取calamari相关代码
    # git clone https://github.com/ceph/calamari.git
    # git clone https://github.com/ceph/calamari-clients.git
    # git clone https://github.com/ceph/Diamond

### 2、生成calamari-server安装包
    # yum install gcc gcc-c++ postgresql-libs python-virtualenv
    # cd calamari && ./build-rpm.sh

### 3、安装calamari-server
    # yum localinstall ../rpmbuild/RPMS/x86_64/calamari-server-<version>.rpm
使用yum可以自动解决依赖，如果手动安装依赖的可以这样：

    # yum install postgresql-server salt-master salt-minion supervisor
    # rpm -ivh ../rpmbuild/RPMS/x86_64/calamari-server-<version>.rpm

### 4、生成calamari-clients安装包
    # yum install npm ruby rubygems
    # npm install -g grunt grunt-cli bower grunt-contrib-compass
    # gem update --sytem && gem install compass
    # cd calamari-clients
    # make build-real
    # make dist
make dist会在上级目录生成calamari-client的压缩包

手动解压缩，建立mkdir -p opt/calamari/webapp

在解压生成的目录下，手动更新目录结构和内容

    # for dir in manage admin login dashboard
    > do
    >     mkdir -p ../opt/calamari/webapp/content/"$dir"
    >     cp -pr "$dir"/dist/* ../opt/calamari/webapp/content/"$dir"/
    > done
重新制作压缩包，然后根据Makefile里面的rpm target手动执行rpmbuild

    # rpmbuild -bb --define "_topdir /xxx/calamari-clients/../rpmbuild" \
    --define "version 1.2" --define "revision rc2_49_g3e3686d" --define \
    "tarname /xxx/rpmbuild/SOURCES/calamari-clients_product_1.2.tar.gz" \
    SPECS/clients.spec

### 5、安装calamari-clients
    # yum localinstall RPMS/x86_64/calamari-clients-1.2-rc2_49_g3e3686d.el6.x86_64.rpm

### 6、初始化calamari
    # calamari-ctl initialize
这一步在最后重启服务（主要是cthulhu）的时候一直没有结束，根据搜索到的信息，说是supervisord的问题，升级到3.0以上就不会有问题了。

### 7、生成diamond安装包
    # cd ../Diamond
    # git checkout origin/calamari
    # make rpm

### 8、将diamond-<version>.noarch.rpm复制到所有的ceph服务器
使用`yum localinstall`安装，或者`yum install python-configobj`然后使用`rpm -ivh`安装。

在所有的ceph服务器上安装salt-minion，创建/etc/salt/minion.d/calamari.conf，内容为：

    master: {fqdn}
{fqdn}对应calamari服务器的域名。

启动salt-minion服务：

    # service salt-minion restart

### 10、在Calamari服务器上配置防火墙和saltstack认证
防火墙（允许ceph服务器访问salt-master和carbon）：

**salt-master**

    # iptables -A INPUT -m state --state NEW -m tcp -p tcp --dport 4505 -j ACCEPT
    # iptables -A INPUT -m state --state NEW -m tcp -p tcp --dport 4506 -j ACCEPT

**carbon**

    # iptables -A INPUT -m state --state NEW -m tcp -p tcp --dport 2003 -j ACCEPT
    # iptables -A INPUT -m state --state NEW -m tcp -p tcp --dport 2004 -j ACCEPT

**saltstack认证：**

当ceph服务器上的salt-minion服务启动之后，会自动向salt-master请求认证。

在Calamari服务器上可以通过下面的命令查看salt-minion密钥的列表：

    # salt-key -L
刚刚启动salt-minion服务的ceph服务器会出现在Unaccepted Keys列表之后，要使得Calamari能够通过saltstack管理ceph服务器，需要对这些密钥进行认证：

    # salt-key -A


### 11、部署完成之后，可以访问calamari

## 二、后期发现的问题：

### 1、SELinux导致500错误：
由于SELinux的限制，访问页面时会出现500错误，原因是httpd_t对于anon_inodefs_t没有写入权限，可以根据审计日志生成SELinux模块：

    # ausearch -m avc -c httpd -se httpd_t -o anon_inodefs_t | audit2allow \
    -R -M httpd_anon_inodefs
    # semodule -i httpd_anon_inodefs.pp

生成的SELinux模块规则如下：

    require {
            type httpd_t;
    }
    
    #============= httpd_t ==============
    fs_rw_anon_inodefs_files(httpd_t)

### 2、打开Manage --> OSD页面无内容
查看calamari.log看到了异常，原因是httpd没有权限访问/etc/salt/master，修改权限临时解决。

### 3、打开Manage --> Logs页面无内容
查看日志，发现是访问http://xxx.xxx.xxx.xxx/api/v2/cluster/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/log发生503错误：

    HTTP 503 SERVICE UNAVAILABLE
    Vary: Accept
    Content-Type: text/html; charset=utf-8
    Allow: GET, HEAD, OPTIONS
    
    {
        "detail": "No mon servers are responding"
    }

经过研究还是SELinux的限制，通过ausearch配合audit2allow生成相应的模块，可以解决问题。

生成的SELinux模块的规则如下：

    require {
            type var_run_t;
            type httpd_t;
            class sock_file { write getattr };
    }
    
    #============= httpd_t ==============
    allow httpd_t var_run_t:sock_file { write getattr };
    files_read_var_files(httpd_t)
    init_stream_connect_script(httpd_t)

### 4、打开graphite/dashboard/页面出现HTTP 500错误
日志中提示找不到graphite的模板，在calamari的bug列表中找到对应的说明——<http://tracker.ceph.com/issues/8669>。

解决方法是：

在/opt/calamari/venv/lib/python2.6/site-packages下找到`calamari_web`的egg文件，解压缩之后手动修改calamari_web/settings.py，然后重新打包。

重启apache之后可以访问graphite/dashboard/。 
