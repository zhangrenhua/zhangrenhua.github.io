---
title: Openssl、Tomcat9(Http2 APR)、Https单/双向认证
description: 去年开始就有很多公司开始尝试HTTP2.0协议，目前为止HTTP协议只有三个版本，HTTP1.0、HTTP1.1、HTTP2.0。现在普遍还是HTTP1.1使用的比较多，具体为什么HTTP2.0现在很火大家可以网上找找答案，去深入了解不同协议之间的具体实现，以及性能差距。具体可以参考：【HTTP2优势分析】http://blog.csdn.net/jiayanhui2877/article/details/44957105</br></br>为什么使用这些版本：</br>1、tomcat9开始才支持HTTP2.0协议</br>2、jdk1.8才有HTTP2.0的具体实现</br>3、由于使用了Tomcat Apr底层实现，需下载native包。（bio、nio、apr性能比较）</br>4、Apr只支持OpenSSL生成的CA证书</br></br>本次尝试Windows下部署openssl、tomcat、tomcat apr、https、HTTP2.0。
tags: [j2ee, tomcat]
categories: j2ee
date: 2016-5-4 16:37:26
---

## 背景

去年开始就有很多公司开始尝试HTTP2.0协议，目前为止HTTP协议只有三个版本，HTTP1.0、HTTP1.1、HTTP2.0。现在普遍还是HTTP1.1使用的比较多，具体为什么HTTP2.0现在很火大家可以网上找找答案，去深入了解不同协议之间的具体实现，以及性能差距。具体可以参考：[HTTP2优势分析](http://blog.csdn.net/jiayanhui2877/article/details/44957105)

本次尝试Windows下部署openssl、tomcat、tomcat apr、https、HTTP2.0。


## 软件版本

软件名称 | 版本号 | 下载地址
--- | --- | ---
jdk | 1.8.0_11 | http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html
tomcat | 9.0.0.M4 | http://apache.fayea.com/tomcat/tomcat-9/v9.0.0.M4/bin/apache-tomcat-9.0.0.M4.tar.gz
tomcat-native | 1.2.2 | http://archive.apache.org/dist/tomcat/tomcat-connectors/native
OpenSSL | 1.0.2g | http://www.openssl.org/news/openssl-1.0.2-notes.html


**为什么使用这些版本：**
1、tomcat9开始才支持HTTP2.0协议
2、jdk1.8才有HTTP2.0的具体实现
3、由于使用了Tomcat Apr底层实现，需下载native包。（bio、nio、apr性能比较）
4、Apr只支持OpenSSL生成的CA证书



## 软件安装


#### OpenSSL

下载软件直接安装，配置Path环境变量即可：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/j2ee-openssl-install.png)


### jdk

下载软件直接安装，配置Path和JAVA_HOME环境变量即可：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/j2ee-jdk-install.png)

别忘了配置Path环境变量。


### tomcat

1、下载apache-tomcat-9.0.0.M4.tar.gz，解压到某个目录，如D:/盘：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/j2ee-tomcat-install.png)

下载tomcat-native-1.2.2-win32-bin.zip并解压，将bin/x64/tcnative-1.dll复制到tomcat目录下的APR/lib（没有则先建立）：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/j2ee-tomcat-apr-install.png)


2、修改bin目录下catalina.bat
```
set CATALINA_OPTS="-Djava.library.path=../APR/lib" 
set JAVA_HOME=C:\\Program Files\\Java\\jdk1.8.0_11
```

在setlocal后面加上如上两行，注意：
`CATALINA_OPTS`是配置APR插件的位置
`JAVA_HOME`是jdk安装目录，<font color='red'>请填写自己的</font>。



## OpenSSL生成CA证书


以前用tomcat配置过HTTPS双向认证，那时候主要用的是JDK自带的keytool工具。这次是用tomcat + openssl，区别比较大

在网上搜索了很多文章，发现全面介绍的不多，或者就是版本比较旧了。所以把我配置的过程完整地记录下来，以供参考

首先要说明一下，HTTPS涉及到的内容非常繁杂，包括各种术语、命令、算法。本文会尽量把我知道的解释一下



### 环境准备

根据上面步骤将jdk/openssl安装好即可，本次配置是运行在Windows上。


### 使用场景

在tomcat配置一个HTTPS双向认证，既向客户端表明自己的身份，也只允许特定的客户端访问。本文说的主要是作为server端角色的配置，至于作为client的配置，最后也会稍微介绍一下，但是不会详细说明

一般来说，互联网站不会去配置双向认证，因为客户端证书的分发和管理会比较麻烦，会把用户挡在门外，所以一般是不能这么做的。当然，像银行等对安全要求很高的网站，也会采用双向认证，比如U盾、安全控件什么的，其实就是固化的客户端证书 

但是对于企业应用来说，客户一般是固定的，比如两个已知的系统对接、内部系统集成等。所以在企业应用领域，双向认证还是比较常见的 


<font color='red'>其实对于某些安全要求很高的网站，会在配置了HTTPS的基础下对内容进行组合加密。或者根本不使用HTTPS，对数据进行多种加密。</font>



### 背景知识 

证书（Certificate）是HTTPS的核心，但是其实证书并不是一个单一的东西，而是几种技术的综合 

为了网络传输的安全，有很多种技术，最主要的是以下3种： 

1、加密/解密 

避免消息明文传输，对消息进行加密。早期一般是用对称加密算法，现在一般都是不对称加密，最常见的算法就是RSA。在后面的介绍中，也会多次看到RSA这个词 

2、消息摘要 

这个技术主要是为了避免消息被篡改。消息摘要是把一段信息，通过某种算法，得出一串字符串。这个字符串就是消息的摘要。如果消息被篡改（发生了变化），那么摘要也一定会发生变化（一般是这样的。如果2个不同的消息生成的摘要是一样的，那么这就叫发生了碰撞） 

消息摘要的算法主要有MD5和SHA，在证书领域，一般都是用SHA（安全哈希算法） 

3、数字签名 

数字签名是为了验证双方的身份，避免身份伪造 

以上三个技术结合起来，就是在HTTPS中广泛应用的证书（certificate），证书本身携带了加密/解密的信息，并且可以标识自己的身份，也自带消息摘要 



### 创建CA（Certificate Authority） 


这个CA，也叫“根证书” 

服务端做了一个证书，但是这是没有法律效力的，谁都可以自己做证书，就根本达不到安全的目的。所以就要有一个机构，负责来确认服务端的身份，然后统一的签发证书。这样才能有权威性 

当浏览器通过HTTPS协议访问一个网站，网站首先会发过来一个自己的证书（certificate）。接下来浏览器就会到权威机构（CA），去验证一下这个证书是不是它签发的。如果是的话，就信任这个网站的证书，继续访问;如果不是的话，要怎么处理就依赖于实现了。一般的浏览器会弹出一个警告，让用户自己决定要不要继续访问。当然直接拒绝也是可以的 

现在国际上有3大CA机构，如果是要自己做一个网站的话，如上所述，一般是需要请这些权威机构帮忙签发证书的。现在所有的主流浏览器，默认都安装了这些CA的根证书，所以如果网站的证书是这些权威机构签发的，浏览器就不会发出警告了。比如支付宝，它的证书是由VeriSign签发的，所以访问支付宝，浏览器不会发出警告 

![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/j2ee-openssl-ca-demo.png)


这里还有一个链条的关系，比如我有10个子网站，如果每个都要去找CA签发证书，就很麻烦。我可以找CA给我签发一个次级根证书，然后再用这个次级根证书给自己签发10个证书。那么只要客户的浏览器里有CA根证书就可以了，这10个证书都可以通过认证，不要求客户安装次级根证书，原文见下： 


如果是企业应用，那完全可以自己给自己当CA，因为可以要求目标用户（系统）安装自己的CA根证书，效果是一样的，还可以省下请权威CA签发证书的费用（互联网应用分发自己的CA到无数的互联网用户上，难度很大） 

下面就介绍如何创建自己的CA 


### 准备工作

在D盘创建几个子目录：
```
d:/ca/private
d:/ca/certificates 
```

其中private放的是私钥和CSR（后面会介绍），certificates里放的就是证书了 


### 创建CA私钥

使用命令行进入d:/ca目录执行下面命令：
```
openssl genrsa -aes256 -out private/ca.key.pem 2048
```

这是会提示输入密码（如：123456），后面的命令也有会提示输入密码的，都设置为一样就好了。

最后的参数是RSA密钥的长度，默认是512。2048其实长了一点，老的浏览器稍后会不支持，不过现在的主流浏览器都是支持的，所以问题不大 

注意：<font color='red'>必须使用管理员进入命令行，否则后面执行会报错。笔者就在这上面吃过亏</font>


用这个命令，可以看一下刚才创建的这个私钥的信息 
```
openssl rsa -noout -text -in private/ca.key.pem
```

另外，最后的.pem扩展名，是表示该私钥用PEM编码。实际上私钥和证书都是用PEM编码的，PEM只是一种编码格式，不需要太在意。tomcat可以直接处理这种编码格式，但是浏览器和JAVA都不行，所以在需要的时候，会把编码从PEM改成PKCS，后面会介绍。只要知道证书和私钥都有编码，只是编码是PEM还是PKCS的区别而已。就像"华哥哥"可以用UTF-8编码，也可以用GBK编码一样，内容是不变的 


### 创建CA签名请求


```
openssl req -new -key private/ca.key.pem -out private/ca.csr -subj "/C=CN/ST=SZ/L=SZ/O=hua/OU=hua/CN=*.hua.com"
```

这里要注意的是，如果不用-subj参数，那么就会在命令行交互输入签发目标的身份识别信息，这叫DN（Distinguished Name）。其中别的都不要紧，最重要的是CN那一行，因为我这里是根证书，所以我设置为*.hua.com，这样我后面用这个CA签发的www.hua.com、game.hua.com、news.hua.com……，全都是有效的 

生成的签名请求文件，是ca.csr


### 自己签发CA根证书 
```
openssl x509 -req -days 3650 -sha1 -extensions v3_ca -signkey private/ca.key.pem -in private/ca.csr -out certificates/ca.cer
```

这里参数很复杂，可以用openssl x509 -help自己研究一下 

<font color='red'>生成的ca.cer，就是最终的根证书了！这个文件非常重要，因为后续的服务端证书、客户端证书，都是用这个CA签发的，也要把它分发给客户，让他们导入到自己的浏览器或者系统中 </font>

注意：<font color='red'>这里使用的是sha1加密算法，在chrome高版本中已经会提示加密算法不安全，建议使用sha2（SHA-256/SHA-512）。但是xp系统上不支持sha2，所以笔者这次还是使用sha1</font>
参考：[Chrome为SHA-1设置倒计时](http://www.infoq.com/cn/news/2014/09/chrome_SHA1)


### 把根证书从PEM编码转为PKCS编码 

这步其实不是必选的，但是前面说过，JAVA环境是不能直接用PEM编码的证书的，很多浏览器也不行，所以有时候也需要转一下编码 

```
openssl pkcs12 -export -cacerts -inkey private/ca.key.pem -in certificates/ca.cer -out certificates/ca.p12
```

得到的ca.p12就是转码后的CA根证书，在不能直接用ca.cer的时候，就用ca.p12代替 

### 签发服务端证书

现在CA根证书和私钥都有了，就可以开始签发服务端证书了（签发请求ca.csr是过程文件，有了cer就不再需要它了，要删掉也可以）。下面的命令和签发CA证书时都差不多，但是参数上有区别 

1、创建服务端私钥 
```
openssl genrsa -aes256 -out private/server.key.pem 2048
```

2、创建服务端证书签发请求
```
openssl req -new -key private/server.key.pem -out private/server.csr -subj "/C=CN/ST=SZ/L=SZ/O=hua/OU=hua/CN=www.hua.com"
```

和ca.csr的区别在于，这里的CN不是*.hua.com，而是www.hua.com，因为我现在是在为www.hua.com申请证书 

由于在本地测试，所以需要修改hosts文件，让www.hua.com指向本机（如忘记配置，可能导致浏览器警告）：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/j2ee-tomcat-hosts.png)

3、利用CA根证书，签发服务端证书 
```
openssl x509 -req -days 3650 -sha1 -extensions v3_req -CA certificates/ca.cer -CAkey private/ca.key.pem -CAserial ca.srl -CAcreateserial -in private/server.csr -out certificates/server.cer
```

这里和前面自己签发CA证书时，参数区别就比较大了，最后得到的server.cer，就是服务端证书 


### 测试单向认证

有了服务端证书就可以配置单向认证了

将刚才生成的证书文件复制到tomcat的conf目录下（server.key.pem、server.cer）

修改tomcat/conf目录下的server.xml文件，笔者tomcat安装目录为D:\apache-tomcat-9.0.0.M4：
```
<!-- 单向认证 -->
<Connector port="443" protocol="org.apache.coyote.http11.Http11AprProtocol"
           maxThreads="150" SSLEnabled="true" >
    <UpgradeProtocol className="org.apache.coyote.http2.Http2Protocol" />
    <SSLHostConfig honorCipherOrder="false" >
        <Certificate certificateKeyFile="conf/server.key.pem"
                     certificateFile="conf/server.cer"
                     type="RSA" />
    </SSLHostConfig>
</Connector>
```

在Service标签下添加入上配置：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/j2ee-tomcat-https-single.png)

`certificateKeyFile`：证书key路径
`certificateFile`：证书路径
`<UpgradeProtocol className="org.apache.coyote.http2.Http2Protocol" />`:添加这行即代表使用HTTP2.0协议，chrome浏览器可以安装[HTTP/2 and SPDY indicator](https://chrome.google.com/webstore/detail/http2-and-spdy-indicator/mpbpobfflnpcgagjijhmgnchggcjblin)插件查看协议版本


#### 启动tomcat

进入tomcat/bin目录，双击startup.bat，，会要求输入一个密码然后回车（笔者密码为：123456）：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/j2ee-tomcat-https-single-start.png)

接下来用https://www.hua.com来访问，浏览器报警： 
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/j2ee-tomcat-https-single.action.png)

可以看到这个证书是由*.hua.com这个CA颁发的，浏览器不认识，所以不信任由这个CA签发的所有证书。接下来就需要把ca.cer导入浏览器。这里直接导入server.cer也是可以的，但是后面如果又创建一个网站比如说www2.hua.com，那么又不行了。所以最好的办法是直接导入CA根证书，那么后续只要是用这个根证书签发的证书，浏览器都会信任 

#### 导入证书

进入到D:\ca\certificates目录，双击ca.p12，点击下一步，输入密码，：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/j2ee-tomcat-https-ca-install.png)

在此双击ca.p12，将证书安装到“受信任的根证书颁发机构”：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/j2ee-tomcat-https-ca-install1.png)

接下来用https://www.hua.com来访问，浏览器还会报警： 
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/j2ee-tomcat-https-single.action1.png)

这是因为我使用的是SHA-1证书，前面提到过，建议升级。到此单向认证+http2.0+apr测试成功。


### 测试双向认证

如果要配置服务器只允许合法的用户访问，就需要配置双向认证 

配置为双向认证之后，除了服务端要发证书给客户端之外，客户端也要发客户端证书到服务端，服务端认证通过，才允许访问

这里要说明一下，客户端证书是怎么来的 

有2种方式：

第一种，客户端也自己CA，然后签发证书给自己。把客户端的CA根证书发过来，配置成SSLCACertificateFile。这在互联网应用里基本是不可能的，安全和管理都是问题。但是在企业应用里，还是比较常见的，双方互相交换CA根证书 

第二种，就用刚才服务端的CA根证书，签发一个客户端证书，发给用户，用户每次用这个证书来发请求，像银行，支付宝等等，用的是这种方式 

当然理论上其实还有一种办法，就是客户自己去找权威CA签证书，但是这个是不可能的，因为很麻烦，找CA签也非常贵 

本文用的是第2种方法。其实都是一样的。关键还是在CA根证书上，所以前面也说过了，CA根证书非常重要 


#### 签发客户端证书 

1、创建客户端私钥 
```
openssl genrsa -aes256 -out private/client.key.pem 2048
```

2、创建客户端证书签发请求 
```
openssl req -new -key private/client.key.pem -out private/client.csr -subj "/C=CN/ST=SZ/L=SZ/O=hua/OU=hua/CN=hua"
```

这里的不同在于，这里的CN不是*.hua.com，也不是www.hua.com，随便填一个就好了，或者干脆叫user都没问题，反正是一个客户端证书 

3、利用CA根证书，签发客户端证书
```
openssl x509 -req -days 3650 -sha1 -extensions v3_req -CA certificates/ca.cer -CAkey private/ca.key.pem -CAserial ca.srl -CAcreateserial -in private/client.csr -out certificates/client.cer
```

这里和签发server.cer基本是一样的 

4、把客户端证书转换成p12格式
```
openssl pkcs12 -export -clcerts -inkey private/client.key.pem -in certificates/client.cer -out certificates/client.p12
```
这步是必须的，因为稍后就需要把客户端证书导入到浏览器里，但是一般浏览器都不能直接使用PEM编码的证书 

#### 测试双向认证

将刚才生成的证书文件复制到tomcat的conf目录下（ca.cer）

修改tomcat/conf目录下的server.xml文件，笔者tomcat安装目录为D:\apache-tomcat-9.0.0.M4：
```
<!-- 双向认证 -->
<Connector port="443" protocol="org.apache.coyote.http11.Http11AprProtocol"
           maxThreads="150" SSLEnabled="true" >
    <UpgradeProtocol className="org.apache.coyote.http2.Http2Protocol" />
    <SSLHostConfig honorCipherOrder="false" 
        certificateVerification="require"
        certificateVerificationDepth="10"
        caCertificateFile="conf/ca.cer" >
        <Certificate certificateKeyFile="conf/server.key.pem"
                     certificateFile="conf/server.cer"
                     type="RSA" />
    </SSLHostConfig>
</Connector>
```

添加下面三个参数，其他不变。
`certificateVerification`：设置客户端必须发送证书
`certificateVerificationDepth`：证书链深度
`caCertificateFile`：验证客户端证书文件

#### 导入证书

进入到D:\ca\certificates目录，双击client.p12，点击下一步，输入密码，：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/j2ee-tomcat-https-ca-install.png)

在此双击client.p12，将证书安装到“受信任的根证书颁发机构”：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/j2ee-tomcat-https-ca-install1.png)

如果根据上面步骤验证了单向认证，可以先在个人证书里把“颁发给”等于“*.hua”的证书删除掉，保留下面的“hua”证书就行。（不删掉也没关系，只不过会提示有两个证书）：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/j2ee-tomcat-https-ca-list.png)

启动tomcat，还是得输入密码。

#### 浏览器验证

Chrome浏览器输入，`https://www.hua.com/`，提示选择证书，点击确定即可：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/j2ee-tomcat-https-ca-chrome-test.png)

进入之后Chrome还是会有警告，还是加密算法是SHA-1的问题，建议后学者更新算法SHA-2。

怎么验证HTTP2.0？根据前面所说安装[HTTP/2 and SPDY indicator](https://chrome.google.com/webstore/detail/http2-and-spdy-indicator/mpbpobfflnpcgagjijhmgnchggcjblin)插件即可查看，如果是HTTP2.0插件就会显示蓝色闪电标识：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/j2ee-tomcat-http2-test.png)


IE浏览器输入，`https://www.hua.com/`，提示选择证书，点击确定即可：
![Alt text](http://7xoqbc.com1.z0.glb.clouddn.com/j2ee-tomcat-https-ca-ie-test.png)

注意：IE是没有提示加密算法不安全的哦。

## 参考资料

网上找了很多资料，发现都不是太详细，尤其在参数配置这块，基本找不到APR配置的。最终笔者在apache官网找到上面配置。

- https://tomcat.apache.org/tomcat-9.0-doc/config/http.html#SSL_Support_-_SSLHostConfig






