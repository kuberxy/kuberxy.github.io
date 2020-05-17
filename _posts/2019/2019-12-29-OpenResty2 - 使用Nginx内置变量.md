---
layout: post
title: OpenResty - 使用Nginx内置变量
date:   2019-12-29 10:02:42 +0800
category: OpenResty
tags: [HTTP]
typora-root-url: ../..
---





在OpenResty中，可以直接引用Nginx内置的变量。这些变量大部分都是在请求进入时解析的，并且在请求的整个生命周期中会一直被缓存着，以方便获取使用。

## 常用变量一览

Nginx提供了很多内置变量，常用的如下：

| 变量                | 含义                                                      |
| ------------------- | --------------------------------------------------------- |
| $remote_addr        | 客户端IP地址                                              |
| $remote_port        | 客户端端口号                                              |
| $limit_rate         | 限制请求的数据传输速度                                    |
| $server_name        | 接收请求的server_name                                     |
| $host               | 请求头中Host字段的值或匹配请求的server name               |
| $binary_remote_addr | 二进制形式的客户端地址（IPv4的长度为4B，IPv6的长度为16B） |
|                     |                                                           |
|                     |                                                           |
|                     |                                                           |
|                     |                                                           |



## 使用变量

### 语法

```lua
-- syntax: 
ngx.var.VAR_NAME

-- context: 
--     set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, 
--     header_filter_by_lua*, body_filter_by_lua*, log_by_lua*
```



### 示例：实现一个求和运算

```nginx
location /sum {
    # 处理业务
    content_by_lua_block {
        local a = tonumber(ngx.var.arg_a) or 0
        local b = tonumber(ngx.var.arg_b) or 0
        ngx.say("sum: ", a + b)
    }
}
```

访问结果：

```shell
$ curl 'http://127.0.0.1/sum'
sum: 0

$ curl 'http://127.0.0.1/sum?a=11&b=14'
sum: 25
```



### 示例：实现一个简单的防火墙

```nginx
location /sum {
    # 通过access阶段,完成请求进入时的处理
    access_by_lua_block {
        local black_ips = {["127.0.0.1"] = true}
        
        local ip = ngx.var.remote_addr
        if black_ips[ip] == true then
            ngx.exit(ngx.HTTP_FORBIDDEN)
        end
    }
    
    # 处理业务
    content_by_lua_block {
        local a = tonumber(ngx.var.arg_a) or 0
        local b = tonumber(ngx.var.arg_b) or 0
        ngx.say('sum: ', a + b)
    }
}
```

访问结果：

```shell
$ curl '192.168.1.250/sum?a=11&b=12'
sum: 23

$ curl '127.0.0.1/sum?a=11&b=12'
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>openresty/1.15.8.2</center>
</body>
</html>
```

可以看到，提取了客户端`IP`地址后进行了限制。在扩充一下，就可以支持 IP 地址段，如果与系统`iptables`进行配合，那么就足以达到软件防火墙的目的。 



### 示例：修改变量的值

> 上面的示例都是在读写Nginx内置的变量，那么是否可以修改这些变量呢？

答案是，Nginx的内置变量大多数是不允许修改的。在可修改的变量中，常用的可能就只有与限速相关的变量了，比如`limit_rate`，它用于限制数据传输的速度，并且它影响的是单个请求级别。

```nginx
location /download {
    access_by_lua_block {
        ngx.var.limit_rate = 1000
    }
}
```

访问结果：

```lua
$ wget '127.0.0.1/download/1.html'
--2020-02-21 10:08:08--  http://127.0.0.1/download/1.html
Connecting to 127.0.0.1:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1024000 (1000K) [text/html]
Saving to: ‘1.html’

1.html                          0%[                                                  ]   5.62K   965 B/s    eta 17m 35s
```

可以看到，下载速度被限制成了1000B/s



## 参考

https://moonbingbing.gitbooks.io/openresty-best-practices/content/













