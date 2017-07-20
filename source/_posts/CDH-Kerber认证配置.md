---
title: CDH-Kerber认证配置
description: Cloudera大数据平台kerberos认证配置。
tags: [hadoop生态圈, kerberos]
categories: hadoop
date: 2017-7-19 10:48:04
---

## 环境说明

| 软件名称 | 软件版本 | 软件描述 |
| ------ | ------ | ------ |
| CDH | 5.11.0 | hadoop企业发行版 |
| Centos | 7.1-1503 | 集群操作系统 |
| Jdk | 1.7.0_67 | java运行环境 |
| kerberos | 1.14 | kerberos认证服务 |

<font color="red">请严格按照以上软件版本要进行操作，不要使用大小写混合的主机名。</font>

## 安装kerberos

选一台<font color="red">主节点</font>安装服务：
```
yum install krb5-server -y
```

在其它子节点安装krb5-devel、krb5-workstation：
```
yum install krb5-devel krb5-workstation -y
```

### 安装JCE

安装JCE包需要下载[Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files for JDK/JRE 7](http://7xoqbc.com1.z0.glb.clouddn.com/UnlimitedJCEPolicyJDK7.zip) 
解压之后放到到$JAVA_HOME/jre/lib/security目录

所有机器必须安装。

### 修改配置文件

kdc服务器包含三个配置文件：
```
# 集群上所有节点都有这个文件而且内容同步
/etc/krb5.conf
# 主服务器上的kdc配置
/var/kerberos/krb5kdc/kdc.conf
# 能够不直接访问 KDC 控制台而从 Kerberos 数据库添加和删除主体，需要添加配置
/var/kerberos/krb5kdc/kadm5.acl
```

修改/etc/krb5.conf为以下内容：
```
[libdefaults]
default_realm = BIGMAN.COM
dns_lookup_kdc = false
dns_lookup_realm = false
ticket_lifetime = 86400
renew_lifetime = 604800
forwardable = true
default_tgs_enctypes = aes256-cts-hmac-sha1-96
default_tkt_enctypes = aes256-cts-hmac-sha1-96
permitted_enctypes = aes256-cts-hmac-sha1-96
udp_preference_limit = 1
kdc_timeout = 3000
[realms]
BIGMAN.COM = {
kdc = bigman-m1
admin_server = bigman-m1
}
```

`bigman-m1`为kdc服务主机hostname


配置项说明：
```
[logging]：日志输出设置
[libdefaults]：连接的默认配置
default_realm：Kerberos应用程序的默认领域，所有的principal都将带有这个领域标志
ticket_lifetime： 表明凭证生效的时限，一般为24小时
renew_lifetime： 表明凭证最长可以被延期的时限，一般为一个礼拜。当凭证过期之后，对安全认证的服务的后续访问则会失败
clockskew：时钟偏差是不完全符合主机系统时钟的票据时戳的容差，超过此容差将不接受此票据。通常，将时钟扭斜设置为 300 秒（5 分钟）。这意味着从服务器的角度看，票证的时间戳与它的偏差可以是在前后 5 分钟内
udp_preference_limit= 1：禁止使用 udp 可以防止一个 Hadoop 中的错误
[realms]：列举使用的 realm
kdc：代表要 kdc 的位置。格式是 机器:端口
admin_server：代表 admin 的位置。格式是 机器:端口
default_domain：代表默认的域名
[domain_realm]：域名到realm的关系
```

修改/var/kerberos/krb5kdc/kdc.conf：
```
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 BIGMAN.COM = {
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
 }
```

配置项说明：
```
kdcdefaults：kdc相关配置，这里只设置了端口信息
realms：realms的配置
XIAOHEI.INFO：设定的realms领域
master_key_type：和 supported_enctypes 默认使用 aes256-cts。JAVA 使用 aes256-cts 验证方式需要安装 JCE 包
acl_file：标注了 admin 的用户权限，文件格式是：Kerberos_principal permissions [target_principal] [restrictions]
supported_enctypes：支持的校验方式
admin_keytab：KDC 进行校验的 keytab
```

创建/var/kerberos/krb5kdc/kadm5.acl，内容为：
```
*/admin@EXAMPLE.COM *
```

创建Kerberos数据库：
```
# 保存路径为/var/kerberos/krb5kdc 如果需要重建数据库，将该目录下的principal相关的文件删除即可
kdb5_util create -r BIGMAN.COM -s
```

该命令会在/var/kerberos/krb5kdc/目录下创建 principal数据库

### 启动服务

在主节点上执行：
```
systemctl enable krb5kdc
systemctl enable kadmin
systemctl start krb5kdc
systemctl start kadmin
```

### 创建Kerberos管理员

**需要设置两次密码，当前设置为root**
kadmin.local -q "addprinc cloudera-scm/admin"

principal的名字的第二部分是admin,那么该principal就拥有administrative privileges这个账号将会被CDH用来生成其他用户/服务的principal


### 测试Kerberos

```
# 列出Kerberos中的所有认证用户，即principals
kadmin.local -q "list_principals"
# 添加认证用户，需要输入密码
kadmin.local -q "addprinc user1"
# 使用该用户登录，获取身份认证，需要输入密码
kinit user1
# 查看当前用户的认证信息ticket
klist
# 更新ticket
kinit -R
# 销毁当前的ticket
kdestroy
# 删除认证用户
kadmin.local -q "delprinc user1"
```

### 配置文件同步

将主节点配置的/etc/krb5.conf和/var/kerberos/krb5kdc/kdc.conf两个文件，通过scp命令复制到其它所有子节点机器上。


## CDH开启kerberos认证

参考文档：https://www.cloudera.com/documentation/enterprise/latest/topics/cm_sg_intro_kerb.html#xd_583c10bfdbd326ba--6eed2fb8-14349d04bee--76dd

通过CDH Manager向导启用kerberos。


## 命令行帮助

### 导出keytab命令

先通过下面命令将CDH默认创建的用户票据信息，导出成keytab文件。


抽取密钥并将其储存在本地 keytab 文件 /etc/krb5.keytab 中。这个文件由超级用户拥有，所以必须是 root 用户才能在 kadmin shell 中执行以下命令：
```
# 进入kadmin命令行
$ kadmin.local
kadmin.local:  ktadd -k /root/hbase.keytab -q hbase/bigman-m2@BIGMAN.COM

# 查看生成的keytab
$ klist -k /root/hbase.keytab 
```

kadmin语法：`kadmin: ktadd [-e enctype] [-k keytab] [-q] [principal | -glob principal-exp]`
`-e enctype`
覆盖 krb5.conf 文件中定义的加密类型列表。
`-k keytab`
指定密钥表文件。缺省情况下，使用 /etc/krb5.keytab。
`-q`
显示简要信息。
`principal`
指定要添加至密钥表文件的主体。可以添加以下服务主体：host、root、nfs 和 ftp。
`-glob principal-exp`
指定主体表达式。与 principal-exp 匹配的所有主体都将添加至密钥表文件。主体表达式的规则与 kadmin 的 list_principals 命令的规则相同。


### Hadoop平台操作命令参考

**查看所有票据信息**
```
kadmin.local -q "list_principals"
```

**删除认证用户**
```
kadmin.local -q "delprinc hadoop@BIGMAN.COM"
```

**hdfs初始化票据**
```
kinit bumblebee/bigman-m2@BIGMAN.COM
```

**hbase初始化票据**
```
kinit -kt /var/kerberos/keytab/hbase.keytab hbase/bigman-m2@BIGMAN.COM
hbase shell
```

**hive**
```
kinit -kt /var/kerberos/keytab/hive.keytab hive/bigman-m2@BIGMAN.COM
beeline -u "jdbc:hive2://bigman-m2:10000/default;principal=hive/_HOST@BIGMAN.COM"
```

**hive on hbase这样也行**
```
kinit -kt /var/kerberos/keytab/hbase.keytab hbase/bigman-m2@BIGMAN.COM
hive HIVE_OPTS="-hiveconf hbase.security.authentication=kerberos -hiveconf hbase.master.kerberos.principal=hbase/_HOST@BIGMAN.COM -hiveconf hbase.regionserver.kerberos.principal=hbase/_HOST@BIGMAN -hiveconf hbase.zookeeper.quorum=bigman-s1,bigman-s2,bigman-s3"
```

**hive on hbase授权hive用户**

进入hbase shell，授权用户权限（执行一次即可）：
```
kinit -kt /var/kerberos/keytab/hbase.keytab hbase/bigman-m2@BIGMAN.COM
hbase shell
grant  'hive', 'RWCA'   #授权hive用户有rwca权限
```

参考：https://discuss.pivotal.io/hc/en-us/articles/201998166-HBase-SQL-statement-fails-with-Insufficient-permissions-for-user

**hive on hbase授权impala用户**
```
kinit -kt /var/kerberos/keytab/hbase.keytab hbase/bigman-m2@BIGMAN.COM
hbase shell
grant  'impala', 'RWCA'
```


**sentry必须所有机器都有权限的用户和用户组，否则报错**

授权操作：
```
kinit -kt /var/kerberos/keytab/hive.keytab hive/bigman-m2@BIGMAN.COM
beeline -u "jdbc:hive2://bigman-m1:10000/;principal=impala/_HOST@BIGMAN.COM"

create role admin_role;
GRANT ALL ON SERVER server1 TO ROLE admin_role;
GRANT ROLE admin_role TO GROUP hive;

drop role test_role;
create role test_role;
GRANT ALL ON table default.test TO ROLE test_role;
GRANT ROLE test_role TO GROUP hadoop;

drop role test_role;
create role test_role;
GRANT ALL ON database default TO ROLE test_role;
GRANT ROLE test_role TO GROUP test;
```


### 创建管理员权限用户

#### 创建系统用户和组

所有机器创建`bigman`用户以及`supergroup`组

```
groupadd supergroup
useradd -g supergroup bigman
```

supergroup代表有hdfs管理员权限，可以根据hadoop配置参数`dfs.permissions.supergroup, dfs.permissions.superusergroup`修改。
由于impala、impala验证的是操作系统的用户，不能使用虚拟用户，所以得创建系统级的用户和组

#### 创建bigman票据

在kerberos服务器执行：
```
kadmin.local -q "addprinc bigman/bigman-m1@BIGMAN.COM"
```

输入密码，回车。

#### 导出keytab

```
kadmin.local
xst -k /var/kerberos/keytab/bigman.keytab bigman/bigman-m1@BIGMAN.COM
```

导出keytab文件，为了不需要使用密码也能使用该票据。

#### 登录hive，授权给bigman管理员权限

```
kinit -kt /var/kerberos/keytab/hive.keytab hive/bigman-m2@BIGMAN.COM
beeline -u "jdbc:hive2://bigman-m2:10000/default;principal=hive/_HOST@BIGMAN.COM"

#赋予admin角色
GRANT ROLE admin_role TO GROUP supergroup;  
```

`admin_role`角色在之前已经创建好了的。

#### ##登录hbase，授权给bigman权限
```
kinit -kt /var/kerberos/keytab/hbase.keytab hbase/bigman-m2@BIGMAN.COM
hbase shell
grant  'bigman', 'RWCA'
```

#### 验证
```
kinit bigman/bigman-m1   #输入密码
beeline -u "jdbc:hive2://bigman-m2:10000/default;principal=hive/_HOST@BIGMAN.COM"
```



## Windows下通过浏览器访问Hadoop web应用

### 环境初始化

1、安装jdk，配置了JAVA_HOME和PATH环境变量
2、根据上面章节，安装JCE
3、下载并安装[kfw-4.0.1-amd64.msi](http://7xoqbc.com1.z0.glb.clouddn.com/kfw-4.0.1-amd64.msi)
4、将krb5.ini文件放到C:\Windows和C:\ProgramData\MIT\Kerberos5目录：
krb5.init
```
[libdefaults]
default_realm = BIGMAN.COM
dns_lookup_kdc = false
dns_lookup_realm = false
ticket_lifetime = 86400
renew_lifetime = 604800
forwardable = true
default_tgs_enctypes = aes256-cts-hmac-sha1-96
default_tkt_enctypes = aes256-cts-hmac-sha1-96
permitted_enctypes = aes256-cts-hmac-sha1-96
udp_preference_limit = 1
kdc_timeout = 3000
[realms]
BIGMAN.COM = {
kdc = bigman-m1
admin_server = bigman-m1
}
```

### 访问hdfs

浏览器打开地址：http://bigman-m1:50070/，直接报错：
![kerberos-hdfs-error](http://7xoqbc.com1.z0.glb.clouddn.com/kerberos-req-error1.png)

报错就对了。。。

### 开启浏览器SPNEGO认证

完成以下步骤以确保已启用 Microsoft Internet Explorer 浏览器来执行 SPNEGO 认证。

1、激活 Internet Explorer。
2、在 Internet Explorer 窗口中，单击**工具 > Internet 选项 > 安全**选项卡。
3、选择**本地 Intranet** 图标并单击**站点**。
4、在“本地 Intranet”窗口中，确保选中了“包括没有列在其他区域的所有本地 (Intranet) 站点”复选框，然后单击**高级**。
5、在**本地 Intranet窗口**中，使用主机名的 Web 地址填写“将此 Web 站点添加到区域中”字段，以便可以对“Web 站点”字段中显示的 Web 站点列表启用单点登录 (SSO)。您的站点信息技术人员提供此信息。单击确定以完成此步骤并关闭“本地 Intranet”窗口。
6、在 **Internet 选项**窗口，单击**高级**选项卡并滚动到**安全设置**。确保选中了**启用集成 Windows 认证（需要重新启动）**框。
单击确定。重新启动 Microsoft Internet Explorer 以激活此配置。


## 激活Firefox 浏览器

1、激活 Firefox。
2、在“地址”字段中，输入 about:config。
3、在“过滤器”中，输入 network.n
4、双击network.negotiate-auth.trusted-uris。此首选项列示那些被允许向浏览器执行 SPNEGO 认证的站点。输入用逗号定界的可信域或 URL 的列表。
Note
必须设置 network.negotiate-auth.trusted-uris 的值。
5、双击network.auth.use-sspi设置为false。
6、单击确定。配置表现为已更新。
7、重新启动 Firefox 浏览器以激活此配置。

配置network.negotiate-auth.trusted-uris：
![firefox-kerberos-config1](http://7xoqbc.com1.z0.glb.clouddn.com/kerberos-config1.png)

配置network.auth.use-sspi：
![firefox-kerberos-config2](http://7xoqbc.com1.z0.glb.clouddn.com/kerberos-config2.png)

### 最后一步，登录Kerberos

打开MIT Kerberos Ticket Manager软件，使用bumblebee/bigman-m2登录（密码是123456），具体操作见下图：

![firefox-kerberos-config3](http://7xoqbc.com1.z0.glb.clouddn.com/kerberos-config3.png)

登录成功之后：
![firefox-kerberos-config4](http://7xoqbc.com1.z0.glb.clouddn.com/kerberos-config4.png)

成功访问：
![firefox-kerberos-success](http://7xoqbc.com1.z0.glb.clouddn.com/kerberos-config5.png)

注：<font color="red">本次测试采用firefox浏览器，其它浏览器访问请参考[这里](https://www.cloudera.com/documentation/enterprise/latest/topics/cdh_sg_browser_access_kerberos_protected_url.html)。</font>


## 最后

本文中提到的主机名，请换成自己环境相对应的主机。kerberos票据创建等命令可参考前面章节。


## 参考

- https://www.ibm.com/support/knowledgecenter/zh/SSAW57_8.5.5/com.ibm.websphere.nd.iseries.doc/ae/tsec_SPNEGO_config_web.html
- http://community.cloudera.com/t5/Security-Apache-Sentry/Cant-access-secured-web-UI-Solr-UI-GSSException-Defective-token/m-p/45470