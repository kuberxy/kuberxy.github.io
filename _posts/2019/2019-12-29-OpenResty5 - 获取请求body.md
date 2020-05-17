---
layout: post
title: OpenResty - 获取请求body
date:   2019-12-29 10:02:42 +0800
category: OpenResty
tags: [HTTP]
typora-root-url: ../..
---



在 Nginx 的典型应用场景中，几乎都是只读取 HTTP的请求头，比如，负载均衡、正反向代理等场景。而对于 API Server 或者 Web Application来说，body就非常重要了。由于 OpenResty是基于 nginx 的，因此，它天然的对body 的读取细节与其他成熟 Web 框架有些不同。 

## 读取请求中的body

 首先，我们来构造最简单的一个请求，POST 一个名字给服务端，服务端应答一个 “Hello xxx”。 

```nginx
location /test {
    content_by_lua_block {
        local data = ngx.req.get_body_data()
        ngx.say("hello ", data)
    }
}
```

访问结果：

```shell
$ curl '127.0.0.1/test'
hello nil
```

我们可以看到data为空，但这是为什么呢？原因是，Nginx 在诞生之初主要是为了解决负载均衡的，而这种情况，是不需要读取 body 就可以决定负载策略的。因此，想要读取到body还需要添加指令 `lua_need_request_body`。

```nginx
lua_need_request_body on;

location /test {
    content_by_lua_block {
        local data = ngx.req.get_body_data()
        ngx.say("hello ", data)
    }
}
```

访问结果：

```shell
$ curl 127.0.0.1/test -d jack
hello jack
```

当然，如果只是某个接口需要读取body，而并非全局的行为，此时可以显示的调用`ngx.req.read_body`函数：

```nginx
location /test {
    content_by_lua_block {
        ngx.req.read_body()
        local data = ngx.req.get_body_data()
        ngx.say("hello ",data)
    }
}
```



## 为什么我读不到body

在调用`ngx.req.get_body_data`函数的时候，会遇到偶尔读不到body而直接返回nil的情况，此时就需要具体分析了：

- 如果事先未读取请求的body，请先调用`ngx.req.read_body`（或配置`lua_need_request_body on`指令，但不推荐此方法）
- 如果请求的body已经被存入临时文件，请使用`ngx.req.get_body_file`函数代替`ngx.req.get_body_data`
- 如果要强制在内存中保存请求的body，请设置`client_body_buffer_size`和`client_max_body`_size为同样的大小。

我们来看下面的示例：

```NGINX
http {
    server {
        listen    80;

        # 强制将请求的body写入到临时文件中（仅为了演示）
        client_body_in_file_only on;

        location /test {
            content_by_lua_block {
                function getFile(file_name)
                    local f = assert(io.open(file_name, 'r'))
                    local string = f:read("*all")
                    f:close()
                    return string
                end

                ngx.req.read_body()
                local data = ngx.req.get_body_data()
                if nil == data then
                    local file_name = ngx.req.get_body_file()
                    ngx.say(">> temp file: ", file_name)
                    if file_name then
                        data = getFile(file_name)
                    end
                end

                ngx.say("hello ", data)
            }
        }
    }
}
```

访问结果：

```shell
$ curl 127.0.0.1/test -d jack
>> temp file: /usr/local/openresty/nginx/client_body_temp/0000000001
hello jack
```

由于 Nginx 是为了解决负载均衡场景诞生的，所以它默认是不读取 body 的行为，而这会对 API Server 和 Web Application 场景造成一些影响。因此，根据需要正确读取、丢弃 body 对 OpenResty 开发是至关重要的 



## 参考

https://moonbingbing.gitbooks.io/openresty-best-practices/content/













