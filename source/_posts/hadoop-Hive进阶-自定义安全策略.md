---
title: Hive进阶-自定义安全策略
description: 一共提供了三种安全认证方式：</br>LDAP Authentication using OpenLDAP</br>Setting up Authentication with Pluggable Access Modules</br>Configuring Custom Authentication</br>我们通常采用第三种自定义的方式，下面介绍如何自定义用户名密码验证。
tags: [hadoop生态圈, hive]
categories: hive
date: 2016-1-31 13:18:32
---

## 背景

一共提供了三种安全认证方式：
[LDAP Authentication using OpenLDAP](http://doc.mapr.com/display/MapR/Using+HiveServer2#UsingHiveServer2-LDAPAuthenticationusingOpenLDAP)
[Setting up Authentication with Pluggable Access Modules](http://doc.mapr.com/display/MapR/Using+HiveServer2#UsingHiveServer2-SettingupAuthenticationwithPluggableAccessModules)
[Configuring Custom Authentication](http://doc.mapr.com/display/MapR/Using+HiveServer2#UsingHiveServer2-ConfiguringCustomAuthentication)

我们通常采用第三种自定义的方式，下面介绍如何自定义用户名密码验证。

笔者使用hive1.1版本作为测试。

## 示例代码

自定义Authticator实现PasswdAuthenticationProvider, Configurable，重写Authenticate方法进行用户名密码校验：
```
import javax.security.sasl.AuthenticationException;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.apache.hadoop.conf.Configurable;
import org.apache.hadoop.conf.Configuration;
import org.apache.hive.service.auth.PasswdAuthenticationProvider;

public class Authticator implements PasswdAuthenticationProvider, Configurable {
    private Configuration conf = null;
    private static final Log LOG = LogFactory.getLog(Authticator.class);

    @Override
    public Configuration getConf() {
        if (this.conf == null) {
            this.conf = new Configuration();
        }
        return this.conf;
    }

    @Override
    public void setConf(Configuration arg0) {
        this.conf = arg0;
    }

    @Override
    public void Authenticate(String username, String password) throws AuthenticationException {

        if (username == null || password == null) {
            throw new AuthenticationException("error.");
        }

        LOG.info("user: " + username + " try login.");

        if (!password.equals("abcd1234")) {
            String message = "user name and password is mismatch. user:" + username;
            throw new AuthenticationException(message);
        }

        LOG.info("user " + username + " login system successfully.");
    }
}
```

<font color="red">在Authenticate方法中可以连接数据库动态进行权限认证也可以使用利用conf对象在配置文件中固定用户名密码（密码建议MD5）。</font>

## 开启自定义验证

修改hive-site.xml：
```
<property>
    <name>hive.server2.authentication</name>
    <value>CUSTOM</value>
</property>
<property>
    <name>hive.server2.custom.authentication.class</name>
    <value>com.pactera.hive.Authticator</value>
</property>
```

`hive.server2.custom.authentication.class`：为自定义Authticator类。

将自定义Authticator代码打包jar包，上传到hive/lib下即可实现HiveServer2的安全策略之自定义用户名密码验证了。

完成上面操作记得重启hive服务哦，此时JDBC/Thrift连接都得出传入密码：
```
Connection con = DriverManager.getConnection(
                "jdbc:hive2://cdh-m1:2181/default;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hive_zookeeper_namespace_hive",
                "hadoop", "abcd1234");
```