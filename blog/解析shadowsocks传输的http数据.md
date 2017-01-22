<!--
author: vlean
date: 2017-01-22
title: 解析shadowsocks传输的http数据
tags: http,shadowsocks,decode
category: python
status: publish
summary: shadowsocks传输数据的解析
-->

## 0x00 shadowsocks
`shadowsocks`使用socket代理，拿到数据后将其加密传输到远程服务器，服务器发起数据的请求之后再将数据返回给客户端．想要对`shadowsocks`的请求数据进行解析的话，在服务端和本地转发数据的时候都可以捕获解析．为了方便调试，这里选取了本地进行调试．

`shadowsocks`的目录结构如下:

```dash
- shadowsocks
- - crypto #加密组件
- - asyncdns.py  #异步dns查询
- - common.py 
- - daemon.py
- - encrypt.py
- - local.py
- - eventloop.py
- - manager.py
- - shell.py
- - server.py
- - tcprelay.py #tcp转发
- - udprelay.py
```

## 0x01 trunked和gzip

本次要修改的地方就是`tcprelay.py`里读取本地数据和远程数据的地方，对于`http`请求头没有进行加密可以直接读取到．而远程传输回来的数据里有部分`http`请求使用了`gzip`压缩．例如某次返回的header信息如下：

```html
HTTP/1.1 200 OK
Server: nginx/1.4.6 (Ubuntu)
Date: Sun, 22 Jan 2017 16:08:38 GMT
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
X-Powered-By: PHP/7.0.13
Content-Encoding: gzip
```

其中`Transfer-Encoding: chunked`和`Content-Encoding: gzip`指明了html的压缩格式为gzip，并且分段传输．要解析其返回的数据就要先把分段的数据整合起来，再使用gzip解压．其中[chunked](https://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.6.1)的格式如下:

```html
Chunked-Body   = *chunk
                last-chunk
                trailer
                CRLF
chunk          = chunk-size [ chunk-extension ] CRLF
                chunk-data CRLF
chunk-size     = 1*HEX
last-chunk     = 1*("0") [ chunk-extension ] CRLF
chunk-extension= *( ";" chunk-ext-name [ "=" chunk-ext-val ] )
chunk-ext-name = token
chunk-ext-val  = token | quoted-string
chunk-data     = chunk-size(OCTET)
trailer        = *(entity-header CRLF)
```

chunked编码使用若干个Chunk组成，由一个标明长度为0的chunk结束，每个Chunk有两部分组成，每个部分用回车换行隔开。在最后一个长度为0的Chunk中的内容是称为footer的内容，是一些没有写的头部内容。所以所谓的chunked编码是如下的格式： 

如果一个HTTP消息（请求消息或应答消息）的Transfer-Encoding消息头的值为chunked，那么，消息体由数量未定的块组成，并以最后一个大小为0的块为结束。

每一个非空的块都以该块包含数据的字节数（字节数以十六进制表示）开始，跟随一个CRLF （回车及换行），然后是数据本身，最后块CRLF结束。在一些实现中，块大小和CRLF之间填充有白空格（0x20）。

最后一块是单行，由块大小（0），一些可选的填充白空格，以及CRLF。最后一块不再包含任何数据，但是可以发送可选的尾部，包括消息头字段。消息最后以CRLF结尾。

第一个chunk数据的字节数+/r/n+第一块chunk的数据 +/r/n+第二个chunk的数据的字节数+/r/n+第二块chunk的数据+n个chunk+/r/n+0+/r/n。

## 0x02 解析数据

```python
#...
if self._is_local:
    data = self._encryptor.decrypt(data)
    htmlData = data.split('\r\n\r\n')
    htmlBody = data[1]
    chunks = htmlBody.split('\r\n')
    content = '\r\n'.join(chunks[1:3]) #对于内容不太长的一次会返回全部内容，所以去掉chunked的长度和结尾数据即可
    
    # 对gzip格式的解压可以直接使用python的gzip扩展
    # import gzip
    # from cStringIO import StringIO
    compressedstream = StringIO(content)
    gzipper = gzip.GzipFile(fileobj=compressedstream)
    realHtml = gzipper.read()
```

***运行效果如下***

```html
2017-01-23 00:49:38 VERBOSE  rec header: 
HTTP/1.1 200 OK
Server: nginx/1.4.6 (Ubuntu)
Date: Sun, 22 Jan 2017 16:49:37 GMT
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
X-Powered-By: PHP/7.0.13
Content-Encoding: gzip 
2017-01-23 00:49:38 VERBOSE  rec content: 
<!DOCTYPE html>
<html>
	<head>
	<meta charset="utf-8">
	<meta http-equiv="X-UA-Compatible" content="IE=edge">
	<meta name="viewport" content="width=device-width, initial-scale=1">
	<title>PHP多进程|Dolinpa </title>
	<meta name="keywords" content="php,多进程,进程通信,php扩展,dolinpa,vlean,php,IT">
	<meta name="description" content="php多进程的开启和调用。">
	<link rel="stylesheet" href="/theme/simple/css/main.css?ver=2.2">
	<link rel="alternate" type="application/rss+xml" title="Dolinpa" href="//feed.xml" />
	
	</head>
....	
```

