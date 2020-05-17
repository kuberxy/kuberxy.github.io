---
layout: post
title: OpenResty - 简单的API Server框架
date:   2019-12-29 10:02:42 +0800
category: OpenResty
tags: [HTTP]
typora-root-url: ../..
---



## 简单的API Server框架

需求： 实现一个最最简单的数学计算：加、减、乘、除 。

### v1

```nginx
http {
    server {
        listen 80;

        # 加法
        location /addition {
           content_by_lua_block {
                local args = ngx.req.get_uri_args()
                ngx.say(args.a + args.b)
            }
        }

        # 减法
        location /subtraction {
            content_by_lua_block {
                local args = ngx.req.get_uri_args()
                ngx.say(args.a - args.b)
            }
        }

        # 乘法
        location /multiplication {
            content_by_lua_block {
                local args = ngx.req.get_uri_args()
                ngx.say(args.a * args.b)
            }
        }

        # 除法
        location /division {
            content_by_lua_block {
                local args = ngx.req.get_uri_args()
                ngx.say(args.a / args.b)
            }
        }
    }
}
```

我们可以看到，上面的代码很冗长，那么该如何优化呢？

- 首先，需要把这些 location 合并；
- 其次，这些接口的实现放到独立文件中，保持 nginx 配置文件的简洁；



### v2

基于以上两点，优化如下。首先来看nginx.conf

```nginx
http {
    # 在Lua默认的搜索路径中，添加 lua 路径
    # 此处写相对路径时，对启动 nginx 的路径有要求，必须在 nginx 目录下启动
    # 当然require一个绝对路径也没问题，但是不可移植，因此应使用变量 $prefix 或
    # ${prefix}会替换为 nginx 的 prefix path。
    # lua_package_path 'lua/?.lua;/blah/?.lua;;';
    lua_package_path '$prefix/lua/?.lua;/blah/?.lua;;';

    # 这里设置为 off，是为了避免每次修改之后都要重新 reload 的麻烦。
    # 在生产环境上务必确保 lua_code_cache 设置成 on。
    lua_code_cache off;

    server {
        listen 80;

        # 在代码路径中使用nginx变量
        # 注意： nginx var 的变量一定要谨慎，否则将会带来非常大的风险
        location ~ ^/api/([-_a-zA-Z0-9/]+) {
            # 准入阶段完成参数验证
            access_by_lua_file  lua/access_check.lua;

            #内容生成阶段
            content_by_lua_file lua/$1.lua;
        }
    }
}
```

然后，再来看Lua脚本的内容：

```lua
-- ========== {$prefix}/lua/addition.lua
local args = ngx.req.get_uri_args()
ngx.say(args.a + args.b)

-- ========== {$prefix}/lua/subtraction.lua
local args = ngx.req.get_uri_args()
ngx.say(args.a - args.b)

-- ========== {$prefix}/lua/multiplication.lua
local args = ngx.req.get_uri_args()
ngx.say(args.a * args.b)

-- ========== {$prefix}/lua/division.lua
local args = ngx.req.get_uri_args()
ngx.say(args.a / args.b)
```

作为一个API Server，参数验证当然是少不了的。而参数过滤的方法对于这几个API来说是统一的，那么在这几个 API 中如何共享这个方法呢？这时就需要 Lua 模块来完成了。 

- 使用统一的公共模块，完成参数验证；

- 验证入口最好也统一，不要分散在不同地方；

验证模块如下：

```lua
-- ========== {$prefix}/lua/comm/param.lua
local _M = {}

-- 对输入参数逐个进行校验，只要有一个不是数字类型，则返回 false
function _M.is_number(...)
    local arg = {...}

    local num
    for _,v in ipairs(arg) do
        num = tonumber(v)
        if nil == num then
            return false
        end
    end

    return true
end

return _M

-- ========== {$prefix}/lua/access_check.lua
local param= require("comm.param")
local args = ngx.req.get_uri_args()

if not args.a or not args.b or not param.is_number(args.a, args.b) then
    ngx.exit(ngx.HTTP_BAD_REQUEST)
    return
end
```

访问结果：

```shell
$ curl '127.0.0.1:80/api/addition?a=1'
<html>
<head><title>400 Bad Request</title></head>
<body bgcolor="white">
<center><h1>400 Bad Request</h1></center>
<hr><center>openresty/1.9.3.1</center>
</body>
</html>

$ curl '127.0.0.1:80/api/addition?a=1&b=3'
4
```

可以看到，基本上是按照预期执行的。参数不全或错误时，会提示400错误。而正确访问，会返回预期结果。 

最后，我们来看一下，当前整体的目录关系： 

```shell
.
├── conf
│   ├── nginx.conf
├── logs
│   ├── error.log
│   └── nginx.pid
├── lua
│   ├── access_check.lua
│   ├── addition.lua
│   ├── subtraction.lua
│   ├── multiplication.lua
│   ├── division.lua
│   └── comm
│       └── param.lua
└── sbin
    └── nginx
```

如上就是一个简单的API Server，但这绝对不是终极版本。这里面还有很多需要进一步去考虑的地方，但一个最基本的框架已经有了。



## 参考

https://moonbingbing.gitbooks.io/openresty-best-practices/content/













