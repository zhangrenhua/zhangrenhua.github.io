---
title: Hive进阶-自定义用户权限控制
description: Hive从0.10版本(包含0.10版本)以后可以通过元数据来控制权限，Hive-0.10之前的版本对权限的控制主要是通过Linux的用户和用户组来控制，不能对Hive表的CREATE、SELECT、DROP等操作进行控制，当然Hive基于元数据来控制权限也不是完全安全的，目的就是为了防止用户不小心做了不该做的操作。本文通过介绍如何配置用户权限，并结合《Hive进阶-自定义安全策略》就可以实现基于用户密码的表/数据库权限认证。
tags: [hadoop生态圈, hive]
categories: hive
date: 2016-1-31 18:14:01
---

## 背景

Hive从0.10版本(包含0.10版本)以后可以通过元数据来控制权限，Hive-0.10之前的版本对权限的控制主要是通过Linux的用户和用户组来控制，不能对Hive表的CREATE、SELECT、DROP等操作进行控制，当然Hive基于元数据来控制权限也不是完全安全的，目的就是为了防止用户不小心做了不该做的操作。本文通过介绍如何配置用户权限，并结合《Hive进阶-自定义安全策略》就可以实现基于用户密码的表/数据库权限认证。

笔者使用hive1.1进行测试。

## 开启权限认证

修改hive-site.xml文件，添加如下参数：
```
<property>
   <name>hive.metastore.authorization.storage.checks</name>
    <value>true</value>
</property>
<property>
    <name>hive.metastore.execute.setugi</name>
    <value>false</value>
</property>
<property>
    <name>hive.security.authorization.enabled</name>
    <value>true</value>
</property>
<property>
   <name>hive.security.authorization.createtable.owner.grants</name>
    <value>ALL</value>
</property>
```

`hive.security.authorization.enabled`：是开启权限验证，默认为false。
`hive.security.authorization.createtable.owner.grants`：是指表的创建者对表拥有所有权限，例如创建一个表table1，这个用户对表table1拥有SELECT、DROP等操作。还有个值是NULL，表示表的创建者无法访问该表，这个肯定是不合理的。

## 自定义超级用户

修改hive-site.xml文件，添加如下参数：
```
<property>
    <name>hive.security.authorization.task.factory</name>
    <value>org.apache.hadoop.hive.ql.parse.authorization.HiveAuthorizationTaskFactoryImpl</value>
</property>
<property> 
    <name>hive.semantic.analyzer.hook</name> 
    <value>com.pactera.hive.AuthHook</value>  
</property>
```

`hive.semantic.analyzer.hook`：超级用户实现类

com.pactera.hive.AuthHook.java示例：
```
import org.apache.hadoop.conf.Configurable;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hive.ql.parse.ASTNode;
import org.apache.hadoop.hive.ql.parse.AbstractSemanticAnalyzerHook;
import org.apache.hadoop.hive.ql.parse.HiveParser;
import org.apache.hadoop.hive.ql.parse.HiveSemanticAnalyzerHookContext;
import org.apache.hadoop.hive.ql.parse.SemanticException;
import org.apache.hadoop.hive.ql.session.SessionState;

/**
 * 
 * @@Description: 设置Hive超级管理员
 */
public class AuthHook extends AbstractSemanticAnalyzerHook implements Configurable {

    private Configuration conf;
    // private Log LOG = LogFactory.getLog(AuthHook.class);

    // 可通过hive-site.xml配置
    private static String admin = "hive";

    @Override
    public ASTNode preAnalyze(HiveSemanticAnalyzerHookContext context, ASTNode ast) throws SemanticException {

        // LOG.info("type:" + ast.getToken().getType());

        switch (ast.getToken().getType()) {
        case HiveParser.TOK_CREATEDATABASE:
        case HiveParser.TOK_DROPDATABASE:
        case HiveParser.TOK_CREATEROLE:
        case HiveParser.TOK_DROPROLE:
        case HiveParser.TOK_GRANT:
        case HiveParser.TOK_REVOKE:
        case HiveParser.TOK_REVOKE_ROLE:
        case HiveParser.TOK_GRANT_ROLE:
        case HiveParser.TOK_IMPORT:
        case HiveParser.TOK_SHOW_ROLES:
            String userName = null;
            if (SessionState.get() != null && SessionState.get().getAuthenticator() != null) {
                userName = SessionState.get().getAuthenticator().getUserName();
            }
            if (!admin.equalsIgnoreCase(userName)) {
                throw new SemanticException(userName + " can't use ADMIN options, except " + admin + ".");
            }
            break;
        default:
            break;
        }
        return ast;
    }

    @Override
    public Configuration getConf() {
        return conf;
    }

    @Override
    public void setConf(Configuration arg0) {
        conf = arg0;
    }
}
```

<font color="red">由于是测试所以代码中写死超级用户为hive，可自定义通过配置文件传入，或者存入数据库中都行。</font>

<font color="red">将自定义AuthHook代码打成jar包，上传到hive/lib下即可实现超级用户功能，如果没有超级用户，则hive集群啥都不没权限使用了。（记得重启hive服务）</font>

## 权限验证

使用超级管理员登录hive：
```
[hadoop@cdh-s1 hive]$ beeline 
Beeline version 1.1.0-cdh5.5.1 by Apache Hive
beeline> !connect jdbc:hive2://cdh-m1:2181/default;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hive_zookeeper_namespace_hive
scan complete in 3ms
Connecting to jdbc:hive2://cdh-m1:2181/default;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hive_zookeeper_namespace_hive
Enter username for jdbc:hive2://cdh-m1:2181/default;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hive_zookeeper_namespace_hive: hive
Enter password for jdbc:hive2://cdh-m1:2181/default;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hive_zookeeper_namespace_hive: ********
```

创建角色，并赋权：
```
create role role_test1;
GRANT CREATE ON DATABASE default TO USER hive_demo; 
grant role_test1 to user hive_demo;
```

本次赋予用户`hive_demo`可在default数据库下创建table，详细权限设置可参考：http://outofmemory.cn/code-snippet/6626/hive-security-authentication

注意hive_demo用户是虚拟的。

使用hive_demo登录命令行并创建hive_demo表：
```
[hadoop@cdh-m1 ~]$ beeline 
Beeline version 1.1.0-cdh5.5.1 by Apache Hive
beeline> !connect jdbc:hive2://cdh-m1:2181/default;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hive_zookeeper_namespace_hive
scan complete in 2ms
Connecting to jdbc:hive2://cdh-m1:2181/default;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hive_zookeeper_namespace_hive
Enter username for jdbc:hive2://cdh-m1:2181/default;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hive_zookeeper_namespace_hive: hive_demo 
Enter password for jdbc:hive2://cdh-m1:2181/default;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hive_zookeeper_namespace_hive: ********
Connected to: Apache Hive (version 1.1.0-cdh5.5.1)
Driver: Hive JDBC (version 1.1.0-cdh5.5.1)
Transaction isolation: TRANSACTION_REPEATABLE_READ
0: jdbc:hive2://cdh-m1:2181/default> create table hive_demo(name string);
No rows affected (0.518 seconds)
0: jdbc:hive2://cdh-m1:2181/default> show tables;
+------------+--+
|  tab_name  |
+------------+--+
| hive_demo  |
+------------+--+
1 row selected (0.108 seconds)
```


使用test用户删除hive_demo表，验证权限是否配置成功：
```
0: jdbc:hive2://cdh-m1:2181/default> !close
Closing: 0: jdbc:hive2://cdh-m1:2181/default;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hive_zookeeper_namespace_hive
beeline> 
beeline> !connect jdbc:hive2://cdh-m1:2181/default;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hive_zookeeper_namespace_hive
Connecting to jdbc:hive2://cdh-m1:2181/default;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hive_zookeeper_namespace_hive
Enter username for jdbc:hive2://cdh-m1:2181/default;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hive_zookeeper_namespace_hive: test
Enter password for jdbc:hive2://cdh-m1:2181/default;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hive_zookeeper_namespace_hive: ********
Connected to: Apache Hive (version 1.1.0-cdh5.5.1)
Driver: Hive JDBC (version 1.1.0-cdh5.5.1)
Transaction isolation: TRANSACTION_REPEATABLE_READ
0: jdbc:hive2://cdh-m1:2181/default> show tables;
+------------+--+
|  tab_name  |
+------------+--+
| hive_demo  |
+------------+--+
1 row selected (0.057 seconds)
0: jdbc:hive2://cdh-m1:2181/default> drop table hive_demo;
Error: Error while compiling statement: No privilege 'Drop' found for outputs { database:default, table:hive_demo} (state=42000,code=403)
```

test没有删除权限，错误信息如下：
```
Error: Error while compiling statement: No privilege 'Drop' found for outputs { database:default, table:hive_demo} (state=42000,code=403)
```

## 总结

通过上面步骤，可以控制虚拟用户的访问权限，再参考[Hive进阶-自定义安全策略](/2016/01/31/hadoop-Hive进阶-自定义安全策略/)设置用户的登录密码，即可实现用户权限控制，提高hive的安全性。


## 参考资料

根据下面两篇blog配置是不会成功的，少了一些关键配置。不过感谢作者的一些提示。

- http://blog.csdn.net/xiaolang85/article/details/14166947
- http://outofmemory.cn/code-snippet/6626/hive-security-authentication


