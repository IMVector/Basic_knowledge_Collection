HTTP报文格式详解

[原文](https://juejin.cn/post/6875936016495345678)

HTTP报文是面向文本的，报文中的每一个字段都是一些ASCII码串，每个字段的长度是不确定的。HTTP报文传过来的都是一堆的0x ASCII码，例如" 41 63 63 65 70 74"这段十六进制ASCII码串对应的是“accept” 单词。

这些十六进制的数字经过浏览器或者专用工具比如wireshark的翻译，可以得到HTTP的报文结构。

HTTP有两种报文：请求报文和响应报文。

# 请求报文

以下是wireshark抓出来的一段HTTP请求报文

```
GET /admin_ui/rdx/core/images/close.png HTTP/1.1
Accept: */*
Referer: http://xxx.xxx.xxx.xxx/menu/neo
Accept-Language: en-US
User-Agent: Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.1; WOW64; Trident/7.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; .NET4.0C; .NET4.0E)
Accept-Encoding: gzip, deflate
Host: xxx.xxx.xxx.xxx
Connection: Keep-Alive
Cookie: startupapp=neo; is_cisco_platform=0; rdx_pagination_size=250%20Per%20Page; SESSID=deb31b8eb9ca68a514cf55777744e339
```

HTTP的请求报文由四部分组成：请求行(request line)、请求头部(header)、空行和请求数据(request data)

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7ce8e2208bd24d66b0f19c548015510f~tplv-k3u1fbpfcp-zoom-1.image)

- 请求行：由请求方法、URL(包含参数)和协议版本组成
- 请求头部：由多个key-value值组成
- 空行：请求报文使用空行将请求头部和请求数据分隔
- 请求数据：GET方法没有携带数据，POST方法会携带一个body

# 响应报文

下面是wireshark抓出来的一段响应报文

```
HTTP/1.1 200 OK
Bdpagetype: 1
Bdqid: 0xacbbb9d800005133
Cache-Control: private
Connection: Keep-Alive
Content-Encoding: gzip
Content-Type: text/html
Cxy_all: baidu+f8b5e5b521b3644ef7f3455ea441c5d0
Date: Fri, 12 Oct 2018 06:36:28 GMT
Expires: Fri, 12 Oct 2018 06:36:26 GMT
Server: BWS/1.1
Set-Cookie: delPer=0; path=/; domain=.baidu.com
Set-Cookie: BDSVRTM=0; path=/
Set-Cookie: BD_HOME=0; path=/
Set-Cookie: H_PS_PSSID=1433_21112_18560_26350_27245_22158; path=/; domain=.baidu.com
Vary: Accept-Encoding
X-Ua-Compatible: IE=Edge,chrome=1
Transfer-Encoding: chunked

<!DOCTYPE html>
<!--STATUS OK-->
```

HTTP的响应报文由四部分组成：状态行、响应头、空行、响应体(数据) ![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/94a0bdb5295342658866b09ab90ab363~tplv-k3u1fbpfcp-zoom-1.image)

- 状态行：由协议版本、状态码和状态值组成
- 响应头：由多个key-value值组成
- 空行：响应报文使用空行将响应头和响应体分隔
- 响应体：响应数据，在上面是一段HTML