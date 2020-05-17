---
layout: post
title: OpenResty - 向外部发起HTTP请求
date:   2019-12-29 10:02:42 +0800
category: OpenResty
tags: [HTTP]
typora-root-url: ../..
---



OpenResty 最主要的应用场景之一是 API Server，有别于传统的 Nginx 代理转发应用场景，在API Server 内部有各种复杂的交易流程和判断逻辑，而调用其他 HTTP Server 是不可或缺的。 

需求： 模拟一个接口场景，一个公共服务专门用来对外提供加“盐” md5 计算，业务系统调用这个公共服务完成业务逻辑，用来判断请求本身是否合法。 

## proxy_pass

```nginx
http {
    upstream md5_server{
        server 127.0.0.1:81;        # ①
        keepalive 20;               # ②
    }

    server {
        listen    80;

        location /test {
            content_by_lua_block {
                ngx.req.read_body()
                local args, err = ngx.req.get_uri_args()

                -- ③
                local res = ngx.location.capture('/spe_md5',
                    {
                        method = ngx.HTTP_POST,
                        body = args.data
                    }
                )

                if 200 ~= res.status then
                    ngx.exit(res.status)
                end

                if args.key == res.body then
                    ngx.say("valid request")
                else
                    ngx.say("invalid request")
                end
            }
        }

        location /spe_md5 {
            proxy_pass http://md5_server;   -- ④
            #For HTTP, the proxy_http_version directive should be set to “1.1” and the “Connection” 
            #header field should be cleared.（from:http://nginx.org/en/docs/http/ngx_http_upstream_module.html#keepalive)
            proxy_http_version 1.1;
            proxy_set_header Connection "";
        }
    }

    server {
        listen    81;           -- ⑤

        location /spe_md5 {
            content_by_lua_block {
                ngx.req.read_body()
                local data = ngx.req.get_body_data()
                ngx.print(ngx.md5(data .. "*&^%$#$^&kjtrKUYG"))
            }
        }
    }
}
```

说明：

 ① 上游访问地址清单，即HTTP Server的访问地址信息(可以按需配置不同的权重规则)；  

 ② 访问上游时，使用长连接。是否开启长连接，对整体性能影响比较大； 

 ③ 对HTTP Server的访问，通过 `ngx.location.capture` 发起子请求； 

 ④ 由于 `ngx.location.capture` 只能对 nginx 实例自身发起请求，因此，需要借助 proxy_pass 发出 HTTP 连接信号； 

 ⑤ 公共 API 服务（在生产环境中，API服务一般运行在外部服务器上，因此不会有这个server配置）； 

可以看到， 为了向外部发起一个HTTP 请求， 我们需要绕好几个弯子，甚至还有可能踩到坑（upstream 中长连接的细节处理），那有没有优雅的方式呢？



## cosocket

我们直接来看示例：

```nginx
http {
    server {
        listen    80;

        location /test {
            content_by_lua_block {
                ngx.req.read_body()
                local args, err = ngx.req.get_uri_args()

                local http = require "resty.http"   -- ①
                local httpc = http.new()
                local res, err = httpc:request_uri( -- ②
                    "http://127.0.0.1:81/spe_md5",
                        {
                        method = "POST",
                        body = args.data,
                      }
                )

                if 200 ~= res.status then
                    ngx.exit(res.status)
                end

                if args.key == res.body then
                    ngx.say("valid request")
                else
                    ngx.say("invalid request")
                end
            }
        }
    }

    server {
        listen    81;

        location /spe_md5 {
            content_by_lua_block {
                ngx.req.read_body()
                local data = ngx.req.get_body_data()
                ngx.print(ngx.md5(data .. "*&^%$#$^&kjtrKUYG"))
            }
        }
    }
}
```

说明：

 ① 引用 `resty.http` 库资源，它来自 [github](https://github.com/pintsized/lua-resty-http) 。 

② 参考 `resty-http` 官方 wiki 说明，我们可以知道 request_uri 函数完成了连接池、HTTP 请求等一系列动作。

我们可以看到，cosocket的方式比proxy_pass的方式要简单很多。当然，如果你的内部请求比较少，使用 `ngx.location.capture`+`proxy_pass` 的方式也没什么问题。但如果你的请求数量比较多，或者需要频繁的修改上游地址，那么 `resty.http`就更适合你了。 

此外， `ngx.thread.*` 与 `resty.http` 相互结合也是很不错的玩法。

 

## 参考

https://moonbingbing.gitbooks.io/openresty-best-practices/content/













