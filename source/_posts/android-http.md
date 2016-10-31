title: Android HTTP请求
author: author2
date: 2016-10-31 17:27:00
tags:
- Android
- 网络
categories:
- Android
---
# HTTP协议
## HTTP协议历史
### HTTP 0.9 
- [Draft](http://www.w3.org/Protocols/HTTP/AsImplemented.html)
- 1991
- 请求地址形式为`http:// hostname[:port]/path[?searchwords]`。hostname可为域名/数字形式，端口默认80
- 数据格式为ASCII字符
- 一次数据传输完断开连接

<!-- more -->

请求：
```
GET http://www.test.com/path/index.html
```
响应：
```
<html>
<body>
<p>Hello World!</p>
</body>
</html>
```

### HTTP 1.0
- [RFC 1945](http://tools.ietf.org/html/rfc1945)
- 1996年
- 支持POST、HEAD方法
- 支持请求行、请求头(User-Agent、If-Modified-Since等)和实体(Entity Header/Body)
- 支持响应状态行、响应头和响应体
- 返回支持状态码

请求：
```
GET http://www.test.com/path/index.html HTTP/1.0
User-Agent: okhttp
```
响应：
```
HTTP/1.0 200 OK
Content-Type: text/html

<html>
<body>
<p>Hello World!</p>
</body>
</html>
```

### HTTP 1.1
- [RFC 2616](http://tools.ietf.org/html/rfc2616)
- 1999年
- 2014修订版[RFC 723X](http://tools.ietf.org/html/rfc7230)
- 支持长连接，避免每个请求都建立一个TCP连接，加重服务器负担和网络拥塞
- 扩展了1.0版本头部字段

请求：
```
GET /path/index.html HTTP/1.1
Host: http://www.test.com
User-Agent: okhttp
Connection: Keep-Alive
```
响应：
```
HTTP/1.1 200 OK
Content-Type: text/html

<html>
<body>
<p>Hello World!</p>
</body>
</html>
```

### HTTP 2 
- [RFC 7540](https://tools.ietf.org/html/rfc7540)
- 2015年
- 头部压缩
- Server Push
- ...

## 格式
目前主要使用1.1版本，需要注意的格式有：
### Request格式
请求行: method + path + version
头：
- 通用头(Cache-Control/Connection/Date等)，Request和Response共用，与请求实体无关。
- 请求头(Host/Accept-Encoding/If_Modified_Since/User-Agent等)，Request专用。
- 实体头(Content-Encoding/Content-Length/Content-Type/Expires/Last-Modified等)。请求携带body时，表明实体的元信息；或者用于自定义头部字段，与是否携带实体无关。
CRLF
[ body ]
> CRLF -- 回车换行符

### Response格式
状态行: version + code + message
头：
- 通用头，同Request
- 回复头(ETag/Server等)，Response专用。
- 实体头，同Request
CRLF
[ body ]

### MIME
POST请求时最好准确指定Body内容的格式。常见的形式有：
- `text/plain`，纯文本
- `application/x-www-form-urlencoded`，形如"name=tom&age=28"的key-value纯文本
- `application/json`，Json格式纯文本
- `image/jpeg`，JPEG图片

### 编码
- 请求行和请求头
特殊字符和中文字符需要编码。如url中查询参数`key=value`中`value`、自定义请求头的值。
- 携带字符串Body的请求(如POST)，发起请求时最好携带编码：
`ContentType: "text/html;charset=UTF-8"`
- 请求的返回结果Body形式是字符串，服务器最好携带编码
`ContentType: "text/html;charset=UTF-8"`

# Java HTTP
HTTP是应用层协议，基于传输层的TCP协议。TCP的Java实现是[Socket](http://docs.oracle.com/javase/tutorial/networking/sockets/definition.html)。
## Socket实现HTTP
Socket对HTTP GET请求的简单实现:
```java
Socket socket = new Socket();
URL url = new URL("http://sethfeng.github.io");
String host = url.getHost();
int port = url.getDefaultPort();
SocketAddress dest = new InetSocketAddress(host, port);
socket.connect(dest);

String path = "/index.html";
OutputStreamWriter streamWriter = new OutputStreamWriter(socket.getOutputStream());
BufferedWriter bufferedWriter = new BufferedWriter(streamWriter);
bufferedWriter.write("GET " + path + " HTTP/1.1\r\n");
bufferedWriter.write("Host: " + host + "\r\n");
bufferedWriter.write("\r\n");
bufferedWriter.flush();

InputStreamReader streamReader = new InputStreamReader(socket.getInputStream());
BufferedReader bufferedReader = new BufferedReader(streamReader);

StringBuilder result = new StringBuilder();
String line = null;
while ((line = bufferedReader.readLine()) != null) {
    result.append(line + "\n");
}    
```
直接用Socket实现HTTP请求，写起来太繁琐。基于Socket的HTTP协议有几个经典实现。
## Apache HttpClient
来自Apache开源项目HttpComponents(http://hc.apache.org)。
```java
HttpClient client = new DefaultHttpClient();  
HttpGet request = new HttpGet();  
request.setURI(new URI("http://sethfeng.github.io/index.html"));  
HttpResponse response = client.execute(request); 
in = new BufferedReader(new InputStreamReader(response.getEntity().getContent()));
...
```
## JDK HttpURLConnection
JDK的内部实现：URL、URLConnection、HttpURLConnection。
```java
URL url = new URL("http://sethfeng.github.io/index.html");
HttpURLConnection urlConnection =  url.openConnection();
if (urlConnection.getResponseCode() == HttpURLConnection.HTTP_OK) {
    BufferedReader in = new BufferedReader(new InputStreamReader(urlConnection.getInputStream()));
    StringBuilder result = new StringBuilder();
    String inputLine;
    while ((inputLine = in.readLine()) != null)
        result.append(line + "\n");
    in.close();
}
```
## Square OkHttp
自带线程池的HTTP请求封装，可同步/异步请求。
```java
OkHttpClient client = new OkHttpClient();
Request request = new Request.Builder()
        .url("http://sethfeng.github.io/index.html")
        .build();
Call call = client.newCall(request);
Response response = call.execute();
String result = response.body().string();
```

# Android HTTP
Android的Java层可以无缝使用以上Java的HTTP实现。Apache HTTP client在Android 2.x时代在用。Android 2.x之后已建议使用HttpURLConnection。[官方博客](http://android-developers.blogspot.jp/2011/09/androids-http-clients.html)
Android 4.4之后，HttpURLConnection底层实现已被OkHttp替换。
## 网络库 vs HTTP库
在讨论Android技术时，对Android网络库常有概念上的模糊。到底怎么样的库才算是Android网络库？广义的网络库应包含了Android上对各种网络协议的封装库，包括对HTTP请求、Socket请求、文件上传/下载、图片请求的封装。由于HTTP请求使用广泛，通常讨论到的网络库经常指的是HTTP库。现在的网络库又常常封装了异步线程、HTTP缓存、请求结果解析等功能。网络库Volley可以比较好的说明网络库和HTTP库的关系。Volley的HTTP库在Android 2.3之前默认使用HttpClient，之后默认使用HttpURLConnection，也可以搭配OkHttp使用；此外，Volley还支持了请求线程调度、缓存、自动重试等功能。
## 常见HTTP网络库
- [OkHttp](https://github.com/square/okhttp)
- [Volley](https://developer.android.com/training/volley/index.html)
- [Robospice](https://github.com/stephanenicolas/robospice)
- [android-async-http](https://github.com/loopj/android-async-http)
- [Retrofit](https://github.com/square/retrofit)

Retrofit同OkHttp一样，是一个纯Java库。Retrofit将HTTP请求用Java Interface表述，极大的简化了请求接口编程。

# 特殊的网络库
## Socket实现自定义协议
自行制定的协议，使用Socket实现TCP网络服务。
## 图片库
图片是Android一种特殊的数据资源，一般要完整展示到UI，处理起来耗流量、耗时、耗内存。
常见的库有Picasso、Glide、Fresco等。
## 文件上传/下载库
类似图片库，单次请求数据量可能很大。大文件不适合在内存一次性操作，最好用流。

# 技能get!
- 命令行telnet发送HTTP请求：
telnet github.com 80(CRLF)
GET /index.html HTTP/1.1(CRLF)
Host: github.com(CRLF)
(CRLF)
- Mac下模拟HTTP请求好用的客户端[Paw](https://paw.cloud)，发送HTTP请求。
- Mac下抓包工具[Charles](https://www.charlesproxy.com)，可设置代理，改Request、改Response，应有尽有。


# 参考
- http://www.w3.org/Protocols/HTTP/AsImplemented.html
- http://tools.ietf.org/html/rfc1945
- http://tools.ietf.org/html/rfc2616
- http://www.w3.org/Protocols/
- https://tools.ietf.org/html/rfc7540
- http://docs.oracle.com/javase/tutorial/networking/
- http://hc.apache.org
- http://www.tuicool.com/articles/mq2qm26
- http://www.jianshu.com/p/3141d4e46240
