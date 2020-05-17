---
layout: post
title: OpenResty - Lua & Nginx Location
date:   2019-12-29 10:02:42 +0800
category: OpenResty
tags: [HTTP]
typora-root-url: ../..
---



在Nginx的世界中，location的功能是异常强大的。而通过Nginx Location + Lua的组合，

我们可以完成内部调用、流水线方法跳转、外部重定向等功能。

## 内部调用

对于数据库、内部公共函数等接口，我们可以把它们放到统一的location中。而在通常情况下，为了保护这些内部接口，会把这些内部接口设置为 internal ，这么做的最主要好处就是可以让这些内部接口相对独立，不受外界干扰。 

```nginx
location = /sum {
    # 只允许内部调用，直接访问返回404
    internal;
    
    # 这里只是做了一个求和运算的例子。
    # 在实际应用中，可以在这里完成一些数据库、缓存服务器的操作，
    # 来达到基础模块和业务逻辑分离目的。
    content_by_lua_block {
        local args = ngx.req.get_uri_args()
        ngx.say(tonumber(args.a) + tonumber(args.b))
    }
}

location = /app/test {
    content_by_lua_block {
        local res = ngx.location.capture("/sum", {args = {a=3,b=8}})
        ngx.say("status: ", res.status, " response: ",res.body)
    }
}
```

访问结果：

```shell
$ curl 127.0.0.1/sum
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>openresty/1.15.8.2</center>
</body>
</html>

$ curl 127.0.0.1/app/test
status: 200 response: 11
```

如上，当我们直接访问`/sum`时，会返回404；而当我们访问`/app/test`时，它会进而访问`/sum`。

我们在稍微扩充一下，实现并行请求的效果，示例如下：

```nginx
location = /sum {
    internal;
    
    content_by_lua_block {
        ngx.sleep(0.1)
        local args = ngx.req.get_uri_args()
        ngx.print(tonumber(args.a) + tonumber(args.b))
    }
}

location = /subductin {
    internal;
    
    content_by_lua_block {
        ngx.sleep(0.1)
        local args = ngx.req.get_uri_args()
        ngx.print(tonumber(args.a) - tonumber(args.b))
    }
}

location = /app/test_parallels {
    content_by_lua_block {
        local start_time = ngx.now()
        local res1,res2 = ngx.location.capture_multi({
            {"/sum",{args = {a=3,b=8}}},
                {"/subduction",{args={a=3,b=8}}}
        })
        ngx.say("status: ",res1.status," response: ",res1.body)
        ngx.say("status: ",res2.status," response: ",res2.body)
        ngx.say("time used: ", ngx.now() - start_time)
    }
}

location = /app/test_queue {
    content_by_lua_block {
        local start_time = ngx.now()
        local res1 = ngx.location.capture_multi({
            {"/sum",{args={a=3,b=8}}}
        })
        local res2 = ngx.location.capture_multi({
            {"/subduction",{args={a=3,b=8}}}
        })
        ngx.say("status: ",res1.status," response: ",res1.body)
        ngx.say("status: ",res2.status," response: ",res2.body)
        ngx.say("time used: ",ngx.now() - start_time)
    }
}
```

访问结果：

```shell
$ curl 127.0.0.1/app/test_parallels
status: 200 response: 11
status: 200 response: -5
time used: 0.1010000705719

$ curl 127.0.0.1/app/test_queue
status: 200 response: 11
status: 200 response: -5
time used: 0.20500016212463
```

如上，利用 `ngx.location.capture_multi` 函数，直接完成了两个子请求的并行执行。

当两个请求没有依赖关系，通过并行执行可以极大提高查询效率。假如两个无依赖请求，各自是 100ms，顺序执行需要 200ms，但通过并行执行可以在 100ms 完成两个请求。实际生产中查询时间可能没这么规整，但思想大同小异，这个特性是很有用的。 

并行执行，可以被广泛应用于广告系统（1：N模型，即一个请求，后端从N家供应商中获取条件最优的广告）、高并发前端页面展示（并行无依赖界面、降级开关等）。 



## 流水线方式跳转

现在的网络请求，已经变得越来越拥挤。各种不同的 API 、下载请求混杂在一起，这就要求不同服务端对下载的动态调整有不同的策略，而这些策略在一天中的不同时间段，规则可能还不一样。此时，我们可以效仿工厂的流水线模式，逐层过滤、处理。 

```lua
location ~ ^/static/([-_a-zA-Z0-9/]+).jpg {
    set image_name $1;
    
    content_by_lua_block {
        ngx.exex("/download_internal/images/" .. ngx.var.image_name .. ".jps");
    };
}

location /download_internal {
    internal;
    # 这里还可以有其他统一的 download 下载设置，例如限速等
    alias ../download;
}
```

这里的两个 location 更像是流水线上工人之间的协作关系。第一环节的工人在完成了自己该处理的部分后，直接交给了第二环节的处理人（实际上可以有更多环节），它们之间的数据流是定向的。

> 注意，ngx.exec 方法与 ngx.redirect 是完全不同的，前者是纯粹的内部跳转并且没有引入任何额外 HTTP 信号。



## 外部重定向

不知道大家有没有注意到，百度的首页已经不再是 HTTP 协议了，它已经全面修改到了 HTTPS 协议。但是对于大家的输入习惯，估计还是在地址栏里面输入 `baidu.com` ，回车后发现它会自动跳转到 `https://www.baidu.com` ，这时候就需要的外部重定向了。 

```lua
location = /foo {
    content_by_lua_block {
        ngx.say([[I am foo]])
    }
}

location = / {
    rewrite_by_lua_block {
        return ngx.redirect("/foo");
    }
}
```

运行结果：

```shell
$ curl 127.0.0.1 -i
HTTP/1.1 302 Moved Temporarily
Server: openresty/1.15.8.2
Date: Thu, 20 Feb 2020 13:59:31 GMT
Content-Type: text/html
Content-Length: 151
Connection: keep-alive
Location: /foo

<html>
<head><title>302 Found</title></head>
<body>
<center><h1>302 Found</h1></center>
<hr><center>openresty/1.15.8.2</center>
</body>
</html>

$ curl 127.0.0.1/foo -i
HTTP/1.1 200 OK
Server: openresty/1.15.8.2
Date: Thu, 20 Feb 2020 13:59:42 GMT
Content-Type: text/plain
Transfer-Encoding: chunked
Connection: keep-alive

I am foo
```

当我们使用浏览器访问页面 `http://127.0.0.1` 就可以发现浏览器会自动跳转到 `http://127.0.0.1/foo` 。

与前两个应用示例不同的是，外部重定向是可以跨域名的。例如从 A 网站跳转到 B 网站是绝对允许的。在 CDN 场景的大量下载应用中，一般分为调度、存储两个重要环节。调度就是通过根据请求方 IP 、下载文件等信息寻找最近、最快节点，应答跳转给请求方完成下载。 



## 参考

https://moonbingbing.gitbooks.io/openresty-best-practices/content/













