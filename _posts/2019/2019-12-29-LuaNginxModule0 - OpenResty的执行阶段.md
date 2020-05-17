---
layout: post
title: LuaNginxModule - OpenResty的执行阶段
date:   2019-12-29 10:02:42 +0800
category: OpenResty
tags: [HTTP]
typora-root-url: ../..
---



Nginx把一个请求分成了多个阶段，这样第三方模块就能根据自己的功能，加载到不同的阶段，以达到请求处理的目的。同样，OpenResty也继承了nginx的这个特性，然而，所不同的是，OpenResty加载的是我们编写的Lua代码。这样，我们就可以根据自己的需要，在不同的阶段直接对请求进行相应的处理。

## OpenResty的执行阶段

### 请求的处理流程

当一个请求到达时（从request start开始），OpenResty会按如下流程进行处理：

![image-20200223104127966](/assets/images/2019/image-20200223104127966.png)

说明：

- init_by_lua：当Nginx Master进程加载Nginx配置文件时，在全局的Lua VM级别上运行指定的Lua代码。
- init_worker_by_lua：在每个Nginx Worker进程启动时，运行指定的Lua代码。
- ssl_certificate_by_lua：在SSL握手阶段，运行指定的Lua代码。
- set_by_lua：由于该阶段会阻塞Nginx事件，因此只适合处理一些简短的工作。比如，初始化变量等。
- rewrite_by_lua：在请求重写阶段，进行转发、重定向、缓存等工作。比如，特定请求代理到外网。
- access_by_lua：在请求进入阶段，进行准入、权限验证等工作。比如，配合iptable实现一个软件防火墙。
- content_by_lua：生成响应主体的内容。
- header_filter_by_lua：过滤或处理响应头部。比如，添加头参数。
- body_filter_by_lua：过滤或处理响应主体。比如，将响应内容统一成大写。
- log_by_lua：在完成了请求的处理后，异步的记录日志。

实际上，只使用`content_by_lua*`阶段，也可以完成所有的处理。但这样做，会让我们的代码比较臃肿，越到后期越发难以维护。因此，把逻辑放在不同的阶段，明确的分工和独立的代码更利于后期的维护。 



### 应用示例

假设，在最开始的开发中，请求主体和响应主体都是通过 HTTP 明文传输的。后期为了传输体积、安全等要求，我们设计了支持压缩、加密的密文协议(上下行)。此时，难道要更改所有 API 的入口、出口么？ 

如果利用不同的执行阶段，我们可以非常简单的实现： 

```nginx
# 明文协议版本
location /mixed {
    content_by_lua_file ...;       # 请求处理
}

# 加密协议版本
location /mixed {
    access_by_lua_file ...;        # 请求解码
    content_by_lua_file ...;       # 请求处理，不需要关心通信协议
    body_filter_by_lua_file ...;   # 响应加密
}
```

如上，我们在 `access_by_lua*` 完成密文协议解码，`body_filter_by_lua*` 完成响应的加密编码。如此一来，我们没有更改已实现功能的一行代码，只是利用 OpenResty 的阶段处理特性，就解决了这个问题。 



## 参考

https://moonbingbing.gitbooks.io/openresty-best-practices/content/













