---
title: 深入Http协议(一)
categories: 学习
tags: [http,java,spring]
---

## start

&emsp;&emsp;搞了很久的SOA平台开发,时间倒是浪费了很久,但是真正学的或是进步却是少之又少.目前做一个API网关,其主要功能是作为请求效验,日志记录和请求转发.java实在是不太适合做这种工作,用python能很简单实现的事情,换做java实在是很麻烦.

- - -

<!--more-->

## music

<center>
  <iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=90% height=100 src="https://music.daoapp.io/iframe?song=26060065&qssl=1&qlrc=0&narrow=1&autoplay=1"></iframe>
</center>

- - -

## content

### 了解HTTP协议

1. HTTP是Hyper Text Transfer Protocol（超文本传输协议）的缩写,用于从WWW服务器传输超文本到本地浏览器的传送协议.它可以使浏览器更加高效，使网络传输减少.

2. HTTP是一个应用层协议,由请求和响应构成,是一个标准的客户端服务器模型（基于TCP/IP）,是一个无状态的协议.

3. 默认HTTP的端口号为80,HTTPS的端口号为443.目前我们使用的HTTP/1.1版本.

### HTTP请求流程

1. 首先客户机与服务器需要建立连接.只要单击某个超级链接,HTTP的工作开始.

2. 建立连接后,客户机发送一个请求给服务器,请求方式的格式为:统一资源标识符（URL）、协议版本号,后边是MIME信息包括请求修饰符、客户机信息和可能的内容.

3. 服务器接到请求后,给予相应的响应信息,其格式为一个状态行,包括信息的协议版本号、一个成功或错误的代码,后边是MIME信息包括服务器信息、实体信息和可能的内容.

4. 客户端接收服务器所返回的信息通过浏览器显示在用户的显示屏上，然后客户机与服务器断开连接.

### HTTP报文格式示例

&emsp;&emsp;http报文分3部分:请求行/响应行(line)、请求头/响应头(header)、请求体/响应体(body--如果method为get请求体非必须).
```
GET /AdminCategory/List?pageNow=5&pageSize=20 HTTP/1.1

Host: tianyouduo.com
Connection: keep-alive
Cache-Control: max-age=0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) 
Chrome/45.0.2454.101 Safari/537.36
Accept-Encoding: gzip, deflate, sdch
Accept-Language: zh-CN,zh;q=0.8
Cookie: ASP.NET_SessionId=j3i3cec4ezl1k33zw44ueibn
```

### 响应码说明

- 1xx：信息，请求收到，继续处理 
- 2xx：成功，行为被成功地接受、理解和采纳 
- 3xx：重定向，为了完成请求，必须进一步执行的动作 
- 4xx：客户端错误，请求包含语法错误或者请求无法实现 
- 5xx：服务器错误，服务器不能实现一种明显无效的请求

具体响应码和请求方法在之后restful规范时会详细说明.

## end

&emsp;&emsp;就这样吧,先起个头开坑,同事用python做请求转发非常方便快捷,我用java倒是困难重重,最显而易见的问题是文件转发,要本地创建临时文件,构造请求转发,再删除临时文件.这其中的效率可想而知.更加麻烦的时,对于java而已,一般用springMVC接收请求,但是构造请求要想传个相同的请求很是麻烦,尤其对于表单类请求.

















