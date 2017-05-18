---
title: Hive/impala进阶-LDAP用户登录控制
description: 一共提供了三种安全认证方式：</br>LDAP Authentication using OpenLDAP</br>Setting up Authentication with Pluggable Access Modules</br>Configuring Custom Authentication</br>本次采用第一种OpenLDAP方式，下面介绍如何自定义用户名密码验证。
tags: [hadoop生态圈, hive]
categories: hive
date: 2017-5-16 09:30:07
---

## 背景

一共提供了三种安全认证方式：
[LDAP Authentication using OpenLDAP](http://doc.mapr.com/display/MapR/Using+HiveServer2#UsingHiveServer2-LDAPAuthenticationusingOpenLDAP)
[Setting up Authentication with Pluggable Access Modules](http://doc.mapr.com/display/MapR/Using+HiveServer2#UsingHiveServer2-SettingupAuthenticationwithPluggableAccessModules)
[Configuring Custom Authentication](http://doc.mapr.com/display/MapR/Using+HiveServer2#UsingHiveServer2-ConfiguringCustomAuthentication)

本次采用第一种OpenLDAP方式，下面介绍如何自定义用户名密码验证。

笔者使用hive1.1版本作为测试。

## 安装OpenLDAP

选择一台机器安装 openldap：
```
$ yum install db4 db4-utils db4-devel cyrus-sasl* krb5-server-ldap -y
$ yum install openldap openldap-servers openldap-clients openldap-devel compat-openldap -y
```

查看安装的版本：
```
$ rpm -qa openldap
openldap-2.4.39-8.el6.x86_64

$ rpm -qa krb5-server-ldap
krb5-server-ldap-1.10.3-33.el6.x86_64
```

## LDAP 服务端配置

更新配置库：
```
rm -rf /var/lib/ldap/*
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
chown -R ldap.ldap /var/lib/ldap
```

在2.4以前的版本中，OpenLDAP 使用 slapd.conf 配置文件来进行服务器的配置，而2.4开始则使用 slapd.d 目录保存细分后的各种配置，这一点需要注意，其数据存储位置即目录 /etc/openldap/slapd.d 。尽管该系统的数据文件是透明格式的，还是建议使用 ldapadd, ldapdelete, ldapmodify 等命令来修改而不是直接编辑。

默认配置文件保存在 /etc/openldap/slapd.d，将其备份：
```
cp -rf /etc/openldap/slapd.d /etc/openldap/slapd.d.bak
```

添加一些基本配置，并引入 kerberos 和 openldap 的 schema：
```
$ cp /usr/share/doc/krb5-server-ldap-1.10.3/kerberos.schema /etc/openldap/schema/

$ touch /etc/openldap/slapd.conf

$ echo "include /etc/openldap/schema/corba.schema
include /etc/openldap/schema/core.schema
include /etc/openldap/schema/cosine.schema
include /etc/openldap/schema/duaconf.schema
include /etc/openldap/schema/dyngroup.schema
include /etc/openldap/schema/inetorgperson.schema
include /etc/openldap/schema/java.schema
include /etc/openldap/schema/misc.schema
include /etc/openldap/schema/nis.schema
include /etc/openldap/schema/openldap.schema
include /etc/openldap/schema/ppolicy.schema
include /etc/openldap/schema/collective.schema
include /etc/openldap/schema/kerberos.schema" > /etc/openldap/slapd.conf
$ echo -e "pidfile /var/run/openldap/slapd.pid\nargsfile /var/run/openldap/slapd.args" >> /etc/openldap/slapd.conf

#更新slapd.d
$ slaptest -f /etc/openldap/slapd.conf -F /etc/openldap/slapd.d

$ chown -R ldap:ldap /etc/openldap/slapd.d && chmod -R 700 /etc/openldap/slapd.d
```

## 启动服务

启动 LDAP 服务：
```
chkconfig --add slapd
chkconfig --level 345 slapd on

/etc/init.d/slapd start
```

可能会报错，根据提示将对应文件所属者修改成ldap用户即可。

查看状态，验证服务端口：
```
$ ps aux | grep slapd | grep -v grep
  ldap      9225  0.0  0.2 581188 44576 ?        Ssl  15:13   0:00 /usr/sbin/slapd -h ldap:/// -u ldap

$ netstat -tunlp  | grep :389
  tcp        0      0 0.0.0.0:389                 0.0.0.0:*                   LISTEN      8510/slapd
  tcp        0      0 :::389                      :::*                        LISTEN      8510/slapd
```

如果启动失败，则运行下面命令来启动 slapd 服务并查看日志：
```
$ slapd -h ldap://127.0.0.1 -d 481
```


## 创建数据库

进入到 /etc/openldap/slapd.d 目录，查看 /etc/openldap/slapd.d/cn\=config/olcDatabase={2}bdb.ldif 可以看到一些默认的配置，例如：
```
olcRootDN: cn=Manager,dc=my-domain,dc=com  
olcRootPW: secret  
olcSuffix: dc=my-domain,dc=com
```

接下来更新这三个配置，建立 modify.ldif 文件，内容如下：
```
dn: olcDatabase={2}bdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=javachen,dc=com

dn: olcDatabase={2}bdb,cn=config
changetype: modify
replace: olcRootDN
# Temporary lines to allow initial setup
olcRootDN: uid=ldapadmin,ou=people,dc=javachen,dc=com

dn: olcDatabase={2}bdb,cn=config
changetype: modify
add: olcRootPW
olcRootPW: secret

dn: cn=config
changetype: modify
add: olcAuthzRegexp
olcAuthzRegexp: uid=([^,]*),cn=GSSAPI,cn=auth uid=$1,ou=people,dc=javachen,dc=com

dn: olcDatabase={2}bdb,cn=config
changetype: modify
add: olcAccess
# Everyone can read everything
olcAccess: {0}to dn.base="" by * read
# The ldapadm dn has full write access
olcAccess: {1}to * by dn="uid=ldapadmin,ou=people,dc=javachen,dc=com" write by * read
```

说明：

- 上面的密码使用的是明文密码 secret ，你也可以使用 slappasswd -s secret 生成的字符串作为密码。
- 上面的权限中指明了只有用户 uid=ldapadmin,ou=people,dc=javachen,dc=com 有写权限。

使用下面命令导入更新配置：
```
$ ldapmodify -Y EXTERNAL -H ldapi:/// -f modify.ldif
```

这时候数据库没有数据，需要添加数据，你可以手动编写 ldif 文件来导入一些用户和组，或者使用 migrationtools 工具来生成 ldif 模板。创建 setup.ldif 文件如下：
```
dn: dc=javachen,dc=com
objectClass: top
objectClass: dcObject
objectclass: organization
o: javachen com
dc: javachen

dn: ou=people,dc=javachen,dc=com
objectclass: organizationalUnit
ou: people
description: Users

dn: ou=group,dc=javachen,dc=com
objectClass: organizationalUnit
ou: group

dn: uid=ldapadmin,ou=people,dc=javachen,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: LDAP admin account
uid: ldapadmin
sn: ldapadmin
uidNumber: 1001
gidNumber: 100
homeDirectory: /home/ldap
loginShell: /bin/bash
```

使用下面命令导入数据，密码是前面设置的 secret：
```
$ ldapadd -x -D "uid=ldapadmin,ou=people,dc=javachen,dc=com" -w secret -f setup.ldif
```

参数说明：

- `-w` 指定密码
- `-x` 是使用一个匿名绑定


## LDAP 的使用

### 导入系统用户

接下来你可以从 /etc/passwd, /etc/shadow, /etc/groups 中生成 ldif 更新 ldap 数据库，这需要用到 migrationtools 工具。

安装：
```
$ yum install migrationtools -y
```

利用迁移工具生成模板，先修改默认的配置：
```
$ vim /usr/share/migrationtools/migrate_common.ph

#line 71 defalut DNS domain
$DEFAULT_MAIL_DOMAIN = "javachen.com";
#line 74 defalut base
$DEFAULT_BASE = "dc=javachen,dc=com";
```

生成模板文件：
```
/usr/share/migrationtools/migrate_base.pl > /opt/base.ldif
```

然后，可以修改该文件，然后执行导入命令：
```
$ ldapadd -x -D "uid=ldapadmin,ou=people,dc=javachen,dc=com" -w secret -f /opt/base.ldif
```

将当前节点上的用户导入到 ldap 中，可以有选择的导入指定的用户：
```
# 先添加用户
$ useradd test hive
# 查找系统上的 test、hive 等用户
$ grep -E "test|hive" /etc/passwd  >/opt/passwd.txt
$ /usr/share/migrationtools/migrate_passwd.pl /opt/passwd.txt /opt/passwd.ldif
$ ldapadd -x -D "uid=ldapadmin,ou=people,dc=javachen,dc=com" -w secret -f /opt/passwd.ldif
```

将用户组导入到 ldap 中：
```
# 生成用户组的 ldif 文件，然后导入到 ldap
$ grep -E "test|hive" /etc/group  >/opt/group.txt
$ /usr/share/migrationtools/migrate_group.pl /opt/group.txt /opt/group.ldif
$ ldapadd -x -D "uid=ldapadmin,ou=people,dc=javachen,dc=com" -w secret -f /opt/group.ldif
```

### 查询

查询新添加的 test 用户：
```
$ ldapsearch -LLL -x -D 'uid=ldapadmin,ou=people,dc=javachen,dc=com' -w secret -b 'dc=javachen,dc=com' 'uid=test'
  dn: uid=test,ou=people,dc=javachen,dc=com
  objectClass: inetOrgPerson
  objectClass: posixAccount
  objectClass: shadowAccount
  cn: test account
  sn: test
  uid: test
  uidNumber: 1001
  gidNumber: 100
  homeDirectory: /home/test
  loginShell: /bin/bash
```

可以看到，通过指定 'uid=test'，我们只查询这个用户的数据，这个查询条件叫做filter。有关 filter 的使用可以查看 ldapsearch 的 manpage。

### 修改

用户添加好以后，需要给其设定初始密码，运行命令如下：
```
$ ldappasswd -x -D 'uid=ldapadmin,ou=people,dc=javachen,dc=com' -w secret "uid=test,ou=people,dc=javachen,dc=com" -S
```

比如输入'test'作为密码，后面登录都使用这个密码。

### 删除

删除用户或组条目：
```
$ ldapdelete -x -w secret -D 'uid=ldapadmin,ou=people,dc=javachen,dc=com' "uid=test,ou=people,dc=javachen,dc=com"
$ ldapdelete -x -w secret -D 'uid=ldapadmin,ou=people,dc=javachen,dc=com' "cn=test,ou=group,dc=javachen,dc=com"
```


### 导入虚拟用户

创建并进入临时目录：
```
$ mkdir /tmp/ldap
$ cd /tmp/ldap
```

创建passwd.txt文件，并写入下面内容：
```
renhua:x:503:504::/tmp/renhua:/bin/bash
```

生成ldif文件并导入ldap：
```
/usr/share/migrationtools/migrate_passwd.pl /tmp/ldap/passwd.txt /tmp/ldap/passwd.ldif
ldapadd -x -D "uid=ldapadmin,ou=people,dc=javachen,dc=com" -w secret -f /tmp/ldap/passwd.ldif
```

创建group.txt文件，并写入下面内容：
```
renhua:x:503:hadoop
```

生成ldif文件并导入ldap：
```
/usr/share/migrationtools/migrate_group.pl /tmp/ldap/group.txt /tmp/ldap/group.ldif
ldapadd -x -D "uid=ldapadmin,ou=people,dc=javachen,dc=com" -w secret -f /tmp/ldap/group.ldif
```

查询：
```
$ ldapsearch -LLL -x -D 'uid=ldapadmin,ou=people,dc=javachen,dc=com' -w secret -b 'dc=javachen,dc=com' 'uid=renhua'
dn: uid=renhua,ou=people,dc=javachen,dc=com
uid: renhua
cn: renhua
objectClass: account
objectClass: posixAccount
objectClass: top
userPassword:: e2NyeXB0fXg=
loginShell: /bin/bash
uidNumber: 503
gidNumber: 504
homeDirectory: /tmp/renhu
```

修改密码：
```
$ ldappasswd -x -D 'uid=ldapadmin,ou=people,dc=javachen,dc=com' -w secret "uid=renhua,ou=people,dc=javachen,dc=com" -S
```

输入密码,如'renhua'。
也可通过ldap提供的java驱动包来操作。


## 配置 Hive 集成 LDAP

修改配置文件hive-size.xml：
```
<property>
  <name>hive.server2.authentication</name>
  <value>LDAP</value>
</property>
<property>
  <name>hive.server2.authentication.ldap.url</name>
  <value>ldap://cdh1</value>
</property>
<property>
  <name>hive.server2.authentication.ldap.baseDN</name>
  <value>ou=people,dc=javachen,dc=com</value>
</property>
```

为什么这样配置，可以参考 [LdapAuthenticationProviderImpl.java](https://svn.apache.org/repos/asf/hive/trunk/service/src/java/org/apache/hive/service/auth/LdapAuthenticationProviderImpl.java) 源码。

修改配置后，记得重启hive。

### 测试

然后使用 beeline 测试：
```
$ beeline -u "jdbc:hive2://cdh1:21050/default;" -n test -p test
  scan complete in 2ms
  Connecting to jdbc:hive2://cdh1:21050/default;
  Connected to: Impala (version 2.0.0-cdh5)
  Driver: Hive JDBC (version 0.13.1-cdh5.2.0)
  Transaction isolation: TRANSACTION_REPEATABLE_READ
  Beeline version 0.13.1-cdh5.2.0 by Apache Hive

  0: jdbc:hive2://cdh1:21050/default>show tables;
  +-----------------------------+--+
  |            name             |
  +-----------------------------+--+
  | t1                          |
  | tab1                        |
  | tab2                        |
  | tab3                        |
  +-----------------------------+--+
  4 rows selected (0.325 seconds)
```

## 配置 Impala 集成 LDAP

修改配置文件：
```
-enable_ldap_auth=true \
-ldap_uri=ldaps://cdh1 \
-ldap_baseDN=ou=people,dc=javachen,dc=com
```

注意：

- 如果没有开启 ssl，则添加-ldap_passwords_in_clear_ok=true，同样如果开启了 ssl，则 ldap_uri 值为 ldaps://XXXX
- ldap_baseDN 的值是 ou=people,dc=javachen,dc=com，因为 impala 会将其追加到 uid={用户名}, 后面

记得重启服务。

### 测试

使用 impala-shell 测试：
```
$ impala-shell -l -u test --auth_creds_ok_in_clear
Starting Impala Shell using LDAP-based authentication
 LDAP password for test:
 Connected to cdh1:21000
 Server version: impalad version 2.0.0-cdh5 RELEASE (build ecf30af0b4d6e56ea80297df2189367ada6b7da7)
 Welcome to the Impala shell. Press TAB twice to see a list of available commands.

 Copyright (c) 2012 Cloudera, Inc. All rights reserved.

 (Shell build version: Impala Shell v2.0.0-cdh5 (ecf30af) built on Sat Oct 11 13:56:06 PDT 2014)
 [cdh1:21000] >
```


## 参考文章

- https://segmentfault.com/a/1190000002532332