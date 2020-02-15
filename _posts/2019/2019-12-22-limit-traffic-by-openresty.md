---

layout: post
title: 使用OpenResty实现动态限流
date:   2019-12-22 10:02:42 +0800
category: 容量规划
tags: [容量规划]
typora-root-url: ../..
---

相比Nginx，OpenResty具有更多的优势：

- 丰富的流控策略（Nginx只有经典的几种）
- 灵活的配置管理（OpenResty可以实现多样化的配置规则）
- 基于请求的生命周期进行多阶段控制（比如，需要在SSL卸载过程中对握手的连接频率进行控制，但Nginx只能在PREACCESS阶段进行控制）
- 跨机器进行状态同步（OpenResty可以连接外部数据库）

## resty.limit.req

### 简介

resty.imit.req使用漏桶算法来限制请求的速率，它的实现类似于Nginx的litmit_req模块。



### demo

```lua
# demonstrate the usage of the resty.limit.req module (alone!)
http {
    lua_shared_dict my_limit_req_store 100m;

    server {
        location / {
            access_by_lua_block {
                -- well, we could put the require() and new() calls in our own Lua
                -- modules to save overhead. here we put them below just for
                -- convenience.

                local limit_req = require "resty.limit.req"

                -- limit the requests under 200 req/sec with a burst of 100 req/sec,
                -- that is, we delay requests under 300 req/sec and above 200
                -- req/sec, and reject any requests exceeding 300 req/sec.
                local lim, err = limit_req.new("my_limit_req_store", 200, 100)
                if not lim then
                    ngx.log(ngx.ERR,
                            "failed to instantiate a resty.limit.req object: ", err)
                    return ngx.exit(500)
                end

                -- the following call must be per-request.
                -- here we use the remote (IP) address as the limiting key
                local key = ngx.var.binary_remote_addr
                local delay, err = lim:incoming(key, true)
                if not delay then
                    if err == "rejected" then
                        return ngx.exit(503)
                    end
                    ngx.log(ngx.ERR, "failed to limit req: ", err)
                    return ngx.exit(500)
                end

                if delay >= 0.001 then
                    -- the 2nd return value holds  the number of excess requests
                    -- per second for the specified key. for example, number 31
                    -- means the current request rate is at 231 req/sec for the
                    -- specified key.
                    local excess = err

                    -- the request exceeding the 200 req/sec but below 300 req/sec,
                    -- so we intentionally delay it here a bit to conform to the
                    -- 200 req/sec rate.
                    ngx.sleep(delay)
                end
            }

            # content handler goes here. if it is content_by_lua, then you can
            # merge the Lua code above in access_by_lua into your content_by_lua's
            # Lua handler to save a little bit of CPU time.
        }
    }
}
```



### 方法

#### new

实例化一个limit.req对象，即一个限速器，语法如下：

```nginx
obj, err = class.new(shdict_name, rate, burst)
```

参数说明：

- `shdict_name`：限速器使用的共享字典空间。当存在多个限速器时，最好每个限速器使用不同的共享字典空间。
- `rate`：阈值（请求速率）
- `burst`：每秒允许有多少个超过阈值的请求被延迟，即允许突发的请求数。

该方法运行失败（如，共享字典空间不存在），则会返回`nil`和错误信息。



我们下面这个示例：

```lua
local limit_req = require "resty.limit.req"
local lim, err = limit_req.new("my_limit_req_store", 200, 100)
```



#### incoming

当一个请求到达时，该方法会触发一个新请求传入事件，并会根据指定的key计算当前请求应该延迟处理或者应该立即拒绝。语法如下：

```nginx
delay, err = obj:incoming(key, commit)
```

参数说明：

- `key`：用户定义的、用来统计速率的指标（key）

- `commit`：是一个布尔值，默认为false。

该方法的返回值有以下几种情况：





#### set_rete

#### set_burst

#### uncommit



## resty.limit.count

### 简介

limit.count 使用固定时间窗口来限制请求数。它的实现类似于GitHub 的API Rate Limiting 。 

 

### demo



## resty.limit.conn

### 简介

limit.conn是一个用来限制并发连接数/并发请求书的模块，它类似于Nginx的limit_conn模块，但与之不同的是limit.conn支持对超过并发限制的请求/连接进行延迟。



### demo



## 参考

https://www.upyun.com/opentalk/417.html




