---
layout: post
title: Koji 文档整理
date: 2014-08-18 15:49:48
tags: Koji building-system
---

这是关于 Koji 软件包构建系统的文档整理，绝大多数文字都从项目 wiki 以及相关技术文档翻译而来，参见结尾的[原文链接](#原文链接)。

对于这些文档中中不是很清晰的地方补充了一些内容和实际的例子。

## Koji 基本概念

### Koji 简介
Koji 是 Fedora 项目的编译打包系统，使用 mock 产生 chroot 环境并在其中完成具体的编译打包任务。

#### 主要的特性

* 每一次构建过程都会创建新的 buildroot 环境；
* 提供可靠的 XML-RPC api，和其他工具整合很容易；
* Web 用户界面，提供 SSL 和 Kerberos 验证机制；
* 简洁可移植的命令行客户端程序；
* 用户可以创建自己的 buildroot ；
* buildroot 之下内容的信息会记录在数据库中；
* 版本化的数据。

### Koji 架构

#### 术语

* Package

    source rpm 包的名称，这是软件包的通用名称，不包括特定的构建信息或者次级软件包，如： kernel 、 glibc 等；

* Build

    软件包的特定构建，包括所有构建平台上可能产生的结果以及所有次级软件包，如： kernel-2.6.9-34.EL、 glibc-2.3.4-2.19；

* RPM

    软件包构建完成之后产生的具体RPM，如： kernel-2.6.9-34.EL.x86_64 、 kernel-devel-2.6.9-34.EL.s390 、 glibc-2.3.4-2.19.i686 、 glibc-common-2.3.4-2.19.ia64 。

### Koji 组件
Koji 由以下部分组成：

#### Koji-hub

Koji-hub 是所有 Koji 操作的中心，是运行在 apache 服务的 mod_wsgi 模块之上的 XML_RPC 服务程序。Koji-hub 只被动地接受 XML_RPC 请求，依赖于 build daemon 和其他组件完成通信交互过程。Koji-hub 也是唯一可以访问数据库的组件，同时是2个对文件系统有写入权限的组件之一。

#### Kojid

Kojid 是 build daemon，需要在所有的构建服务器上运行。它主要的工作是获取构建请求并对这些请求进行相应的处理。Kojid 实际上是从 Koji-hub 那里获取任务，Koji 也支持构建之外的其他任务，比如创建安装介质镜像。Kojid 使用 mock 来完成构建任务，每次构建都会创建新的 buildroot。Kojid 是使用 Python 语言开发的，和 Koji-hub 之间通过 XML_RPC 协议进行通信。

#### Koji-web

Koji-web 是运行在 mod_python 模块之上的一系列脚本，加上 Cheetah 模板引擎提供了一套基于 Web 的交互界面。Koji-web 是 Koji-hub 的客户端，为最终用户提供了可视化的操作界面，在该界面上可以完成有限的管理操作。Koji-Web 展示了 Koji 系统中大量的信息，同时也支持完成特定的操作，如终止构建任务等。

#### Koji-client
 
Koji-client 是使用 Python 语言编写的命令行工具，也提供了可以和 Koji 交换的大量 hook 接口。使用 Koji-client 可以直接查询 Koji 系统中很多数据，也可以完成不少操作，如添加用户、提交新的构建任务等。

#### Kojira

Kojira 是维护由构建结果组成的仓库的 daemon 程序，负责清楚冗余的 build root 目录以及构建任务完成之后的清理工作。

### 软件包的组织方式

#### tag 和 target

Koji 通过 tag 对软件包组织管理：

* tag 相关信息在数据库中进行维护，而不是在硬盘上；
* tag 之间可以形成多重继承关系；
* 每个 tag 都拥有自己的 package 列表（该列表可以继承）；
* package 的所有者可以根据 tag 进行设置（所有者属性也可以继承）；
* tag 之间的继承关系可以进行配置；
* 构建软件包时需要指定的是 target ， 而不是 tag。

target 用来确认软件包的构建过程在何处进行，构建完成之后如何对软件包进行标记（ tag ）。这样，在软件的发布周期之间，tag 可以不断进行变化，而 target 则可以保持不变。使用下面的命令可以获取 Koji 系统中所有的 target 列表：

    $ koji list-targets

加上 `--name` 选项可以针对指定的 target 进行查询：

    $  koji list-targets --name EayunOS-4-0
    Name                           Buildroot                      Destination   
    ----------------------------------------------------------------------------
    EayunOS-4-0                    EayunOS-4-0-build              EayunOS-4-0

上述示例的含义是： EayunOS-4-0 （ target ） 构建时使用 EayunOS-4-0-build （ tag ）中的软件包列表并创建 buildroot ，生成的软件包的标签为 EayunOS-4-0 （ tag ）。
    
要获取 Koji 系统中所有的 tag 列表，使用下面的命令：

    $ koji list-tags

#### 软件包列表

每一个 tag 都有自己的软件包列表，使用 `list-pkgs` 命令获取软件包列表（需要指定 tag 名称）：

    $ koji list-pkgs --tag EayunOS-4-0
    Package                 Tag                     Extra Arches     Owner    
    ----------------------- ----------------------- ---------------- ---------
    eayunos-release         EayunOS-4-0                              zhaochao 
    eayunos-lic             EayunOS-4-0                              zhaochao 
    virt-viewer-win         EayunOS-4-0                              zhaochao

命令输出结果中，第一列是软件包的名称，第二列显示软件包列表的继承关系，最后一列则显示软件包的所有者。

#### 最近的构建任务

使用 latest-pkg 命令可以查看最近的构建任务：

    $ koji latest-pkg --all EayunOS-4-0
    Build                                     Tag                   Built by
    ----------------------------------------  --------------------  -----------
    eayunos-lic-0.0.1-3                       EayunOS-4-0           zhaochao
    eayunos-release-4-0.el6.2                 EayunOS-4-0           zhaochao
    virt-viewer-win-0.0.1-4.el6               EayunOS-4-0           kojiadmin

命令的输出结果中会显示最近的构建任务，所属的 tag ，以及提交任务的用户。

## Koji 部署

### 选择安全验证方式
Koji 支持 2 种安全验证方式： SSL 和 Kerberos 。如果只是使用 koji 命令行工具也可以直接使用用户名和密码登录这种方式，但是 kojiweb 不支持这种方式，而且一旦 kojiweb 配置了 SSL 或者 Kerberos 验证，则用户名和密码登录这种方式将会失效，因此在进行 Koji 部署之前需要首先确定安全验证的方式。

**Kerberos验证**，前提是已经具备了可用的 Kerberos 环境，初始化 Koji 用户数据时必须已经建立好管理用户的安全凭证；

**SSL验证**，需要为 xmlrpc 服务程序、Koji 其他各组件、管理用户准备好 SSL 证书。

#### 准备 SSL 安全验证的环境

* 生成证书

    可以考虑建立 `/etc/pki/koji` 目录，创建 `ssl.cnf`，然后使用 `openssl` 命令生成证书。

    在 `ssl.cnf` 中添加个性化的设置可以简化证书的生成。

* 生成 CA 证书

    CA 证书是用来签发其他 SSL 证书的密钥/证书对。在对 Koji 各组件进行配置时，客户端和服务端程序要用到的 CA 证书都来自这一步骤生成的 CA 证书。CA 证书放在 `/etc/pki/koji` 目录下，其他各组件的证书则放在 `/etc/pki/koji/certs` 目录下。`index.txt` 中会记录所有签发证书的相关信息。

        cd /etc/pki/koji/
        mkdir {certs,private,confs}
        touch index.txt
        echo 01 > serial
        openssl genrsa -out private/koji_ca_cert.key 2048
        openssl req -config ssl.cnf -new -x509 -days 3650 -key \
            private/koji_ca_cert.key -out koji_ca_cert.crt -extensions v3_ca

    上述最后一条命令会要求对证书相关的信息进行确认或者输入。其中 organizationUnitName 和 commonName 在生成不同组件的证书是需要进行相应调整。对于 CA 证书本身，这些信息没有特殊限制，建议使用使用服务器的 FQDN 作为 commonName 。

    如果使用配置管理工具来自动化证书生成，可以参考使用下面的命令：

        openssl req -config ssl.cnf -new -x509 \
        -subj "/C=CN/ST=Beijing/L=Beijing/O=Eayun/CN=koji.eayunci.eayun.com" \
        -days 3650 -key private/koji_ca_cert.key -out koji_ca_cert.crt \
        -extensions v3_ca

* 为 Koji 各组件以及管理用户生成证书

    Koji 每一个组件都需要单独的证书来标识自身。其中 kojihub 和 kojiweb 的证书是作为服务端证书需要通过客户端验证的，因此应将证书的 CN ( common name ) 配置为服务器的 FQDN，这样客户端程序就不会提示证书的 common name 和服务器主机名不匹配了。为了方便识别，可以将这2个证书的 OU （ organization name ）分别配置为 kojihub 和 kojiweb 。

    其他证书（如 kojira、kojid、管理用户以及其他用户）作为客户端证书，CN 只需要设置为这些对象用来登录的服务端的用户名即可，如 kojira 的证书 CN 应该设置为 kojira。原因是证书的 CN 需要和 koji 数据库中的用户名进行匹配。如果这些组件和服务端程序通信时，在数据库中找不到和证书 CN 对应的用户时，验证将无法通过，客户端访问将会被拒绝。

    当使用 `koji add-host` 命令添加构建主机时，会在 koji 数据库中添加一个用户（这个用户不会出现在用户列表中）。这个用户的名称需要和主机使用的证书的 CN 保持一致，否则构建主机将会无法访问 kojihub 服务器。而生成 kojiweb 的证书时，证书的所有相关信息（如，C、ST、L、O、CN 等）都需要加入到 /etc/koji-hub/hub.conf 中的 `ProxyDNs` 配置中去。

    要生成很多证书时将上述手动执行的命令编写为循环或者脚本是比较便捷的方式，下面是一个例子：

        #!/bin/bash
        # if you change your certificate authority name to something else you will
        # need to change the caname value to reflect the change.
        caname=koji

        # user is equal to parameter one or the first argument when you actually
        # run the script
        user=$1

        openssl genrsa -out private/${user}.key 2048
        cat ssl.cnf | sed 's/insert_hostname/'${user}'/'> ssl2.cnf
        openssl req -config ssl2.cnf -new -nodes -out certs/${user}.csr \
            -key private/${user}.key
        openssl ca -config ssl2.cnf -keyfile private/${caname}_ca_cert.key \
            -cert ${caname}_ca_cert.crt -out certs/${user}.crt -outdir certs \
            -infiles certs/${user}.csr
        cat certs/${user}.crt private/${user}.key > ${user}.pem
        mv ssl2.cnf confs/${user}-ssl.cnf

* 生成 PKCS12 用户证书 （供网页浏览器使用）

    PKCS12 用户证书是为了用户登录 kojiweb 的站点准备的。

        openssl pkcs12 -export -inkey private/${user}.key -in certs/${user}.crt \
            -CAfile ${caname}_ca_cert.crt -out certs/${user}_browser_cert.p12

* 将 kojiadmin （管理用户）的证书复制到用户的 `~/.koji` 目录之下

    管理用户使用 koji 命令行工具可以对 koji 系统进行管理。创建 kojiadmin 用户（koji 系统中的用户，并生成用户证书）之后，将 CA 证书 和 kojiadmin 用户证书复制到系统用户（普通用户也可以）的 `~/.koji` 目录之下：

        $ mkdir ~/.koji
        $ cp /etc/pki/koji/kojiadmin.pem ~/.koji/client.crt
        # 注意：用户证书需要用 PEM 文件而不是 CRT
        $ cp /etc/pki/koji/koji_ca_cert.crt ~/.koji/clientca.crt
        $ cp /etc/pki/koji/koji_ca_cert.crt ~/.koji/serverca.crt

    koji 命令行工具默认使用全局配置文件 /etc/koji.conf ，将该文件复制到 ~/.koji/ 目录下，并根据情况进行调整。

#### 准备 Kerberos 安全验证环境

Kerberos 环境的部署需要参考其他文档，和 Koji 系统相关的内容说明如下：

* DNS

    Koji 构建服务器（kojid）通过 DNS 查找特定 realm 中的 kerberos 服务器，如：

        _kerberos._udp  IN SRV 0 100 88 ipa-server.zhaochao.eayunos.eayun.com.

* Principal 和 Keytab

    需要注意的是，在为 Koji 各组件生成 keytab 时，需要使用组件所在服务器的 FQDN。

    以下 principal 需要导出到 keytab 文件中，其中 kojihub 要用到的 主机 principal 的名称部分（host/kojihub）目前是直接固定写在 koji 命令行工具的代码中的，不过更改。

    * host/kojihub@EXAMPLE.COM
        * kojihub 在和 koji 命令行工具通信时需要用到。
    * HTTP/kojiweb@EXAMPLE.COM
        * kojiweb 在和网络浏览器之间进行 kerberos 验证时需要用到，也是 apache 的 mod_auth_kerb 使用的服务 principal 。
    * koji/kojiweb@EXAMPLE.COM
        * kojiweb 在和 kojihub 通信时需要用到。这是一个用户 principal，即 kojiweb 在 kerberos 环境使用的是该用户。kojiweb 将会把 mod_auth_kerb 中用户信息（用户登录 Web 界面登录的情况）转发到 kojihub （ kojihub 配置文件中的 ProxyPrincipals 选项）。
    * koji/kojira@EXAMPLE.COM
        * kojira 和 kojihub 通信时使用。
    * compile/builder1@EXAMPLE.COM
        * 构建服务器 builder1 （服务器的主机名）和 kojihub 通信时使用。

* Kerberos 环境示例

    上面关于“准备 Kerberos 安全验证环境”的说明均来自官方的 wiki 页面，但是其实有不少含糊不清的地方，在实际部署的过程会碰到不少问题，下面做一些说明：

    测试的环境中使用 FreeIPA，对于 DNS 记录在初始化时已经建立，不用额外配置，但是由于测试环境只用了一台虚拟机，Koji 所有的组件都跑在该虚拟机上，因此 principal 建立就会产生不同。

    * 所有 Koji 组件运行在同一台服务器上，在 FreeIPA 中建立对应的 Principal

        * host/koji.zhaochao.eayunos.eayun.com@ZHAOCHAO.EAYUNOS.EAYUN.COM
        * HTTP/koji.zhaochao.eayunos.eayun.com@ZHAOCHAO.EAYUNOS.EAYUN.COM
        * kojiweb/koji.zhaochao.eayunos.eayun.com@ZHAOCHAO.EAYUNOS.EAYUN.COM
        * kojira/ipa-server.zhaochao.eayunos.eayun.com@ZHAOCHAO.EAYUNOS.EAYUN.COM
        * compile/koji.zhaochao.eayunos.eayun.com@ZHAOCHAO.EAYUNOS.EAYUN.COM

    也就是说 Principal 中除去前面的关键字前缀，在 realm 之前应该使用的是服务器的 FQDN。

### 初始化数据库
这里略去了 Postgres 数据库服务自身的初始化过程。

#### 创建 koji 系统用户

    # useradd koji
    # password koji

#### 建立数据库环境

分为以下几步：

* 建立 koji 数据库用户
* 建立 koji 数据库
* 为 koji 数据库用户设置密码
* 导入 koji 数据库的 schema

        root@localhost$ su - postgres
        postgres@localhost$ createuser koji
        Shall the new role be a superuser? (y/n) n
        Shall the new role be allowed to create databases? (y/n) n
        Shall the new role be allowed to create more new roles? (y/n) n
        postgres@localhost$ createdb -O koji koji
        postgres@localhost$ psql -c "alter user koji with encrypted password \
                                    'mypassword';"
        postgres@localhost$ logout
        root@localhost$ su - koji
        koji@localhost$ psql koji koji < /usr/share/doc/koji*/docs/schema.sql
        koji@localhost$ exit

#### 设置数据库允许 koji-web 和 koji-hub 访问

/var/lib/pgsql/data/pg_hba.conf ：

    #TYPE   DATABASE    USER    CIDR-ADDRESS      METHOD
    host    koji        koji    127.0.0.1/32      trust
    host    koji        koji     ::1/128          trust

如果服务器还有外部 IP 地址的话，可以再加上：

    host    koji        koji    $IP_ADDRESS/32    trust

如果要使用 UNIX socket 和数据库通信（这种方式下，Koji 组件的配置文件中 DBHost 选项不能进行设备），可以使用这样的配置：

    local   koji        apache                            trust
    local   koji        koji                              trust

**注意：** 如果强制要求使用密码验证访问数据，可以将上述配置中的 `trust` 改为 `md5`：

    #TYPE   DATABASE    USER    CIDR-ADDRESS      METHOD
    host    koji        koji    127.0.0.1/32      md5
    host    koji        koji     ::1/128          md5
    host    koji        koji    $IP_ADDRESS/32    md5

为了配置生效，让数据库重新读取配置文件：

    # service postgresql reload

#### 在数据库中添加 Koji 管理用户

Koji 管理用户需要手动通过 sql 语句添加到数据库中。当管理用户添加成功之后，其他的用户操作（增加、删除用户，修改用户权限等）可以通过 koji 命令完成。

不过，如果使用简单的用户名和密码验证方式，则配置或者修改用户密码仍然需要通过 sql 语句直接操作数据库完成，因为 koji 命令没有提供对用户密码进行操作的接口。

针对不同的验证方式，添加 Koji 管理用户的 sql 语句也不同：

* 用户名和密码验证：

        root@localhost$ su - koji
        koji@localhost$ psql
        koji=> insert into users (name, password, status, usertype)
        koji-> values ('admin-user-name', 'admin-password-in-plain-text', 0, 0);

* Kerberos 验证：

        root@localhost$ su - koji
        koji@localhost$ psql
        koji=> insert into users (name, krb_principal, status, usertype)
        koji-> values ('admin-user-name', 'admin@EXAMPLE.COM', 0, 0);

* SSL 证书验证：

        root@localhost$ su - koji
        koji@localhost$ psql
        koji=> insert into users (name, status, usertype)
        koji-> values ('admin-user-name', 0, 0);

#### 赋予指定用户管理权限

使用下面的 sql 语句可以为指定用户（需要知道用户的 ID）赋予管理管线：

    koji=> insert into user_perms (user_id, perm_id, creator_id)
    koji-> values (<用户 ID >, 1, <用户 ID >);

#### 让 Postgresql 数据库监听所有地址

* 编辑 `/var/lib/pgsql/data/postgresql.conf`
* 修改配置项，`listen_addresses = '*'`
* 重启数据库

        service postgresql restart

### 安装配置 Koji-hub

#### 安装 Koji-hub

    # yum install koji-hub httpd mod_ssl mod_auth_kerb

#### 基本配置

* /etc/httpd/conf/httpd.conf

    apache 有2处配置对最大可处理的请求数进行限制，koji-hub 的 xmlrpc 接口是一个 python 程序，可能由于没有及时回收内存而出现问题，因此强烈建议对2处 `MaxRequestsPerChild` 都进行限制（该值设为 100 时，httpd 大约在占用大约 75 MB 物理内存之后会创建新的进程）：

        <IfModule prefork.c>
        ...
        MaxRequestsPerChild  100
        </IfModule>
        <IfModule worker.c>
        ...
        MaxRequestsPerChild  100
        </IfModule>

* /etc/httpd/conf.d/kojihub.conf

    该文件需要针对安全验证方案进行一些调整，不过需要改动的地方不多，参阅配置文件中的注释信息即可。

* /etc/httpd/conf.d/ssl.conf

    * 如果使用 SSL 证书验证的话，需要指定证书文件所在位置，以及强制要求对客户端进行 SSL 证书限制，如：

            SSLCertificateFile /etc/pki/koji/certs/kojihub.crt
            SSLCertificateKeyFile /etc/pki/koji/certs/kojihub.key
            SSLCertificateChainFile /etc/pki/koji/koji_ca_cert.crt
            SSLCACertificateFile /etc/pki/koji/koji_ca_cert.crt
            SSLVerifyClient require
            SSLVerifyDepth  10

    * 如果使用 Kerberos 验证，则由于没有 SSL 证书，需要调整 SSLVerifyClient 配置项：

            SSLVerifyClient optional

* /etc/koji-hub/hub.conf

    该文件是 Koji-hub 的主要配置文件，首先需要数据库、KojiWebURL等基本信息。关于安全验证方面也在该文件中，后面将进行详细说明。

        DBName = koji
        DBUser = koji
        DBPass = mypassword
        DBHost = db.example.com
        KojiDir = /mnt/koji
        LoginCreatesUser = On
        KojiWebURL = http://kojiweb.example.com/koji

#### 安全验证配置

* /etc/koji-hub/hub.conf

    * 使用 Kerberos 验证时

            AuthPrincipal host/kojihub@EXAMPLE.COM
            AuthKeytab /etc/koji.keytab
            ProxyPrincipals koji/kojiweb@EXAMPLE.COM
            HostPrincipalFormat compile/%s@EXAMPLE.COM

    * 使用 SSL 证书验证时
    
        * mod_ssl 版本低于 2.3.11 的情况下：

                DNUsernameComponent = CN
                ProxyDNs = /C=US/ST=Massachusetts/O=Example Org/OU=kojiweb/CN=example/emailAddress=example@example.com

        * mod_ssl 版本不低于 2.3.11 的情况下：

                DNUsernameComponent = CN
                ProxyDNs = CN=example.com,OU=kojiweb,O=Example Org,ST=Massachusetts,C=US

#### Koji 文件目录

在上面的配置示例中，`KojiDir` 设置为 /mnt/koji，如果更改了该路径，仍然需要建立一个 /mnt/koji 软连接指向实际的 Koji 文件目录。在部署 Koji 其他组件之前需要为该目录建立基本的结构，并配置好权限：

    cd /mnt
    mkdir koji
    cd koji
    mkdir {packages,repos,work,scratch}
    chown apache.apache *

#### 配置 SELinux

如果服务器系统中 SELinux 以 Enforcing 模式运行，则进行一些配置：

* 允许 apache 访问 Postgresql 数据库；
* 允许 apache 在磁盘上写入某些文件。

配置方法参考：

    root@localhost$ setsebool -P httpd_can_network_connect_db=1 allow_httpd_anon_write=1
    root@localhost$ chcon -R -t public_content_rw_t /mnt/koji/*

如果 /mnt/koji 是挂载的 NFS 共享目录，则还需要设置 `httpd_use_nfs`：

    root@localhost$ setsebool -P httpd_use_nfs=1

#### 检查配置并重启 httpd

完成上述配置之后，理论上重新启动 apache 服务之后，Koji-hub 应该可以正常工作，可以通过下文即将提到的 koji 命令进行检测。

### koji 命令行工具
koji 命令行工具是 Koji 系统的标准客户端工具，使用 koji 命令可以完成大部分的工作。

koji 命令默认使用用户配置文件 ~/.koji/config 和全局配置文件 /etc/koji.conf 。

#### 配置 koji 命令使用 SSL 证书验证

主要的配置项是 `cert` 、 `ca` 、 `serverca`。举例如下：

    ; configuration for SSL athentication

    ;client certificate
    cert = ~/.koji/client.crt

    ;certificate of the CA that issued the client certificate
    ca = ~/.koji/clientca.crt

    ;certificate of the CA that issued the HTTP server certificate
    serverca = ~/.koji/serverca.crt

使用 SSL 验证时，url 地址可以直接使用 ip 地址（如，127.0.0.1）：

    ;;url of XMLRPC server
    server = http://127.0.0.1/kojihub

    ;;url of web interface
    weburl = http://127.0.0.1/koji

    ;;url of package download site
    topurl = http://127.0.0.1/kojifiles

#### 配置 koji 命令使用 Kerberos 验证

使用 Kerberos 验证时，koji 命令的配置文件其实不用做修改，只需要将 SSL 验证相关部分（如果曾经开启的话）注释即可，举例如下：

    [koji]
    server = http://koji.zhaochao.eayunos.eayun.com/kojihub

    weburl = http://koji.zhaochao.eayunos.eayun.com/koji

    topurl = http://koji.zhaochao.eayunos.eayun.com/kojifiles

    topdir = /mnt/koji

    ;the service name of the principal being used by the hub 
    ;krbservice = host

    ;client certificate
    ;cert = ~/.koji/client.crt
    
    ;certificate of the CA that issued the client certificate
    ;ca = ~/.koji/clientca.crt

    ;certificate of the CA that issued the HTTP server certificate
    ;serverca = ~/.koji/serverca.crt

但是使用 koji 命令之前必须首先完成 Kerberos 的身份验证，获取相关凭证，如：

    $ kinit kadmin
    Password for kojiadmin@ZHAOCHAO.EAYUNOS.EAYUN.COM:
    # klist
    Ticket cache: FILE:/tmp/krb5cc_0
    Default principal: kojiadmin@ZHAOCHAO.EAYUNOS.EAYUN.COM

    Valid starting     Expires            Service principal
    08/14/14 12:00:58  08/15/14 12:00:55  krbtgt/ZHAOCHAO.EAYUNOS.EAYUN.COM@ZHAOCHAO.EAYUNOS.EAYUN.COM
            renew until 08/21/14 12:00:55

#### 尝试登录 Koji-hub

使用 koji moshimoshi 命令测试是否可能正常登录到 Koji-hub：

    $ koji moshimoshi

### 安装配置 Koji-web

#### 安装 koji-web

    # yum install koji-web mod_ssl mod_auth_kerb

#### 基本配置

* /etc/httpd/conf.d/kojiweb.conf

    根据安全验证的类型进行调整，根据配置文件中提供的注释信息进行配置即可。

    * koji-web Kerberos 配置

        确保 `KrbServiceName` 、 `KrbAuthRealm` 、 `Krb5Keytab` 几项准确无误。

    * koji-web SSL 验证

        开启 /koji/login 的 SSL 验证。

* /etc/kojiweb/web.conf

    koji-web 的主要配置文件，需要指明 koji-hub 服务、koji 文件目录以及 kojiweb 自身的 url 地址，另外关于安全验证相关的配置也是必须的。

    配置文件中的 `Secret` 需要需要进行配置，指定一个特定的字符串即可。

    * 配置 Koji-web 使用 SSL 验证

            WebCert = /etc/pki/koji/kojiweb.pem
            ClientCA = /etc/pki/koji/koji_ca_cert.crt
            KojiHubCA = /etc/pki/koji/koji_ca_cert.crt

        其中 `WebCert` 对应的文件必须同时包含公钥和私钥信息。

    * 配置 Koji-web 使用 Kerberos 验证

            WebPrincipal = kojiweb/koji.zhaochao.eayunos.eayun.com@ZHAOCHAO.EAYUNOS.EAYUN.COM
            WebKeytab = /etc/koji.keytab
            WebCCache = /var/tmp/kojiweb.ccache

* 文件系统相关配置

    为了 Koji-web 可以正常工作，需要对 /mnt/koji 目录的访问权限进行修改，使得不同的组件都可以访问该目录。该目录会在下面这些配置文件中出现：

    * /etc/kojiweb/web.conf 中的 `KojiFilesURL`
    * /etc/kojid/kojid.conf 中的 `topurl`
    * /etc/koji.conf 中的 `topurl`

    apache 的配置如下（注意：针对 apache 的版本不同，访问控制配置的格式也有不同）

        Alias /kojifiles/ /mnt/koji/
        <Directory "/mnt/koji/">
            Options Indexes
            AllowOverride None
            # Apache < 2.4
            #   Order allow,deny
            #   Allow from all
            # Apache >= 2.4
            Require all granted
        </Directory>

#### 验证 Koji-web 是否可以正常工作

重启 httpd 服务之后，访问 koji-web 的地址，确认是否可以正常访问、登录以及完成各种操作。

### 安装配置Kojid

#### 安装 Kojid

    # yum install koji-builder

#### 基本配置

* 将构建主机的信息添加到数据库

    使用 koji add-host 命令可以将构建主机的信息添加到 Koji 数据库之后，需要注意的是这一步操作必须在构建主机上 kojid 服务启动之前进行；否则 kojid 启动之后，再想添加主机时，需要手动清理数据库中 sessions 和 users 表中的相关条目，才能成功。

        $ koji add-host kojibuilder1.example.com i386 x86_64

    `add-host` 之后第一个参数时 kojid 使用的用户名，第二部分是 kojid 执行构建任务时支持哪些 cpu 架构。

* /etc/kojid/kojid.conf

    kojid 的配置文件。首先需要明确指定 koji-hub 的 url 地址，以及 kojid 访问 koji-hub 使用的用户名：

        ; The URL for the xmlrpc server
        server=http://hub.example.com/kojihub

        ; the username has to be the same as what you used with add-host
        ; in this example follow as below
        user = kojibuilder1.example.com

    koji 文件目录也需要通过 http 进行访问：

        # The URL for the file access
        topurl=http://koji-filesystem.example.com/kojifiles

    kojid 的工作目录可以根据需要更改：

        ; The directory root for temporary storage
        workdir=/tmp/koji

    kojid 的构建根目录也必须该构建服务器上挂载。使用只读挂载的 NFS 目录是比较简单的方案。

        # The directory root where work data can be found from the koji hub
        topdir=/mnt/koji

#### 安全验证配置

* 使用 SSL 证书验证

    使用 SSL 证书验证时，要在配置文件中指明所有相关的证书文件。
    
    /etc/kojid/kojid.conf:
    
        ;client certificate
        ; This should reference the builder certificate we created on the kojihub CA, for kojibuilder1.example.com
        ; ALSO NOTE: This is the PEM file, NOT the crt
        cert = /etc/kojid/kojid.pem
    
        ;certificate of the CA that issued the client certificate
        ca = /etc/kojid/koji_ca_cert.crt
    
        ;certificate of the CA that issued the HTTP server certificate
        serverca = /etc/kojid/koji_ca_cert.crt

* 使用 Kerberos 验证

    /etc/kojid/kojid.conf:
    
        ; the username has to be the same as what you used with add-host
        ;user =
    
        host_principal_format=compile/%s@EXAMPLE.COM
    
    注意：要将 `user` 注释掉，`user` 配置项保留的情况下， kojid 不会使用 Kerberos 验证。
    
    默认 kojid 会从 /etc/kojid/kojid.keytab 中获取 Principal 信息然后和 koji-hub 进行通信，可以根据实际情况调整 keytab 文件位置：
    
        keytab = /etc/kojid/kojid.keytab

#### 配置通过 SCM 构建软件包（可选）

* /etc/kojid/kojid.conf

    SCM 地址的格式是： hostname:/path/match:checkout /common?, hostname2:/path/match:checkout /common?

    checkout /common 是一个布尔值，默认为 true，举例如下：

        allowed_scms=scm-server.example.com:/repo/base/repos*/:false

    使用上述配置时，当 kojid 把代码库 checkout 之后，会进行下面的操作：

        make sources
        mv *.spec <rpmbuild>/SPECS
        mv * <rpmbuild>/SOURCES
        rpmbuild -bs <rpmbuild>/SPECS/*.spec

#### 将主机添加到 createrepo 通道（channel）中

通道（channel） 是用来控制构建主机可以执行哪些构建任务的。默认情况下，所有构建主机都会加入到 `default` 通道。但是至少需要一些构建主机加入到 `createrepo` 通道，它们可以完成 kojira 发起的创建软件包仓库的任务。

    $ koji add-host-to-channel kojibuilder1.example.com createrepo

#### 关于构建主机容量（ capacity ）参数的说明

添加构建主机时，默认的容量为 2 ，含义是如果该构建主机的 load average 大于 2 时， kojid 将不再接受新的任务。该配置和 kojid 配置文件中的 `maxjobs` 配置项是不同的。当 kojid 接受新的任务时，需要满足系统 load average 小于 capacity，同时同时执行的任务数不能超过 maxjobs 。不过由于服务器硬件配置的不断增强，可以根据实际情况调整 capacity 的大小：

    koji edit-host --capacity=16 kojibuilder1.example.com

#### 启动 kojid

当构建主机添加到 Koji 系统，配置完成之后，可以启动 kojid 服务：

    # service kojid start

检查 kojid 的日志文件 `/var/log/kojid.log` 确认 kojid 是否已经启动成功。启动成功之后，可以通过 koji-web 的界面查看主机列表中各构建主机的状态和信息。

### 安装配置 Kojira

#### 安装 Kojira

    # yum install koji-utils

#### 基本配置

* 为 kojira 添加用户

    kojira 用户需要拥有 `repo` 权限：

        $ koji add-user kojira
        $ koji grant-permission repo kojira

    在 /etc/kojira/kojira.conf 中添加 koji-hub 的 url 地址：

        ; The URL for the xmlrpc server
        server=http://koji-hub.example.com/kojihub

* 其他说明：

    * kojira 对 /mnt/koji 要有读写权限；
    * kojira 只能同时运行一个示例；
    * 不建议将 kojira 和 kojid 放在相同的服务器，因为 kojid 对 /mnt/koji 只需要读权限；
    * 添加新的 tag 时，kojira 可能需要重启才能正确识别。

#### 安全验证配置

* 使用 SSL 证书验证

    /etc/kojira/kojira.conf 示例：

        ;client certificate
        ; This should reference the kojira certificate we created above
        cert = /etc/pki/koji/kojira.pem

        ;certificate of the CA that issued the client certificate
        ca = /etc/pki/koji/koji_ca_cert.crt

        ;certificate of the CA that issued the HTTP server certificate
        serverca = /etc/pki/koji/koji_ca_cert.crt

* 使用 Kerberos 验证

    /etc/kojira/kojira.conf 示例：

        ; For Kerberos authentication
        ; the principal to connect with
        principal=koji/kojira@EXAMPLE.COM
        ; The location of the keytab for the principal above
        keytab=/etc/kojira.keytab

* /etc/sysconfig/kojira

    kojira 程序对 /mnt/koji/repos 目录要有读写权限，如果该目录的权限无法调整（比如配置了 root_squash 的 NFS 共享目录等），可以在 /etc/sysconfig/kojira 中配置 RUNAS= 指定的用户来获取需要的权限。

#### 启动 Kojira

    $ service kojira start

    检查 kojira 日志 `/var/log/kojira.log` 确认是否已经正常启动。

## 使用 Koji

Koji 提供的功能不少，根据不同的环境、不同的需求可以设计不同的系统方案，这里只对最基本的使用进行介绍。

### 确定是否使用外部软件仓库

Koji 构建软件包时使用 mock，构建过程中利用了 chroot 环境，因此涉及到构建过程依赖的软件包如何安装的问题。有 2 种情况：

* 导入已有的软件包

    使用 koji import 命令可以将已有的 srpm 和 rpm 包导入到 Koji 系统，其他软件包基于这些软件包进行构建；

* 使用外部软件仓库

    使用 koji add-external-repo 命令添加外部软件仓库，构建软件包时依赖的软件包从外部仓库中获取并安装。

使用哪一种方式要根据具体的需求决定，后面的文档将基于使用外部软件仓库的情形进行说明。

### 建立 tag 和 target

tag 是用来建立 Koji 工作流程的，对于最建单的情况，需要建立软件包最终发布所属的 tag ：

    $ koji add-tag EayunOS-4-0

接下来为具体的构建过程创建 tag ：

    $ koji add-tag --parent EayunOS-4-0 --arches "i686 x86_64" EayunOS-4-0-build

Koji 构建软件包时使用的是 target，target 决定了构建过程在何处进行（构建过程对应的 tag ），以及构建产生的结果如何标记（目标 tag），所以需要继续建立 target ：

    $ koji add-target EayunOS-4-0 EayunOS-4-0-build

使用 koji add-target 时，最多可以使用 3 个参数，分别对应 target 名称、构建过程对应的 tag、目标 tag。在上面的示例中，最后的目标 tag 省略了，这时目标 tag 默认使用和 target 名称相同的 tag ，即，上述命令中 EayunOS-4-0 是 target，该命令其实表示的是：

    $ koji add-target EayunOS-4-0 EayunOS-4-0-build EayunOS-4-0

### 添加外部软件仓库

外部软件仓库需要添加到构建过程对应的 tag ：

    $ koji add-external-repo -t EayunOS-4-0-build centos6-base-repo \
    http://192.168.3.239/mirrors/CentOS/6.5/os/$arch/

    $ koji add-external-repo -t EayunOS-4-0-build centos6-update-repo \
    http://192.168.3.239/mirrors/CentOS/6.5/updates/$arch/

### 创建构建相关的软件包组

在 Koji 系统中可以对软件包进行分组，其中有 2 个比较特殊的软件包组： `build` 和 `srpm-build` 。Koji 的构建过程使用 chroot 环境，建立该 chroot 环境时需要要装一些基本的软件包才能完成构建工作。根据构建任务的不同（这里只说明 rpm 和 source rpm 的情况），构建 rpm 包时，建立 chroot 环境的过程中会安装 build 软件包组的所有软件包；构建 source rpm 包时，建立 chroot 环境的过程中会安装 srpm-build 软件包组中的所有软件包。

因此必须首先建立这 2 个软件组，同时软件组中的软件包选择也需要注意。因此 koji 每次构建都会新建 chroot 环境，如果要安装太多的软件包，会影响整个构建过程的时间，当然如果构建过程中需要的软件包不会在创建 chroot 环境时安装（而且也不在该软件包的依赖关系中）则构建过程会失败。

#### 创建 build 和 srpm-build 软件包组

    $ koji add-group EayunOS-4-0-build build

    $ koji add-group EayunOS-4-0-build srpm-build

#### 为 build 和 srpm-build 添加软件包

    $ koji add-group-pkg EayunOS-4-0-build build bash bzip2 coreutils diffutils \
    findutils gawk gcc grep gzip make patch rpm-build shadow-utils tar \
    util-linux-ng which

    $ koji add-group-pkg EayunOS-4-0-build srpm-build bash bzip2 gzip rpm-build \
    rpmdevtools shadow-utils tar

#### 为构建过程 tag 重建 repo 信息

    $ koji regen-repo EayunOS-4-0-build

### 测试构建环境是否可用

koji build 命令会创建并启动软件包构建任务，加上 --scratch 参数可以用来测试构建任务，而不会干扰正常的 koji 工作流程。

    $ koji build --scratch eayunos-release-4-0.el6.2.src.rpm

### 开始构建软件包

首先将软件包添加到目标 tag 的软件包列表（注意只需要软件包名称即可，即 Koji 架构术语中的 Package ）：

    $ koji add-pkg --owner zhaochao EayunOS-4-0 eayunos-release

然后就可以开始构建软件包了

    $ koji build EayunOS-4-0 eayunos-release-4-0.el6.2.src.rpm

### 从 SCM 代码仓库构建软件包

除了只是通过 srpm 构建软件包之外，Koji 系统还是从 SCM 代码仓库中构建软件包。要从 SCM 代码仓库中构建软件包，首先需要对 kojid 进行配置，允许 kojid 从指定的 SCM 地址构建，配置方法见 kojid 的安装配置部分。

举例说明：

/etc/kojid/kojid.conf

     allowed_scms=192.168.3.239:*:no

koji build 从 SCM 代码仓库构建软件包的命令中，url 地址的格式是：

    scheme://[user@]host/path/to/repo?path/to/module#revision_or_tag_identifier

举例说明：

    koji build --scratch EayunOS-4-0 'git+http://192.168.3.239/git/virt-viewer-win-koji.git?.#HEAD'

需要注意的是，url 地址中除了代码仓库的地址以外需要指定代码所在的目录以及代码 revision 。另外地址的协议头也有特定的格式，目前支持以下组合：

    cvs://， cvs+ssh://
    git://， git+http://， git+https://， git+rsync://， git+ssh://
    svn://， svn+http://， svn+https://， svn+ssh://

在 kojid 安装配置部分已经说过代码从 SCM 服务器 checkout 之后，会自动完成一些操作，其中第一步是生成构建 srpm 需要的压缩包。 kojid 默认执行的命令是 make source，当然可以通过配置让 kojid 执行其他的命令，比如 `fedpkg sources`，那么上例就改成：

    allowed_scms=192.168.3.239:*:no:fedpkg,sources

默认的情况下 kojid 执行 make sources ，因此需要在代码所在的目录下编写一个 Makefile，例如如：

    package = virt-viewer-win
    src_pkgs = $(shell spectool -l $(package).spec | awk '{print $$NF}')
    
    sources:
            tar zcf $(src_pkgs) virt-viewer-win

## 原文链接

1. [koji - RPM building and tracking system](https://fedorahosted.org/koji/wiki)
2. [Koji wiki](https://fedoraproject.org/wiki/Koji)
3. [Koji/ServerHowTo](https://fedoraproject.org/wiki/Koji/ServerHowTo)
4. [devops-blog 上的 koji系列博客](http://www.devops-blog.net/category/koji)
