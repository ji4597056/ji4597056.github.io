---
title: https踩坑
categories: 技术
tags: [java,https]
---

## start

&emsp;&emsp;在一个月黑风高的夜晚,我正要下班回家,领导找到我说,你的聚合接口连K8的服务要走https验证.验证就验证吧,总共给了我四个文件,`app.crt`,`app.key`,`ca.crt`,`ca.key`.搞了两天,各种坑爹,终于解决,记录一下.

- - -

<!--more-->

- - -

## content

### 何为https

&emsp;&emsp;HTTPS（Hypertext Transfer Protocol over Secure Socket Layer，基于SSL的HTTP协议）使用了HTTP协议，但HTTPS使用不同于HTTP协议的默认端口及一个加密、身份验证层（HTTP与TCP之间）。这个协议的最初提供了身份验证与加密通信方法，现在它被广泛用于互联网上安全敏感的通信。HTTPS是一个安全通信通道，用于在客户计算机和服务器之间交换信息。它使用安全套接字层(SSL)进行信息交换，简单来说它是HTTP的安全版。它是由Netscape开发并内置于其浏览器中，用于对数据进行压缩和解压操作，并返回网络上传送回的结果。

HTTPS实际上应用了Netscape的安全全套接字层(SSL)作为HTTP应用层的子层。(HTTPS使用端口443，而不是象HTTP那样使用端口80来和TCP/IP进行通信。)SSL使用40 位关键字作为RC4流加密算法，这对于商业信息的加密是合适的。HTTPS和SSL支持使用X。509数字认证，如果需要的话用户可以确认发送者是谁。

客户端在使用HTTPS方式与Web服务器通信时有以下几个步骤。
（1）客户使用https的URL访问Web服务器，要求与Web服务器建立SSL连接。
（2）Web服务器收到客户端请求后，会将网站的证书信息（证书中包含公钥）传送一份给客户端。
（3）客户端的浏览器与Web服务器开始协商SSL连接的安全等级，也就是信息加密的等级。
（4）客户端的浏览器根据双方同意的安全等级，建立会话密钥，然后利用网站的公钥将会话密钥加密，并传送给网站。
（5）Web服务器利用自己的私钥解密出会话密钥。
（6）Web服务器利用会话密钥加密与客户端之间的通信

### 踩坑

&emsp;&emsp;上述那些话为复制粘贴,我的理解是服务端和客户端都要相互认证,因此客户端和服务端都需要证书,既然是双向认证,那么服务端凭什么能认证通过客户端的证书呢,所以说,客户端的证书也是服务端发的.这个认证过程采用非对称加密,当双方验证成功,再由双方协议生成个秘钥用于之后通讯的对称加密.

&emsp;&emsp;在java代码层面,可以看到这个证书称之为`Keystone`,载入服务端证书的是`KeyManagerFactory`,载入客户端证书的是`TrustManagerFactory`.

&emsp;&emsp;那么问题来了,到底证书是什么?用golang写https服务端、https客户端非常方便,直接用开始我提到的crt文件和key文件路径作为参数就可以构建客户端和服务端.那么java也支持么?很遗憾不支持.网上也有人说过用`p12`文件(pkcs类型)做证书参数,但是经过尝试依然失败,经过无数次失败,最终发现java似乎只支持`jks`文件证书.

```java
 public String certificateValidation() throws Exception {
		// 客户端证书
        String caFile = CA_FILE_PATH;
        // 服务端证书
        String appFile = APP_FILE_PATH;
        KeyStore clientTrustKeyStore = KeyStore.getInstance(KeyStore.getDefaultType());
        clientTrustKeyStore.load(new FileInputStream(caFile), CA_PASSWORD.toCharArray());
        KeyStore serverStore = KeyStore.getInstance(KeyStore.getDefaultType());
        serverStore.load(new FileInputStream(appFile), APP_PASSWORD.toCharArray());
		// TrustManagerFactory
        TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
        trustManagerFactory.init(clientTrustKeyStore);
		// KeyManagerFactory
        KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
        keyManagerFactory.init(serverStore, APP_PASSWORD.toCharArray());
		// SSLContext
        SSLContext sslContext = SSLContext.getInstance("SSL");
        sslContext.init(keyManagerFactory.getKeyManagers(), trustManagerFactory.getTrustManagers(), new SecureRandom());
        return send(new SSLConnectionSocketFactory(sslContext));
    }
```

```java
private static String send(LayeredConnectionSocketFactory sslSocketFactory) throws IOException {
        CloseableHttpClient httpclient = HttpClients.custom()
                .setSSLSocketFactory(sslSocketFactory)
                .build();

        HttpGet request = new HttpGet(url);
        request.setHeader("User-Agent", "Mozilla/5.0 (Windows NT 6.1; WOW64) ...");

        CloseableHttpResponse response = httpclient.execute(request);
        String responseBody = readResponseBody(response.getEntity());
        System.out.println(responseBody);
        return responseBody;
    }
```

&emsp;&emsp;可以看出其实这么大费周章的只是为了通过证书生成`SSLConnectionSocketFactory`,然后在`HttpClient`中传入`SSLConnectionSocketFactory`,就可以进行https请求了.

### 怎么生成证书

&emsp;&emsp;jdk自带的工具`keytool`可以直接生成`jks`类型证书.但是往往我们的证书是非java服务端,所以就出现开始我提到的给了我crt文件和key文件.这两种类型文件不能直接生成jks文件,需要先转p12文件再转jks文件,转换过程中用到的密码即为代码中载入jks文件时需要传入的密码参数.

**crt/key -> p12**
`openssl pkcs12 -export -in app.crt -inkey app.key -out app.p12 -name app -passin pass:123456 -passout pass:123456`

**p12 -> jks**
`keytool -importkeystore -v  -srckeystore app.p12 -srcstoretype pkcs12 -srcstorepass 123456 -destkeystore app.jks -deststoretype jks -deststorepass 123456 `

### end

&emsp;&emsp;我并没有深入了解https,只是大致了解下过程,以及怎么用代码去解决问题.不过这次踩坑给我最大的收货是要敢于去尝试,因为谷歌了网上很多的解决方案,但都没有试成功,因为我并不知道java里的`Keystone`只支持jks类型的证书,只是不断的去尝试,去断点调试才最终瞎猫碰到死耗子解决了问题.
