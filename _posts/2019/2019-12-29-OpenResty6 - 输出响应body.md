---
layout: post
title: OpenResty - 输出响应body
date:   2019-12-29 10:02:42 +0800
category: OpenResty
tags: [HTTP]
typora-root-url: ../..
---





## ngx.say & ngx.print

`ngx.say`和`ngx.print`，这两个函数都是异步的，即调用它们后不会立即输出响应主体。

```nginx
server {
    listen 80;
    
    location /test {
        content_by_lua_block {
            ngx.say('hello')
            ngx.sleep(3)
            ngx.say("the world")
        }
    }
    
    location /test2 {
        content_by_lua_block {
            ngx.say('hello')
            -- 显示地向客户端刷新响应输出
            ngx,flush()
            ngx.sleep(3)
            ngx.say('the world')
        }
    }
}
```

访问结果：

```shell
$ curl 127.0.0.1/test
(sleep 3)
hello 
the world

$ curl 127.0.0.1/test2
hello
(sleep 3)
then world
```

如果仔细观察结果输出的过程会看到，请求`/test`s时会等待3s然后一起接收到响应主体，而请求`/test2`则是先收到一个`hello`停顿3s后又接收到后面的`the world`。

我们再来看一个示例：

```nginx
server {
    listen 80;
    
    lua_code_cache off;
    
    location /test {
        content_by_lua_block {
            ngx.say(string.rep('hello', 1000))
            ngx.sleep(3)
            ngx.say('the world')
        }
    }
}
```

访问结果：

```lua
$ curl 127.0.0.1/test
hello
hello
hello
...
hello
hello
hello
(sleep 3)

then world
```

仔细观察结果输出的过程可以看到，首先收到了所有的“hello”，然后停顿了大概3秒后，收到了“the world”。

因为是异步输出，所以两个响应主体的输出时机是不一样的。



## 如何处理响应主体过大的输出

当响应主体比较小时，相对比较随意。但当响应主体过大时，是不能直接调用API输出响应主体的。响应主体过大，可以分为两种情况：

- 要输出的内容本身体积过大。比如，超过2G的文件下载。
- 要输出的内容是由各种碎片拼凑的，而碎片数量非常庞大。比如，响应数据是某地区所有人的姓名。



### chunked编码

对于第一种情况，可以使用HTTP1.1中的chunked编码来解决。chunked编码，就是把一个大的响应主体拆分成多个小的响应主体，然后分批、有节制的响应给请求方。

```nginx
location /test {
    content_by_lua_block {
        local file, err = io.open(ngx.config.prefix() .. "data.db", 'r')
        if not file then
            ngx.log(ngx.ERR, "open file error:", err)
            ngx.exit(ngx.HTTP_SERVICE_UNAVAILABLE)
        end
            
        local data
        while true do
            data = file:read(1024)
            if nil == data then
                break
            end
            ngx.print(data)
            ngx.flush(true)
        end
        file:close()
    }
}
```

按块（每次1KB）读取本地文件data.db的内容，并以流式方式进行响应。这样即使文件很大，nginx服务也可以稳定运行，并维持内存占用在几MB的范围。

> 注： nginx 自带的静态文件解析能力已经非常优秀了。这里只是一个例子，实际中过大的响应主体都是由后端服务生成的，这里为了演示，所以选择读取本地文件。 



### ngx.print

对于第二种情况，就可以利用`ngx.print`的特性了。`ngx.print`的输入参数可以是单个或多个字符串，也可以是table对象。如下，是官方的示例代码：

```nginx
local table = {
    "hello, ",
    {"world: ", true, " or ", false,
        {": ", nil}}
}

ngx.print(table)
```

输出结果：

```lua
hello, world: true or false: nil
```

这个示例是想说明，当有很多碎片数据时，没必要一定要连成字符串后再进行输出。完全可以直接将这些碎片数据存放在一个 table 中，用数组的方式把这些碎片数据统一起来，最后直接调用 `ngx.print(table)` 即可。这种方式效率更高，并且更容易被优化。



## 参考

https://moonbingbing.gitbooks.io/openresty-best-practices/content/













