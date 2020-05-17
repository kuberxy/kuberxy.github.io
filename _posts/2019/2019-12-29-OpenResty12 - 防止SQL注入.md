---
layout: post
title: OpenResty - 防止SQL注入
date:   2019-12-29 10:02:42 +0800
category: OpenResty
tags: [HTTP]
typora-root-url: ../..
---





所谓SQL注入，就是把恶意SQL语句插入到Web表单、域名或页面请求中，达到欺骗服务器并执行恶意SQL语句。具体来说，它是利用现有应用程序，将恶意的SQL语句注入到后台数据库执行的能力，它不是按照设计者的意图去执行SQL语句，而是通过在Web表单中输入恶意SQL语句，来获取一个存在安全漏洞的网站的数据。比如，在早些年前很多影视网站泄露 VIP 会员密码，大多就是通过 Web 表单提交恶意SQL语句而查出来的，这类表单特别容易受到 SQL 注入式攻击。 



## 一个SQL注入的例子

我们来看一个简单的SQL注入示例：

```nginx
location /test {
    content_by_lua_block {
        local mysql = require "resty.mysql"
        local db, err = mysql:new()
        if not db then
            ngx.say("failed to instantiate mysql: ", err)
            return
        end

        db:set_timeout(1000) -- 1 sec

        local ok, err, errno, sqlstate = db:connect{
            host = "127.0.0.1",
            port = 3306,
            database = "ngx_test",
            user = "ngx_test",
            password = "ngx_test",
            max_packet_size = 1024 * 1024 }

        if not ok then
            ngx.say("failed to connect: ", err, ": ", errno, " ", sqlstate)
            return
        end

        ngx.say("connected to mysql.")

        local res, err, errno, sqlstate =
            db:query("drop table if exists cats")
        if not res then
            ngx.say("bad result: ", err, ": ", errno, ": ", sqlstate, ".")
            return
        end

        res, err, errno, sqlstate =
            db:query("create table cats "
                     .. "(id serial primary key, "
                     .. "name varchar(5))")
        if not res then
            ngx.say("bad result: ", err, ": ", errno, ": ", sqlstate, ".")
            return
        end

        ngx.say("table cats created.")

        res, err, errno, sqlstate =
            db:query("insert into cats (name) "
                     .. "values (\'Bob\'),(\'\'),(null)")
        if not res then
            ngx.say("bad result: ", err, ": ", errno, ": ", sqlstate, ".")
            return
        end

        ngx.say(res.affected_rows, " rows inserted into table cats ",
                "(last insert id: ", res.insert_id, ")")

        -- 这里有 SQL 注入（后面的 drop 操作）
        local req_id = [[1'; drop table cats;--]]
        res, err, errno, sqlstate =
            db:query(string.format([[select * from cats where id = '%s']], req_id))
        if not res then
            ngx.say("bad result: ", err, ": ", errno, ": ", sqlstate, ".")
            return
        end

        local cjson = require "cjson"
        ngx.say("result: ", cjson.encode(res))

        -- 再次查询，table 被删
        res, err, errno, sqlstate =
            db:query([[select * from cats where id = 1]])
        if not res then
            ngx.say("bad result: ", err, ": ", errno, ": ", sqlstate, ".")
            return
        end

        db:set_keepalive(10000, 100)
    }
}
```



## 解决SQL注入

解决 SQL 注入，有很多方法，比如，替换各种关键字等。在 OpenResty 中，其实就简单很多了，只需要对输入参数进行一层过滤即可。 

对于 MySQL ，可以调用 `ndk.set_var.set_quote_sql_str` ，进行一次过滤即可。 

```lua
-- for mysql
local req_id = [[1'; drop table cats; --]]
res, err, errno, sqlstate = db:query(string.format(
        [[select * from cats where id = %s]],
        ndk.set_var.set_quote_sql_str(req_id)
    )
)
if not res then
    ngx.say("bad result: ", err, ": ", errno, ": ", sqlstate, ".")
    return
end
```





## 参考

https://moonbingbing.gitbooks.io/openresty-best-practices/content/













