---
title: IBM JDK进行RSA加解密时遇到的问题
description: 最近几天有位同事在接口开发中遇到一个难点。本来在tomcat上运行的好好的程序，部署到IBM WebSphere上就出现数据加解密报错。经过一番排除终于找到问题的根源，问题出在了IBM WebSphere用的jdk版本是IBM自带的，所以才会出现加解密错误的问题。下面我对使用`RSA`加解密时需要注意的细节进行详细说明，并附上本次问题的完美解决方案。
tags: "j2ee-疑难解决"
#tags：[hexo, next]
categories: java
date: 2015-11-01 20:20:00
---
最近几天有位同事在接口开发中遇到一个难点。本来在tomcat上运行的好好的程序，部署到IBM WebSphere上就出现数据加解密报错。经过一番排除终于找到问题的根源，问题出在了IBM WebSphere用的jdk版本是IBM自带的，所以才会出现加解密错误的问题。下面我对`RSA`加解密进行详细说明，并附上本次问题的`完美解决`方案。
## 问题解决
### 错误代码
```
import java.io.FileInputStream;
import java.io.ObjectInputStream;
import java.security.Key;

import javax.crypto.Cipher;

import sun.misc.BASE64Decoder;

/**
 * RSA加密算法
 */
public class RSAEncryption {

    /**
     * RSA加密算法
     */
    private static RSAEncryption rsaEncrypt = null;

    /**
     * 指定加密算法为RSA
     */
    private static final String ALGORITHM = "RSA";

    /**
     * 公钥
     */
    private static Key PublicKey;
    /**
     * 私钥
     */
    private static Key PrivateKey;
    /**
     * 指定公钥存放文件
     */
    private static String PUBLIC_KEY_FILE = "D:\\PublicKey.crt";

    /**
     * RSA加密算法
     */
    public RSAEncryption() {
        // 将文件中的公钥对象读出
        try {
            ObjectInputStream ois = new ObjectInputStream(new FileInputStream(PUBLIC_KEY_FILE));
            PublicKey = (Key) ois.readObject();
            ois.close();
        } catch (Exception localException) {
            localException.printStackTrace();
        }

    }

    /**
     * 初始化RSA加密算法
     */
    public static RSAEncryption initialize() {
        synchronized (RSAEncryption.class) {
            if (rsaEncrypt == null) {
                rsaEncrypt = new RSAEncryption();
            }
        }
        return rsaEncrypt;
    }

    /**
     * 解密
     * 
     * @param cryptograph
     *            密文
     * @param key
     *            解密KEY
     * @return 解密后数据
     */
    private String doDecrypt(String cryptograph, Key key) {
        try {
            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.DECRYPT_MODE, key);
            BASE64Decoder decoder = new BASE64Decoder();
            byte[] b1 = decoder.decodeBuffer(cryptograph);
            // 执行解密操作
            byte[] b = cipher.doFinal(b1);
            return new String(b, "UTF-8");
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 私钥解密
     * 
     * @param cryptograph
     *            密文
     * @return 解密后数据
     */
    public static String PrivateDecrypt(String cryptograph) {
        return initialize().doDecrypt(cryptograph, PrivateKey);
    }

    // 测试代码
    public static void main(String[] args) {
        String key = "MM9YaThlm3vVg+l8/BkTA8+Xa5XdkEzB/1cbxLOodAjuB0xWNJu9NiR/58LCE2xoqKH3d78I+FxW9pDOn1n5Ze7aRDPEIa0hyK6jw9YTXBS4ChJfaduUSDvzCOteqmjuJIkZ5Ob+hHYPcOHSuovLc8nfQA3qYSRBJQKNQRcwDFc=";
        initialize().doDecrypt(key, PublicKey);

    }
}
```
执行main方法会报如下错误：
```
java.security.InvalidKeyException: Public Key cannot be used to decrypt.
    at com.ibm.crypto.provider.RSACipher.a(Unknown Source)
    at com.ibm.crypto.provider.RSACipher.engineInit(Unknown Source)
    at javax.crypto.Cipher.a(Unknown Source)
    at javax.crypto.Cipher.a(Unknown Source)
    at javax.crypto.Cipher.init(Unknown Source)
    at javax.crypto.Cipher.init(Unknown Source)
    at com.dongfeng.pactera.RSAEncryption.doDecrypt(RSAEncryption.java:78)
    at com.dongfeng.pactera.RSAEncryption.main(RSAEncryption.java:104)
```
当我看到这个错误的时候，我还以为是不是秘钥文件的问题。当我google查看异常信息时，才发现原来我们这边使用的是IBM JDK。这里不得不吐槽一下，IBM的东西真是烂到家了。下面列出WebSphere几点让人恶心的个人观点供到家参考：安装包大、启动慢、资源占用多、部署的时候点多了进程直接挂掉等...
### 解决方案
为什么会报这样的错呢？原因在与IBM JDK采用的算法提供商与Oracle JDK不一致导致的。那如何解决呢？通过网上找资料发现一个比较完美的解决方案，使用[bcprov](http://www.bouncycastle.org/java.html)一个开源的基于java1.x的加密算法实现。本文使用的是bcprov-jdk16-1.46.jar。核心步骤：
1、项目中添加bcprov-jdk16-1.46.jar
2、注册bc算法提供商：Security.addProvider(new BouncyCastleProvider())
3、选择正确的加密算法：Cipher.getInstance("<font color="red">RSA/None/PKCS1Padding</font>", "BC");
最终代码：
```
import java.io.FileInputStream;
import java.io.ObjectInputStream;
import java.security.Key;
import java.security.Security;

import javax.crypto.Cipher;

import org.bouncycastle.jce.provider.BouncyCastleProvider;

import sun.misc.BASE64Decoder;

/**
 * RSA加密算法
 * 
 * @author liguoxiang
 * 
 */
public class RSAEncryption {

    static {
        /**
         * 注册算法提供商
         */
        Security.addProvider(new BouncyCastleProvider());
    }

    /**
     * RSA加密算法
     */
    private static RSAEncryption rsaEncrypt = null;

    /**
     * 公钥
     */
    private static Key PublicKey;
    /**
     * 指定公钥存放文件
     */
    private static String PUBLIC_KEY_FILE = "D:\\PublicKey.crt";

    /**
     * RSA加密算法
     */
    public RSAEncryption() {
        // 将文件中的公钥对象读出
        try {
            ObjectInputStream ois = new ObjectInputStream(new FileInputStream(PUBLIC_KEY_FILE));
            PublicKey = (Key) ois.readObject();
            ois.close();
        } catch (Exception localException) {
            localException.printStackTrace();
        }

    }

    public static void main(String[] args) {
        String message = "JLlSjMGxZWNgy71lPWHfeczrbpweQ26RQdjXj7un1FqcvVoKtOU/vlQ7SCxCOT5ANwdns1BIE+PeBPhqnyKQqnPxC9OHNW/5ioDJxXKq10+/D68VL/51c6Auzc5lo81FqYsyiS+MEbBLrUmJpAYpuNoD5T25k45DzjDVD/4u4ro=";
        System.out.println(initialize().doDecrypt(message, PublicKey));
    }

    /**
     * 初始化RSA加密算法
     */
    public static RSAEncryption initialize() {
        if (rsaEncrypt == null) {
            synchronized (RSAEncryption.class) {
                rsaEncrypt = new RSAEncryption();
            }
        }
        return rsaEncrypt;
    }

    /**
     * 解密
     * 
     * @param cryptograph
     *            密文
     * @param key
     *            解密KEY
     * @return 解密后数据
     */
    private String doDecrypt(String cryptograph, Key key) {

        try {
            Cipher cipher = Cipher.getInstance("RSA/None/PKCS1Padding", "BC");
            cipher.init(Cipher.DECRYPT_MODE, key);
            BASE64Decoder decoder = new BASE64Decoder();
            byte[] b1 = decoder.decodeBuffer(cryptograph);
            // 执行解密操作
            byte[] b = cipher.doFinal(b1);
            return new String(b);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }

}
```
正确输出：0a3QcKRRCpQ8A2KN574j524trq93kjM1Mt2zR0ntYge
注：<font color="red">Cipher.getInstance("RSA/None/PKCS1Padding", "BC")</font>这行中的<font color="red">"RSA/None/PKCS1Padding"</font>算法必须根据自己实际情况来填写，否则会出现解析错误的情况。
比如：我将上面代码中的Cipher.getInstance(<font color="red">"RSA/None/PKCS1Padding"</font>, "BC")改成Cipher.getInstance(<font color="red">"RSA"</font>, "BC")，执行main函数结果输出乱码（������...）
### 总结
我希望大家遇到问题时能冷静思考，了解程序运行原理，多上网找资料，最后果断解决。其实开发中遇到异常或者bug，是个人能力提升的一个机会。
以上代码采用同事的demo，代码不规范之处还请多多包涵。下面对RSA算法的一些细节进行讲解。

## RSA算法细节
RSA这种算法1978年就出现了，它是第一个既能用于数据加密也能用于数字签名的算法。它易于理解和操作，也很流行。算法的名字以发明者的名字命名：Ron Rivest, AdiShamir 和Leonard Adleman。 
RSA同时有两把钥匙，公钥与私钥。同时支持数字签名。数字签名的意义在于，对传输过来的数据进行校验。确保数据在传输工程中不被修改。 
### 加密的系统不要具备解密的功能，否则RSA可能不太合适
公钥加密，私钥解密。加密的系统和解密的系统分开部署，加密的系统不应该同时具备解密的功能，这样即使黑客攻破了加密系统，他拿到的也只是一堆无法破解的密文数据。否则的话，你就要考虑你的场景是否有必要用 RSA 了。
### 可以通过修改生成密钥的长度来调整密文长度
生成密文的长度等于密钥长度。密钥长度越大，生成密文的长度也就越大，加密的速度也就越慢，而密文也就越难被破解掉。著名的"安全和效率总是一把双刃剑"定律，在这里展现的淋漓尽致。我们必须通过定义密钥的长度在"安全"和"加解密效率"之间做出一个平衡的选择。
### 生成密文的长度和明文长度无关，但明文长度不能超过密钥长度
不管明文长度是多少，RSA 生成的密文长度总是固定的。
但是明文长度不能超过密钥长度。比如 Java 默认的 RSA 加密实现不允许明文长度超过密钥长度减去 11(单位是字节，也就是 byte)。也就是说，如果我们定义的密钥(我们可以通过 java.security.KeyPairGenerator.initialize(int keysize) 来定义密钥长度)长度为 1024(单位是位，也就是 bit)，生成的密钥长度就是 1024位 / 8位/字节 = 128字节，那么我们需要加密的明文长度不能超过 128字节 -
11 字节 = 117字节。也就是说，我们最大能将 117 字节长度的明文进行加密，否则会出问题(抛诸如 javax.crypto.IllegalBlockSizeException: Data must not be longer than 53 bytes 的异常)。
而 BC 提供的加密算法能够支持到的 RSA 明文长度最长为密钥长度。
### byte[].toString() 返回的实际上是内存地址，不是将数组的实际内容转换为 String
警惕 toString 陷阱：Java 中数组的 toString() 方法返回的并非数组内容，它返回的实际上是数组存储元素的类型以及数组在内存的位置的一个标识。
大部分人跌入这个误区而不自知，包括一些写了多年 Java 的老鸟。比如这篇博客《[How To Convert Byte[] Array To String In Java](http://www.mkyong.com/java/how-do-convert-byte-array-to-string-in-java/)》中的代码
```
public class TestByte
{    
    public static void main(String[] argv) {
 
            String example = "This is an example";
            byte[] bytes = example.getBytes();
 
            System.out.println("Text : " + example);
            System.out.println("Text [Byte Format] : " + bytes);
            System.out.println("Text [Byte Format] : " + bytes.toString());
 
            String s = new String(bytes);
            System.out.println("Text Decryted : " + s);
 
 
    }
}
```
输出：
Text : This is an example
Text [Byte Format] : [B@187aeca
Text [Byte Format] : [B@187aeca
Text Decryted : This is an example
这些输出其实都是字节数组在内存的位置的一个标识，而不是作者所认为的字节数组转换成的字符串内容。如果我们对密钥以 byte[].toString() 进行持久化存储或者和其他一些字符串打 json 传输，那么密钥的解密者得到的将只是一串毫无意义的字符，当他解码的时候很可能会遇到 "javax.crypto.BadPaddingException" 异常。
### 字符串用以保存文本信息，字节数组用以保存二进制数据
java.lang.String 保存明文，byte 数组保存二进制密文，在 java.lang.String 和 byte[] 之间不应该具备互相转换。如果你确实必须得使用 java.lang.String 来持有这些二进制数据的话，最安全的方式是使用 Base64(推荐 Apache 的 commons-codec 库的 org.apache.commons.codec.binary.Base64)：
```
      // use String to hold cipher binary data
      Base64 base64 = new Base64(); 
      String cipherTextBase64 = base64.encodeToString(cipherText);
      
      // get cipher binary data back from String
      byte[] cipherTextArray = base64.decode(cipherTextBase64);
```
### 每次生成的密文都不一致证明你选用的加密算法很安全
一个优秀的加密必须每次生成的密文都不一致，即使每次你的明文一样、使用同一个公钥。因为这样才能把明文信息更安全地隐藏起来。
    Java 默认的 RSA 实现是 <font color="red">"RSA/None/PKCS1Padding"</font>(比如 Cipher cipher = Cipher.getInstance("<font color="red">RSA</font>");句，这个 Cipher 生成的密文总是不一致的)，Bouncy Castle 的默认 RSA 实现是 "RSA/None/NoPadding"。
    为什么Java默认的RSA实现每次生成的密文都不一致呢，即使每次使用同一个明文、同一个公钥？<font color="red">这是因为 RSA 的 PKCS #1 padding 方案在加密前对明文信息进行了随机数填充。</font>
你可以使用以下办法让同一个明文、同一个公钥每次生成同一个密文，但是你必须意识到你这么做付出的代价是什么。比如，你可能使用RSA来加密传输，但是由于你的同一明文每次生成的同一密文，攻击者能够据此识别到同一个信息都是何时被发送。
```
    Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());
    final Cipher cipher = Cipher.getInstance("RSA/None/NoPadding", "BC");
```
### 可以通过调整算法提供者来减小密文长度
Java 默认的 RSA 实现 "RSA/None/PKCS1Padding" 要求最小密钥长度为 512 位(否则会报 java.security.InvalidParameterException: RSA keys must be at least 512 bits long 异常)，也就是说生成的密钥、密文长度最小为 64 个字节。如果你还嫌大，可以通过调整算法提供者来减小密文长度:
```
Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());
final KeyPairGenerator keyGen = KeyPairGenerator.getInstance("RSA", "BC");
keyGen.initialize(128);
```
如此这般得到的密文长度为 128 位(16 个字节)。但是这么干之前请先回顾一下本文[修改秘钥长度](#u53EF_u4EE5_u901A_u8FC7_u4FEE_u6539_u751F_u6210_u5BC6_u94A5_u7684_u957F_u5EA6_u6765_u8C03_u6574_u5BC6_u6587_u957F_u5EA6)所述。
### Cipher 是有状态的，而且是线程不安全的
javax.crypto.Cipher 是有状态的，不要把Cipher当做一个静态变量，除非你的程序是单线程的，也就是说你能够保证同一时刻只有一个线程在调用 Cipher。否则你可能会像笔者似的遇到 java.lang.ArrayIndexOutOfBoundsException: too much data for RSA block异常。遇见这个异常，你需要先确定你给 Cipher 加密的明文(或者需要解密的密文)是否过长；排除掉明文(或者密文)过长的情况，你需要考虑是不是你的 Cipher 线程不安全了。

虽然《[RSA Encryption Example](https://javadigest.wordpress.com/2012/08/26/rsa-encryption-example/)》存在一些认识上的误区，不过是一篇很不错的入门级文章。结合本文所列内容，笔者将其代码做了一些调整以供参考：
```
import java.io.File;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.NoSuchAlgorithmException;
import java.security.PrivateKey;
import java.security.PublicKey;
import java.security.Security;

import javax.crypto.Cipher;

import org.apache.commons.codec.binary.Base64;

/**
 * @author hua
 * 
 */
public class EncryptionUtil {

    /**
     * String to hold name of the encryption algorithm.
     */
    public static final String ALGORITHM = "RSA";

    /**
     * String to hold name of the encryption padding.
     */
    public static final String PADDING = "RSA/NONE/NoPadding";

    /**
     * String to hold name of the security provider.
     */
    public static final String PROVIDER = "BC";

    /**
     * String to hold the name of the private key file.
     */
    public static final String PRIVATE_KEY_FILE = "e:/defonds/work/20150116/private.key";

    /**
     * String to hold name of the public key file.
     */
    public static final String PUBLIC_KEY_FILE = "e:/defonds/work/20150116/public.key";

    /**
     * Generate key which contains a pair of private and public key using 1024
     * bytes. Store the set of keys in Prvate.key and Public.key files.
     * 
     * @throws NoSuchAlgorithmException
     * @throws IOException
     * @throws FileNotFoundException
     */
    public static void generateKey() {
        try {

            Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());
            final KeyPairGenerator keyGen = KeyPairGenerator.getInstance(
                    ALGORITHM, PROVIDER);
            keyGen.initialize(256);
            final KeyPair key = keyGen.generateKeyPair();

            File privateKeyFile = new File(PRIVATE_KEY_FILE);
            File publicKeyFile = new File(PUBLIC_KEY_FILE);

            // Create files to store public and private key
            if (privateKeyFile.getParentFile() != null) {
                privateKeyFile.getParentFile().mkdirs();
            }
            privateKeyFile.createNewFile();

            if (publicKeyFile.getParentFile() != null) {
                publicKeyFile.getParentFile().mkdirs();
            }
            publicKeyFile.createNewFile();

            // Saving the Public key in a file
            ObjectOutputStream publicKeyOS = new ObjectOutputStream(
                    new FileOutputStream(publicKeyFile));
            publicKeyOS.writeObject(key.getPublic());
            publicKeyOS.close();

            // Saving the Private key in a file
            ObjectOutputStream privateKeyOS = new ObjectOutputStream(
                    new FileOutputStream(privateKeyFile));
            privateKeyOS.writeObject(key.getPrivate());
            privateKeyOS.close();
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    /**
     * The method checks if the pair of public and private key has been
     * generated.
     * 
     * @return flag indicating if the pair of keys were generated.
     */
    public static boolean areKeysPresent() {

        File privateKey = new File(PRIVATE_KEY_FILE);
        File publicKey = new File(PUBLIC_KEY_FILE);

        if (privateKey.exists() && publicKey.exists()) {
            return true;
        }
        return false;
    }

    /**
     * Encrypt the plain text using public key.
     * 
     * @param text
     *            : original plain text
     * @param key
     *            :The public key
     * @return Encrypted text
     * @throws java.lang.Exception
     */
    public static byte[] encrypt(String text, PublicKey key) {
        byte[] cipherText = null;
        try {
            // get an RSA cipher object and print the provider
            Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());
            final Cipher cipher = Cipher.getInstance(PADDING, PROVIDER);
            
            // encrypt the plain text using the public key
            cipher.init(Cipher.ENCRYPT_MODE, key);
            cipherText = cipher.doFinal(text.getBytes());
        } catch (Exception e) {
            e.printStackTrace();
        }
        return cipherText;
    }

    /**
     * Decrypt text using private key.
     * 
     * @param text
     *            :encrypted text
     * @param key
     *            :The private key
     * @return plain text
     * @throws java.lang.Exception
     */
    public static String decrypt(byte[] text, PrivateKey key) {
        byte[] dectyptedText = null;
        try {
            // get an RSA cipher object and print the provider
            Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());
            final Cipher cipher = Cipher.getInstance(PADDING, PROVIDER);

            // decrypt the text using the private key
            cipher.init(Cipher.DECRYPT_MODE, key);
            dectyptedText = cipher.doFinal(text);

        } catch (Exception ex) {
            ex.printStackTrace();
        }

        return new String(dectyptedText);
    }

    /**
     * Test the EncryptionUtil
     */
    public static void main(String[] args) {

        try {

            // Check if the pair of keys are present else generate those.
            if (!areKeysPresent()) {
                // Method generates a pair of keys using the RSA algorithm and
                // stores it
                // in their respective files
                generateKey();
            }

            final String originalText = "12345678901234567890123456789012";
            ObjectInputStream inputStream = null;

            // Encrypt the string using the public key
            inputStream = new ObjectInputStream(new FileInputStream(
                    PUBLIC_KEY_FILE));
            final PublicKey publicKey = (PublicKey) inputStream.readObject();
            final byte[] cipherText = encrypt(originalText, publicKey);

            // use String to hold cipher binary data
            Base64 base64 = new Base64();
            String cipherTextBase64 = base64.encodeToString(cipherText);

            // get cipher binary data back from String
            byte[] cipherTextArray = base64.decode(cipherTextBase64);

            // Decrypt the cipher text using the private key.
            inputStream = new ObjectInputStream(new FileInputStream(
                    PRIVATE_KEY_FILE));
            final PrivateKey privateKey = (PrivateKey) inputStream.readObject();
            final String plainText = decrypt(cipherTextArray, privateKey);

            // Printing the Original, Encrypted and Decrypted Text
            System.out.println("Original=" + originalText);
            System.out.println("Encrypted=" + cipherTextBase64);
            System.out.println("Decrypted=" + plainText);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

先生成一对密钥，供以后加解密使用(不需要每次加解密都生成一个密钥)，密钥长度为 256 位，也就是说生成密文长度都是 32 字节的，支持加密最大长度为 32 字节的明文，因为使用了 nopadding 所以对于同一密钥同一明文，本文总是生成一样的密文；然后使用生成的公钥对你提供的明文信息进行加密，生成 32 字节二进制明文，然后使用 Base64 将二进制密文转换为字符串保存；之后演示了如何把 Base64 字符串转换回二进制密文；最后把二进制密文转换成加密前的明文。以上程序输出如下：
Original=12345678901234567890123456789012
Encrypted=GTyX3nLO9vseMJ+RB/dNrZp9XEHCzFkHpgtaZKa8aCc=
Decrypted=12345678901234567890123456789012

## 参考资料
- http://blog.csdn.net/defonds/article/details/42775183
- http://www.bouncycastle.org/wiki/display/JA1/Frequently+Asked+Questions