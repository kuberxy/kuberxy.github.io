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
- 基于请求的生命周期进行多阶段控制（比如，需要在SSL卸载过程中对握手的连接频率进行控制，但Nginx只能在ACCESS阶段进行控制）
- 跨机器进行状态同步（OpenResty可以连接外部数据库）

## resty.limit.req

### 简介

resty.imit.req使用漏桶算法来限制请求的速率，它的实现类似于Nginx的litmit_req模块。



### demo

```nginx
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



### 内置方法

#### new

实例化一个limit.req对象，即一个请求速率限速器，语法如下：

```nginx
obj, err = class.new(shdict_name, rate, burst)
```

参数说明：

- `shdict_name`：限速器使用的共享字典空间。当存在多个限速器时，最好每个限速器使用不同的共享字典空间。
- `rate`：阈值（请求速率），单位为n/s。
- `burst`：每秒允许有多少个超过阈值的请求被延迟，即允许突发的请求数。

该方法运行失败（如，共享字典空间不存在），则会返回`nil`和错误信息。

```lua
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
```



#### incoming

当一个请求到达时，该方法会触发一个新请求传入事件，并会根据指定的key计算当前请求应该延迟处理或者应该立即拒绝。语法如下：

```nginx
delay, err = obj:incoming(key, commit)
```

参数说明：

- `key`：用户定义的、用来统计速率的指标（key）

- `commit`：是一个布尔值，默认为false。true表示在共享字典空间中记录实际的事件；false表示这只是一次“演习（dry run）”

该方法的返回值有以下几种情况：

- 如果请求没有超过new方法中定义的rate值，则delay的返回值为0，err的返回值为0（当前每秒超过了多少请求）。
- 如果请求超过了new方法中定义的rate值，但没有超过rate+burst，则delay的返回值为该请求应该延时的时间（单位：秒），err的返回值为当前每秒超过了多少请求（注：该值可用于监控未调整的传入请求的速率）。
- 如果请求超过rate+burst，则delay的返回值为nil，err的返回值为字符串rejected。

- 如果该方法发生错误，则delay的返回值为nil，err的返回值为错误信息。

该方法仅返回延迟时间，不会调用sleep方法。对于需要延迟处理的请求，需要自行调用ngx.sleep方法。

```lua
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
```



#### set_rete

重写new方法中的rate参数的值。语法如下：

```lua
obj:set_rate(rate)
```



#### set_burst

重写new方法中的burst参数的值。语法如下：

```lua
obj:set_burst(burst)
```



#### uncommit

取消incoming调用时的提交。该方法主要用于resty.limit.traffic模块中同时组合多个限制器时使用。语法如下：

```lua
ok, err = obj:uncommit(key)
```



## resty.limit.count

### 简介

limit.count 使用固定时间窗口来限制请求的数量。它的实现类似于GitHub 的API Rate Limiting 。 

 

### demo

```nginx
http {
    lua_shared_dict my_limit_count_store 100m;

    init_by_lua_block {
        require "resty.core"
    }

    server {
        location / {
            access_by_lua_block {
                local limit_count = require "resty.limit.count"

                -- rate: 5000 requests per 3600s
                local lim, err = limit_count.new("my_limit_count_store", 5000, 3600)
                if not lim then
                    ngx.log(ngx.ERR, "failed to instantiate a resty.limit.count object: ", err)
                    return ngx.exit(500)
                end

                -- use the Authorization header as the limiting key
                local key = ngx.req.get_headers()["Authorization"] or "public"
                local delay, err = lim:incoming(key, true)

                if not delay then
                    if err == "rejected" then
                        ngx.header["X-RateLimit-Limit"] = "5000"
                        ngx.header["X-RateLimit-Remaining"] = 0
                        return ngx.exit(503)
                    end
                    ngx.log(ngx.ERR, "failed to limit count: ", err)
                    return ngx.exit(500)
                end

                -- the 2nd return value holds the current remaining number
                -- of requests for the specified key.
                local remaining = err

                ngx.header["X-RateLimit-Limit"] = "5000"
                ngx.header["X-RateLimit-Remaining"] = remaining
            }
        }
    }
}
```





### 内置方法

#### new

实例化一个limit.count对象，即一个请求数量限速器，语法如下：

```lua
obj, err = class.new(shdict_name, count, time_window)
```

该方法可接收如下参数：

- `shdict_name`：共享字典空间
- `count`：阈值（请求数）
- `time_window`：时间窗口（单位：秒）

如果该方法运行失败，则会返回`nil`和错误信息（如，a bad `lua_shared_dict` name）。

```lua
local limit_count = require "resty.limit.count"

-- rate: 5000 requests per 3600s
local lim, err = limit_count.new("my_limit_count_store", 5000, 3600)
if not lim then
    ngx.log(ngx.ERR, "failed to instantiate a resty.limit.count object: ", err)
    return ngx.exit(500)
end
```



#### incomming

当一个请求到达时，该方法会触发一个新请求传入事件，并会根据指定的key计算当前请求应该延迟处理或者应该立即拒绝。语法如下：

```lua
delay, err = obj:incoming(key, commit)
```

该方法接收如下参数：

- `key`：用户定义的、用来统计请求数的指标（key）
- `commit`：是一个布尔值，默认为false。

该方法的返回值有以下几种情况：

- 如果请求没有超过new方法中定义的conn值，则delay的返回值为0，err的返回值为当前剩余的、可接受的连接数。
- 如果请求超过new方法中定义的conn值，则delay的返回值为nil，err的返回值为字符串rejected。
- 如果该方法发生错误，则delay的返回值为nil，err的返回值为错误信息。

```lua
-- use the Authorization header as the limiting key
local key = ngx.req.get_headers()["Authorization"] or "public"
local delay, err = lim:incoming(key, true)
if not delay then
    if err == "rejected" then
        ngx.header["X-RateLimit-Limit"] = "5000"
        ngx.header["X-RateLimit-Remaining"] = 0
        return ngx.exit(503)
    end
    ngx.log(ngx.ERR, "failed to limit count: ", err)
    return ngx.exit(500)
end
```



#### uncommit

取消incoming调用时的提交。此方法主要用于将指定的请求排除在限制之外，如条件请求。语法如下：

```lua
remaining, err = obj:uncommit(key)
```



## resty.limit.conn

### 简介

limit.conn是一个用来限制并发连接数/并发请求数的模块，它类似于Nginx的limit_conn模块，但与之不同的是limit.conn支持对超过并发限制的请求/连接进行延迟。



### demo

```nginx
# demonstrate the usage of the resty.limit.conn module (alone!)
http {
    lua_shared_dict my_limit_conn_store 100m;

    server {
        location / {
            access_by_lua_block {
                -- well, we could put the require() and new() calls in our own Lua
                -- modules to save overhead. here we put them below just for
                -- convenience.

                local limit_conn = require "resty.limit.conn"

                -- limit the requests under 200 concurrent requests (normally just
                -- incoming connections unless protocols like SPDY is used) with
                -- a burst of 100 extra concurrent requests, that is, we delay
                -- requests under 300 concurrent connections and above 200
                -- connections, and reject any new requests exceeding 300
                -- connections.
                -- also, we assume a default request time of 0.5 sec, which can be
                -- dynamically adjusted by the leaving() call in log_by_lua below.
                local lim, err = limit_conn.new("my_limit_conn_store", 200, 100, 0.5)
                if not lim then
                    ngx.log(ngx.ERR,
                            "failed to instantiate a resty.limit.conn object: ", err)
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

                if lim:is_committed() then
                    local ctx = ngx.ctx
                    ctx.limit_conn = lim
                    ctx.limit_conn_key = key
                    ctx.limit_conn_delay = delay
                end

                -- the 2nd return value holds the current concurrency level
                -- for the specified key.
                local conn = err

                if delay >= 0.001 then
                    -- the request exceeding the 200 connections ratio but below
                    -- 300 connections, so
                    -- we intentionally delay it here a bit to conform to the
                    -- 200 connection limit.
                    -- ngx.log(ngx.WARN, "delaying")
                    ngx.sleep(delay)
                end
            }

            -- content handler goes here. if it is content_by_lua, then you can
            -- merge the Lua code above in access_by_lua into your
            -- content_by_lua's Lua handler to save a little bit of CPU time.

            log_by_lua_block {
                local ctx = ngx.ctx
                local lim = ctx.limit_conn
                if lim then
                    -- if you are using an upstream module in the content phase,
                    -- then you probably want to use $upstream_response_time
                    -- instead of ($request_time - ctx.limit_conn_delay) below.
                    local latency = tonumber(ngx.var.request_time) - ctx.limit_conn_delay
                    local key = ctx.limit_conn_key
                    assert(key)
                    local conn, err = lim:leaving(key, latency)
                    if not conn then
                        ngx.log(ngx.ERR,
                                "failed to record the connection leaving ",
                                "request: ", err)
                        return
                    end
                end
            }
        }
    }
}
```



### 内置方法

#### new

实例化一个limit.conn对象，即一个并发连接数限速器，语法如下：

```lua
obj, err = class.new(shdict_name, conn, burst, default_conn_delay)
```

该方法可接收如下参数：

- `shdict_name`：共享字典空间的名称。对于不同类型的限制器，最好使用单独的共享字典空间
- `conn`：允许的最大并发连接数。对于超出最大并发连接数，但小于conn+burst的连接，会被延迟处理
- `burst`：允许延迟的连接数。对于超出conn+burst的连接，会被直接拒绝
- `default_conn_delay`：连接默认的延迟时间（单位为秒）。该值可在log_by_lua中通过leave方法动态调整

如果该方法运行失败，则会返回`nil`和错误信息（如，a bad `lua_shared_dict` name）。



#### incomming

当一个请求到达时会触发一个传入事件，并会根据指定的key计算当前请求应该延迟处理或者应该立即拒绝。语法如下：

```lua
delay, err = obj:incoming(key, commit)
```

该方法可接收如下参数：

- `key`：用户定义的、用来统计并发的指标（key）。例如，可以使用host name（或server zone）作为key，来限制每个host name的并发；还可以使用client address作为key，避免单个客户端使用并发请求淹没我们的服务。

  注：该模块不会为key添加前缀或后缀。因此，我们需要自行保证key在共享字典空间中是唯一的。

- `commit`：是一个布尔值，默认为false。

该方法的返回值有如下几种情况：

- 如果请求没有超过new方法中定义的conn值，则delay的返回值为0，err的返回值为当前的并发数。
- 如果请求过new方法中定义的conn值，但没有超过conn+burst，则delay的返回值为该请求应该延时的时间，err的返回值为当前的并发数（包括当前请求）。
- 如果请求超过conn+burst，则delay的返回值为nil，err的返回值为字符串rejected。
- 如果该方法发生错误，则delay的返回值为nil，err的返回值为错误信息。

该方法仅返回延迟时间，不会调用sleep方法。对于需要延迟处理的请求，需要自行调用ngx.sleep方法。

此外，在该方法将事件实际记录到共享字典空间时，该方法通常会和log_by_lua上下文中的leaving方法成对调用。



#### is_committed

当incoming方法中的commit参数为true时，该方法会进行实际的提交操作，将请求事件存储到共享字典空间。语法如下：

```lua
bool = obj:is_committed()
```

注：该方法非常重要。因为只有`is_committed`返回true，在调用incoming方法时，才会调用leaving方法



#### leaving

该方法用于清理已经完成的连接。当请求完成后会触发一个完成事件，以降低当前的并发数量（实际上会执行一个连接数减一的操作）。语法如下：

```lua
conn = obj:leaving(key, req_latency?)
```

注：该方法通常和incoming方法成对使用。在调用incoming之后会接着调用该方法（is_committed返回false时除外）。如果在 log 阶段没有做这个清理的动作，那么连接数就会一直上涨，很快就会达到并发的阈值。

该方法可接收如下参数：

- `key`：与incoming方法中的key相同
- `req_latency`：当前连接实际延迟的时间，该参数是可选的。通常会使用nginx内置变量`$request_time`或`$upstream_response_time`的值。可以用于记录每个连接的延迟时间。

该方法会返回新的并发请求数（或活跃的连接数）。与incoming不同是，该方法总是会提交到共享字典空间。



#### set_conn

重写new方法中的conn参数的值。语法如下：

```lua
obj:set_conn(conn)
```



#### set_burst

重写new方法中的burst参数的值。语法如下：

```lua
obj:set_burst(burst)
```



#### umcommit

取消incoming调用时的提交。该方法主要用于在Lua的resty.limit.traffic模块中同时组合多个限制器时使用。语法如下：

```lua
ok, err = obj:uncommit(key)
```

注：尽管该方法的效果和实现与leaving方法相似，但其不能代替leaving方法



## resty.limit.traffic

### 简介

limit.traffic用于将limit.rate、limit.conn 和 limit.count 组合起来使用。也就是，将多个限速器组合成一个限速器。



### demo

```nginx
# demonstrate the usage of the resty.limit.traffic module
http {
    lua_shared_dict my_req_store 100m;
    lua_shared_dict my_conn_store 100m;

    server {
        location / {
            access_by_lua_block {
                local limit_conn = require "resty.limit.conn"
                local limit_req = require "resty.limit.req"
                local limit_traffic = require "resty.limit.traffic"

                local lim1, err = limit_req.new("my_req_store", 300, 200)
                assert(lim1, err)
                local lim2, err = limit_req.new("my_req_store", 200, 100)
                assert(lim2, err)
                local lim3, err = limit_conn.new("my_conn_store", 1000, 1000, 0.5)
                assert(lim3, err)

                local limiters = {lim1, lim2, lim3}

                local host = ngx.var.host
                local client = ngx.var.binary_remote_addr
                local keys = {host, client, client}

                local states = {}

                local delay, err = limit_traffic.combine(limiters, keys, states)
                if not delay then
                    if err == "rejected" then
                        return ngx.exit(503)
                    end
                    ngx.log(ngx.ERR, "failed to limit traffic: ", err)
                    return ngx.exit(500)
                end

                if lim3:is_committed() then
                    local ctx = ngx.ctx
                    ctx.limit_conn = lim3
                    ctx.limit_conn_key = keys[3]
                end

                print("sleeping ", delay, " sec, states: ",
                      table.concat(states, ", "))

                if delay >= 0.001 then
                    ngx.sleep(delay)
                end
            }

            # content handler goes here. if it is content_by_lua, then you can
            # merge the Lua code above in access_by_lua into your
            # content_by_lua's Lua handler to save a little bit of CPU time.

            log_by_lua_block {
                local ctx = ngx.ctx
                local lim = ctx.limit_conn
                if lim then
                    -- if you are using an upstream module in the content phase,
                    -- then you probably want to use $upstream_response_time
                    -- instead of $request_time below.
                    local latency = tonumber(ngx.var.request_time)
                    local key = ctx.limit_conn_key
                    assert(key)
                    local conn, err = lim:leaving(key, latency)
                    if not conn then
                        ngx.log(ngx.ERR,
                                "failed to record the connection leaving ",
                                "request: ", err)
                        return
                    end
                end
            }
        }
    }
}
```



### combine方法

将所有的限制器和为其指定的限制key结合起来，并计算所有限制器的总延迟。同时，还支持记录每个具体的限制器返回的当前状态信息。语法如下：

```lua
delay, err = class.combine(limiters, keys)
delay, err = class.combine(limiters, keys, states)
```

该方法可接受如下参数：

- `limiters`：是一个Lua表，用于保存所有限制器对象（如：resty.limit.req、resty.limit.conn、resty.limit.count或其他兼容模块实例化的对象）。
- `keys`：是一个Lua表，用于保存与每个限制器相对应的限制key。元素的个数应该与limiters相等。
- `states`：一个Lua表（可选参数），用于输出每个限制器返回的所有状态信息。

该方法返回以秒为单位的延迟，该延迟是所有限制器返回的延迟时间之和。

如果存在一个限制器拒绝了当前的请求，则该方法会确保当前传入的请求事件不会在这些限制器中提交。此时，delay会返回nil、err会返回字符串“rejected”。

对于其他错误，它返回nil和错误信息。

该方法不会执行sleep操作，需要自行调用ngx.sleep方法。



## resty.limit.rate

### 简介

resty.limit.rate是又拍云开源的一个限速模块，它使用令牌桶算法来限制请求的速率。



### demo

```lua
http {
    lua_shared_dict my_limit_rate_store 100m;
    lua_shared_dict my_locks 100k;

    server {
        location / {
            access_by_lua_block {
                local limit_rate = require "resty.limit.rate"

                local lim, err = limit_rate.new("my_limit_rate_store", 500, 10, 3, 200, {
                    lock_enable = true, -- use lua-resty-lock
                    locks_shdict_name = "my_locks",
                })

                if not lim then
                    ngx.log(ngx.ERR,
                            "failed to instantiate a resty.limit.rate object: ", err)
                    return ngx.exit(500)
                end

                -- the following call must be per-request.
                -- here we use the remote (IP) address as the limiting key
                local key = ngx.var.binary_remote_addr
                local delay, err = lim:incoming(key, true)
                -- local delay, err = lim:take(key, 1, ture)
                if not delay then
                    if err == "rejected" then
                        return ngx.exit(503)
                    end
                    ngx.log(ngx.ERR, "failed to take token: ", err)
                    return ngx.exit(500)
                end

                if delay >= 0.001 then
                    -- the 2nd return value holds the current avail tokens number
                    -- of requests for the specified key
                    local avail = err

                    ngx.sleep(delay)
                end
            }

            # content handler goes here. if it is content_by_lua, then you can
            # merge the Lua code above in access_by_lua into your content_by_lua's
            # Lua handler to save a little bit of CPU time.
        }

        location /take_available {
            access_by_lua_block {
                local limit_rate = require "resty.limit.rate"

                -- global 20r/s 6000r/5m
                local lim_global = limit_rate.new("my_limit_rate_store", 100, 6000, 2, nil, {
                    lock_enable = true,
                    locks_shdict_name = "my_locks",
                })

                if not lim_global then
                    return ngx.exit(500)
                end

                -- single 2r/s 600r/5m
                local lim_single = limit_rate.new("my_limit_rate_store", 500, 600, 1, nil, {
                    locks_shdict_name = "my_locks",
                })

                if not lim_single then
                    return ngx.exit(500)
                end

                local t0, err = lim_global:take_available("__global__", 1)
                if not t0 then
                    ngx.log(ngx.ERR, "failed to take global: ", err)
                    return ngx.exit(500)
                end

                -- here we use the userid as the limiting key
                local key = ngx.var.arg_userid or "__single__"

                local t1, err = lim_single:take_available(key, 1)
                if not t1 then
                    ngx.log(ngx.ERR, "failed to take single: ", err)
                    return ngx.exit(500)
                end

                if t0 == 1 then
                    return -- global bucket is not hungry
                else
                    if t1 == 1 then
                        return -- single bucket is not hungry
                    else
                        return ngx.exit(503)
                    end
                end
            }
        }
    }
}
```



### 方法

#### new

实例化一个limit.rate对象，即一个速率限速器，语法如下：

```lua
obj, err = class.new(shdict_name, interval, capacity, quantum?, max_wai?, opts?)
```

正常调用该方法会返回一个令牌桶，这个桶会按照特定的时间间隔（interval）每次生成若干个（quantum）令牌，但它的容量是有上限的（capacity）。此外，最开始时这个令牌桶是满的。

参数说明：

- `shdict_name`：共享字典空间的名称。
- `interval`：生成令牌的时间间隔（单位：毫秒）。
- `capacity`：令牌桶可容纳的最大令牌数。
- `quantum`：每次生成的令牌数，默认为1个。
- `max_wait`：当令牌桶为空时，需要等待令牌生成的最大时间（单位：毫秒）。默认为nil，表示具体等待时间需要根据实际负载来计算。

可选参数`opt`是一个表，它的元素可以是：

- `lock_enable`：启用后，会跨多个nginx worker进程原子地更新共享字典空间的状态，否则将在“先读后写”操作之间有一个小的竞争条件窗口。默认为false。
- `locks_shdict_name`：要锁定的共享字典空间的名称（通过lua_shard_dict创建）。默认为locks。



#### incoming

当一个请求到达时，该方法会触发一个请求传入事件，并会根据指定的key计算当前请求应该延迟处理还是应该立即拒绝。语法如下：

```lua
delay, err = obj:incoming(key, commit)
```

参数说明：

- `key`：用来统计速率的指标（key）
- `commit`：是一个布尔值，默认为false。

该方法的返回值有如下几种情况：

- 请求速度超过阈值，同时通过`new`方法或`set_max_wait`方法定义了`max_wait`，当获取令牌所需要等待的时间没有超过`max_wait`时，返回需要等待的时间；而超过`max_wait`时，`delay`为nil，`err`为字符串`rejected`。
- 请求速度超过阈值，同时`max_wait`为`nil`，则直接返回等待时间（ it returns the time that the caller should wait until the tokens are actually available.）。
- 请求速度没有超过阈值，`err`为当前可用的令牌数。

该方法仅返回延迟时间，不会调用sleep方法。对于需要延迟处理的请求，需要自行调用ngx.sleep方法。



#### take

该方法的作用与`incoming`方法类似，所不同是该方法用于从令牌桶中拿取指定的令牌数，语法如下：

```lua
delay, err = obj:take(key, count, commit)
```

参数说明：

- `key`：用来统计速率的指标（key）
- `count`：每次拿取令牌的数量
- `commit`：是一个布尔值，默认为false



#### take_available

该方法用于统计令牌桶中是否有充足的令牌。当令牌充足时，它会返回每次调用所删除的令牌数，而当令牌不足时，它的返回值为0。此外，该方法是非阻塞的。

```lua
count, err = obj:take_available(key, count)
```

参数说明：

- `key`：用来统计速率的指标（key）
- `count`：要删除的令牌数



#### set_max_wait

重写new方法中的`max_wait`参数的值。语法如下：

```lua
obj:set_max_wait(max_wait?)
```



#### uncommit

取消incoming调用时的提交。语法如下：

```lua
ok, err = obj:uncommit(key)
```

This is simply an approximation and should be used with care. This method is mainly for being used in the [resty.limit.traffic](https://github.com/openresty/lua-resty-limit-traffic/blob/master/lib/resty/limit/traffic.md) Lua module when combining multiple limiters at the same time. 



### 安装

```shell
sudo luarocks install lua-resty-limit-rate
```



### 实际应用

```lua
local lim_global = limit_rate.new('my_limit_rate_store', 100, 6000, 2) -- global 20r/s 6000r/5m
local lim_single = limit_rate.new('my_limit_rate_store', 500, 600, 1) -- single 2r/s 600r/5m

local t0, err = lim_global:take_available("__global__", 1)
local t1, err = lim_single:take_available(ngx.var.arg_userid, 1)

if t0 == 1 then
    return -- global bucket is not hungry
else
    if t1 == 1 then
        return -- single bucket is not hungry
    else
        return ngx.exit(503)
    end
end
```

该示例申请两个令牌桶，一个是全局的令牌桶，一个是针对某个用户的令牌桶。

我们知道，肯定会有很多用户向系统发起请求，而全局是一个桶，每个用户是一个桶，这样就可以做一个组合设置。如果全局的桶没有满，单个用户超过了其单独的频次限制，我们一般会允许其突发。因为后端对于处理A用户、B用户的消耗一般是相同的，它们只是在业务逻辑上被分成了A用户和B用户而已。

这样，整体容量没有超过限制，单个用户即便超过了它的限制配置，也允许它突发。只有在全局桶拿不出令牌时，才会判断每个用户的桶，如果它拿不出令牌就将其拒绝。此时，整体系统已经达到了瓶颈，而为了用户体验，我们就不能无差别地弹掉用户的请求，而是挑出当前突发比较大的用户并将其拒绝，从而保障其他正常的用户请求不受任何影响。



除了请求速率限制（即一个令牌一个请求），它还能对字节传输进行流量整形。此时，一个令牌相当于一个字节。因为，流量都是有一个个字节组成的。如果把字节变成令牌，那流量的流出流入也可以通过令牌桶来给流量做一些整形。 整形就是流量按期望设计的形状带宽（单位时间内的流量）进行传输。 

```lua
body = function()
    local siz = 8192
    if size > request_size then
        size = request_size
    end

    local count, err = token_bucket:take_available("service_id", size)
    if count == 0 then
        return nil, err
    end

    size = count

    request_size = request_size - size

    local bytes, err = req_socket:receive(size)
    if err then
        return nil, err
    end

    return bytes
end
```



## 参考

https://www.upyun.com/opentalk/417.html

https://github.com/openresty/lua-resty-limit-traffic

https://github.com/upyun/lua-resty-limit-rate
