---

layout: post
title: HTTP报文首部
date:   2020-01-30 10:02:42 +0800
category: HTTP
tags: [HTTP协议]
typora-root-url: ../..
---



##  HTTP报文结构

一个HTTP报文通常由报文首部、空行、报文主体三部分组成。其中，首部为客户端和服务器处理请求和响应提供所需要的信息；主体则用来存储所需要的用户和资源信息。

![image-20200215222138106](/assets/images/2020/image-20200215222138106.png)

在HTTP协议中，请求报文和响应报文一会包含HTTP首部，但它们所包含的内容是不同的。

## HTTP请求报文

![image-20200215231347885](/assets/images/2020/image-20200215231347885.png)

一个HTTP请求报文的首部通常由以下部分组成：

- 请求行（请求方法、URI、HTTP版本）

- 首部字段（通用、请求、实体等首部字段）

如下是访问https://www.baidu.com时，请求报文的首部信息。

```http
GET / HTTP/1.1
Host: www.baidu.com
Connection: keep-alive
Pragma: no-cache
Cache-Control: no-cache
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/72.0.3626.109 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,en-AU;q=0.8,en;q=0.7
Cookie: BAIDUID=6424EC7F0D91B403C6C1D9C24AEE6C46:FG=1; BIDUPSID=6424EC7F0D91B403C6C1D9C24AEE6C46; PSTM=1545475361; BD_UPN=12314353; BDUSS=JzTFhFS2hPaHk5MXB6ZTM2dkxCaXlPb3JKZ3E2eC1SQ1ZkT2N3Qi04NGFQWFZjQVFBQUFBJCQAAAAAAAAAAAEAAAD21CwicXF4aWF5YW4AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABqwTVwasE1cQX; MCITY=-%3A; yjs_js_security_passport=0ce81c2f8a031628dc3c6fb818e2ff43108ad82b_1581728058_js; BDORZ=B490B5EBF6F3CD402E515D22BCDA1598; BD_HOME=1; sug=3; sugstore=0; ORIGIN=0; bdime=0; BDRCVFR[feWj1Vr5u3D]=I67x6TjHwwYf0; delPer=0; BD_CK_SAM=1; PSINO=7; H_PS_PSSID=30745_1431_21090; H_PS_645EC=a023s50WL5N69NHcAr7N%2FQMlldgQFGpFL0rdXrExVHpY2HCjV5wIdY9zsRtl3%2Ba8rkcY; BDSVRTM=0
```



## HTTP响应报文

![image-20200215231323363](/assets/images/2020/image-20200215231323363.png)

一个HTTP响应报文的首部通常由以下部分组成：

- 状态行（HTTP版本、状态码）

- 首部字段（通用、响应、实体等首部字段）

如下是访问https://www.baidu.com时，返回的响应报文的首部信息。

```http
HTTP/1.1 200 OK
Bdpagetype: 2
Bdqid: 0xa4c0a0ef00014992
Cache-Control: private
Connection: keep-alive
Content-Encoding: gzip
Content-Type: text/html;charset=utf-8
Date: Sat, 15 Feb 2020 15:03:12 GMT
Expires: Sat, 15 Feb 2020 15:03:12 GMT
Server: BWS/1.1
Set-Cookie: BDSVRTM=84; path=/
Set-Cookie: BD_HOME=1; path=/
Set-Cookie: H_PS_PSSID=30745_1431_21090_30791; path=/; domain=.baidu.com
Strict-Transport-Security: max-age=172800
Traceid: 1581778992075782426611871665566106339730
X-Ua-Compatible: IE=Edge,chrome=1
Transfer-Encoding: chunked
```





