---

layout: post
title: 使用Nginx实现限流
date:   2019-12-22 10:02:42 +0800
category: 容量规划
tags: [容量规划]
typora-root-url: ../..
---

如果我们要做流量管理，应该尽量往前做，而不要等流量转发到后端。因为让应用服务去做可能已经来不及了，应用服务只需关系业务。因此，这些事情就应该让前端的代理服务器来完成。



## 请求速率限制 - limit_req模块

>  The `ngx_http_limit_req_module` module (0.7.21) is used to limit the request processing rate per a defined key, in particular, the processing rate of requests coming from a single IP address. The limitation is done using the “leaky bucket” method. 

该模块用于限制请求的处理速率。在使用该模块时需要定义一个key，limit_req会基于这个key来统计它的速率是否超过一个设定的阈值。



该模块包含如下指令：

- limit_req_zone
- limit_req
- limit_req_status
- limit_req_log_level
- limit_req_dry_run

下面我们一一来了解这些模块的作用



### limit_req_zone

#### 语法

```nginx
Syntax:	limit_req_zone key zone=name:size rate=rate [sync];
Default:	—
Context:	http
```



#### 作用

>  Sets parameters for a shared memory zone that will keep states for various keys. In particular, the state stores the current number of excessive requests. The `*key*` can contain text, variables, and their combination. Requests with an empty key value are not accounted. 

定义一个共享内存空间，这个空间会维护大量key的状态（可理解为一个key-value）。其中，key可以是文本、变量或两者的组合。如果一个请求不包含这个key，那么将不会统计这个请求。



#### 示例

```
imit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;
```

这里我们定义一个名为one的共享空间，它的大小为10M，并规定对于同一个客户端，请求速率不能超过每秒一个请求。

客户端IP这里之所以使用$binary_remote_addr，而不是用$remote_addr，是因为$remote_addr占用的空间大小为7～15B，而$binary_remote_addr占用的空间大小始终是4B（IPv4地址）或16B（IPv6地址）；另外，存储一个value占用的空间大小始终是64B（32位平台）或128B（64位平台），因此，1M的共享空间可以保存大约16000个64B的value或8000个128B的value。

当共享空间不足时，会删除最近最少使用的key-value。如果还是无法为一个新请求创建key-value，则直接返回错误码。

rate通常指定为每秒的请求数，即r/s。如果想要指定小于每秒一个请求的速率，则可以指定为每分钟的请求数，比如，30r/m表示每秒半个请求。



### limit_req

#### 语法

```nginx
Syntax:	limit_req zone=name [burst=number] [nodelay | delay=number];
Default:	—
Context:	http, server, location
```



#### 作用

>  Sets the shared memory zone and the maximum burst size of requests. If the requests rate exceeds the rate configured for a zone, their processing is delayed such that requests are processed at a defined rate. Excessive requests are delayed until their number exceeds the maximum burst size in which case the request is terminated with an [error](http://nginx.org/en/docs/http/ngx_http_limit_req_module.html#limit_req_status). By default, the maximum burst size is equal to zero.  

设置限速模块所使用的共享内存空间，此外还可以通过指定burst参数允许一定数量的突发请求（也就是漏桶的容量）。

请求的处理速度不会超过阈值，对于超过阈值的请求，则会被延迟处理。如果被延迟的请求的数量超过了burst，那么将会拒绝超出的那个部分请求。

burst默认为0，即对于超过阈值的请求直接拒绝。



#### 示例

```nginx
limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;

server {
    location /search/ {
        limit_req zone=one burst=5;
    }
```

这里我们首先通过limit_req_zone指令，定义了一个大小为10M、名为one的共享内存空间，同时设定请求速度的阈值为每秒一个请求；然后在通过limit_req指令使用了one这个共享内存空间，并设置burst为5；因此，最后的配置效果为：请求速度=1r/s，漏桶的容量=5，即请求进入的速度限制为每秒一个请求，对于超出的请求，允许5个被延迟处理，其他超出的请求则被拒绝。



如果不想有过多的请求被延迟，则可以指定nodelay参数。nodelay配合burst一起使用就能使原本需要延迟的请求立即得到处理。如下配置：

```nginx
limit_req zone=one burst=5 nodelay;
```



delay参数用于指定一个请求被延迟处理的界限值。对于超速的请求，小于该界限值的请求被直接处理，大于该界限值的请求则被延迟。



可以同时配置多个limit_req指令，如下：

```nginx
limit_req_zone $binary_remote_addr zone=perip:10m rate=1r/s;
limit_req_zone $server_name zone=perserver:10m rate=10r/s;

server {
    ...
    limit_req zone=perip burst=5 nodelay;
    limit_req zone=perserver burst=10;
}
```

这里我们，限制客户端请求速率为1r/s，服务端的处理速率为10r/s。

注：limit_req指令具有继承性。如果当前级别没有设置limit_req指令，那么会从上级继承。



### limit_req_status

#### 语法

```nginx
Syntax:	limit_req_status code;
Default:	limit_req_status 503;
Context:	http, server, location
This directive appeared in version 1.3.15.
```



#### 作用

>  Sets the status code to return in response to rejected requests. 

设置被拒绝请求的响应状态码



### limit_req_log_level

#### 语法

```nginx
Syntax: limit_req_log_level info | notice | warn | error;
Default:	limit_req_log_level error;
Context:	http, server, location
This directive appeared in version 0.8.18.
```



#### 作用

> Sets the desired logging level for cases when the server refuses to process requests due to rate exceeding, or delays request processing. Logging level for delays is one point less than for refusals; for example, if “`limit_req_log_level notice`” is specified, delays are logged with the `info` level.

被延迟和被拒绝的请求会记录在日志中。该指令用于设置被拒绝的请求在日志中的级别，同时也会设置被延迟的请求在日志中的级别。

被延迟请求比被决绝请求低一个级别，比如，当limit_req_log_level为notice时，表示被拒绝的请求为notice级别，被延迟的请求为info级别。



### limit_req_dry_run

#### 语法

```nginx
Syntax:	limit_req_dry_run on | off;
Default:	
limit_req_dry_run off;
Context:	http, server, location
This directive appeared in version 1.17.1.
```



#### 作用

> Enables the dry run mode. In this mode, requests processing rate is not limited, however, in the shared memory zone, the number of excessive requests is accounted as usual.

启用演习模式。在该模式下，不会对请求进行限速，但是，仍会共享内存空间中统计请求的速率。



### 示例

#### 一个简单的配置

```nginx
http {
    limit_req_zone $binary_remote_addr zone=mylimit:10m rate=5r/s;

    limit_req_log_level error;
    limit_req_status 429;

    server {
        location /search/ {
            limit_req zone=mylimit;

            proxy_pass http://website;
        }
    }
}
```

我们可以看到，该配置基于来源IP作为唯一的Key，即针对客户端的IP做速率统计，而这里的速率配置为5r/s，也就是1秒内允许进来5个请求。

基于limit_req模块的实现，5r/s的实质是每隔200ms能够进来一个请求，即对于两个进来的请求，它们之间的时间间隔应该是大于等于200ms，如果一个新请求与前一个进来的请求之间的时间间隔小于200ms就直接拒绝这个请求。因此，我们可以得到一个这样的效果图：

![image-20200211170121793](/assets/images/2019/image-20200211170121793.png)

图中，整个灰色的条带是一个跨度为1s的时间轴，它被等分成5个时间片，也就是每个时间片的跨度为200ms。时间轴上方的每个箭头都代表一个进入的请求，允许进入的请求会越过时间轴并变成绿色，不允许进入的请求会被拦截在时间轴的上方并变成红色。

我们可以看到，第一请求在0s时到达，因为它是第一个时间片上的第一个请求，因此直接放行；第二个请求大约在0.1s时到达，因为与第一个请求之间的间隔不足200ms，因此，直接被拒绝；第三个请求在0.2s时到达，因为与第一个请求之间的间隔等于200ms，因此，直接允许······以此类推，即可得出每个请求的处理情况。



#### burst参数

在实际的业务中，偶尔的突增现象也是十分正常的。显然，上图图解中处理请求的方法过于敏感，当用户请求过于频繁（间隔小于200ms），就有很明显的间隔错误问题。这时，就需要使用burst参数了。我们将上面的配置修改一下：

```nginx
limit_req zone=mylimit burst=4;
```

这里设置burst=4，表示在超过速率限制时，原本要直接拒绝的请求，现在系统允许额外有4个请求排队等候，待速率回归正常后，队列中的请求会按顺序被处理。这样，提前到达的部分请求，会在Nginx这一侧进行排队，等到请求可以进来时就会从队列中放行一个请求；而对于后端的应用服务器来说，它所感知到的始终是每隔200ms进来一个请求。

![image-20200212101956483](/assets/images/2019/image-20200212101956483.png)

我们可以看到：

- 第一个请求、第二个请求、第三个请求分别在0s、0.2s、0.4s时达到，因为它们与前一个进入的请求之间的间隔都是大于等于200ms，所以，会直接将它们放行；
- 第四个请求在大约0.5s时达到，因为与前一个进入的请求之间的间隔小于200ms，因此它被放到队列中了。在时间流转到0.6s时，它才被转发给后端。可以看出第四个请求实际上等待了100ms。
- 第五个请求在0.6s时达到，由于此时它所在的时间片已经被第四个请求占用，因此第五个请求也会被放到队列中，等到下一个时间片它才能被转发，也就是说它需要等待200ms。
- 第六、第七、第八个请求达到时，由于它们所处的第4个时间片已经被使用，并且此时排队的请求数为1，因此它们都会被放到队列中。此时，我们可以看出，在第4个时间片时，有一个请求被放行，有四个请求被排队。
- 第九个请求在0.8s时达到，由于此时队列中已经有四个请求在排队，而当前正好是第5个时间片的开始，因此，此时会放行队列中的第一个请求，并将第九个放到队列的末尾。此外，我们可以看出第就个请求需要等待800ms。
- 第十、第十一、第十二个请求到达时，由于它们所处的第5个时间片已经被使用，并且此时排队的请求数为4，因此它们都会被直接拒绝。

此外，从上图我们可以看到，不论前端请求有多么频繁，后端服务的接收间隔永远是恒定的200ms。

> 我们需要注意的是，burst虽然允许了一定程度的突发，但在有些业务场景中，因排队导致的处理延迟是不可接受的（上图中第九个请求需要延迟800ms）。

对于一些敏感的业务，我们不允许排队太久，因为这些延时根本就不是在进行有效处理，它只是等候在 Nginx 这侧，这时很多业务场景可能就接受不了。因此，这样的机制需要我们结合新的要求再优化。但是如果你对延时没有要求，允许一定的突发，用起来就比较舒服了。 



#### nodelay参数

burst参数虽然实现了允许一定程度的突发，但存在很延迟问题。这时，就需要使用nodelay参数了。我们将上面的配置再修改一下：

```nginx
limit_req zone=mylimit burst=4 nodelay;
```

这里在设置burst参数时，同时开启了nodelay。这样就使得在突发时，原本需要等待的请求立即得到了处理。

![image-20200212144315295](/assets/images/2019/image-20200212144315295.png)

如上图，在这种场景下，我们可以认为，当请求速率等于阈值时，会出现一个有着四个槽位的桶。桶中的每个槽位可以存放一个token。此外，这个桶每隔200ms会释放出一个槽位。

在请求速率等于阈值后，新达到的请求会尝试将一个token放入桶中，如果放入成功就放行，如果放入失败就拒绝。因此，我们可以看到：

- 在第四个请求到达时，由于此时的请求速率已经等于阈值了，因此它在桶中放入token后，直接被放行了。
- 在第五个请求到达时，由于此时的请求速率大于阈值，因此它会在桶中放入token。又因此此时正好处在一个槽位释放间隔上，因此此时桶中仍可以放入三个token。
- 在第六、第七、第八个请求达到时，它们都会向桶中放入一个token，因此此时桶会被填满。
- 在第九个请求达到时，由于此时正好处在一个槽位释放间隔上，此时会空出一个槽位，所以第九个请求会直接被放行。
- 在第十、第十一、第十二个请求到达时，桶均已被填满，因此会直接拒绝它们。

这样，原本需要排队的请求就不需要消耗在无谓的等待上了，可以直接被处理，而对于超过突发值的请求还是被拒绝的。

在这个模式下，在控制请求速率的同时，允许了一定程度的突发，并且这些突发的请求由于不需要排队，它能够立即得到处理，改善了延迟体验。 



#### delay参数

在某些场景下，我们既需要保证正常的少量关联资源能够被快速地加载，同时也需要对突发请求及时地进行限制。因此，这时就需要delay参数了，它配合burst参数能实现更精细的控制。

比如，某页面引用的资源大概为2-3个，我们需要确保并发加载这些资源，同时我们又能肯定资源不会超过6个，那么针对该页面的限制策略除了burst=6外，还可以设置delay=4。这样，在突发请求的情况下，其中前4个不需要进行延迟处理，后2个则进行排队，而超过6的依然会被拒绝。

![image-20200212182000315](/assets/images/2019/image-20200212182000315.png)

在这个例子中， 一个页面会并发加载其他资源（如JS、CSS等） ，在加载这个页面的时候会跑过来 2-3 个请求， 某个用户点一下页面，服务端收到的是关于这个页面的 2-3 个并发请求；当用户很快地点了两下，服务端收到的是关于这个页面的 4-6 个并发请求，如果我们觉得需要禁止用户很快地刷这个页面刷两次，就需要把超过的这部分请求限制掉，但 burst 设置太小又担心有误伤，设置太大可能就起不到任何效果。 

此时，我们可以配置一个策略，整体突发配置成 6，超过 6 个肯定是需要拒绝的。而在 6范围内，我们希望前面过来的 2-3 个并发请求能够更快地加载，不要进行无效地等待，这里设置 delay=4 ，队列中前 4个等候的请求会直接传给上游，而不会排队，而第 4个之后的请求仍然会排队，但不会被直接拒绝，只是会慢一些，避免在这个尺度内出现一些误伤，同时也起到了一定限制效果（增大时延）。 



## 并发连接数限制 -limit_conn模块

> The `ngx_http_limit_conn_module` module is used to limit the number of connections per the defined key, in particular, the number of connections from a single IP address.
>
>  Not all connections are counted. A connection is counted only if it has a request being processed by the server and the whole request header has already been read. 

该模块用于限制连接的数量。在使用该模块时需要定义一个key，limit_conn会基于这个key来统计它的连接数量是否超过一个设定的阈值。

需要注意的是，只有那些正在被服务器处理的、并且整个请求头已经被读取完的连接才会被统计。



该模块包含如下指令：

- limit_conn_zone（ 在1.1.8之前是limit_zone ）
- limit_conn
- limit_conn_status
- limit_conn_log_level
- limit_conn_dry_run

这些指令的作用大致上与limit_req模块类似，下面只介绍几个重要的模块。



### limit_conn_zone

#### 语法

```nginx
Syntax:	limit_conn_zone key zone=name:size;
Default:	—
Context:	http
```



#### 作用

> Sets parameters for a shared memory zone that will keep states for various keys. In particular, the state includes the current number of connections. The `*key*` can contain text, variables, and their combination. Requests with an empty key value are not accounted.

定义一个共享内存空间。作用基本上与limit_req_zone相同。



#### 示例

```nginx
limit_conn_zone $binary_remote_addr zone=addr:10m;
```

这里存储一个value占用的空间大小始终是32B（32位平台）或64B（64位平台）。如果空间不足，对于其他请求，服务器将直接返回错误码。



### limit_conn

#### 语法

```nginx
Syntax:	limit_conn zone number;
Default:	—
Context:	http, server, location
```



#### 作用

>  Sets the shared memory zone and the maximum allowed number of connections for a given key value. When this limit is exceeded, the server will return the [error](http://nginx.org/en/docs/http/ngx_http_limit_conn_module.html#limit_conn_status) in reply to a request. 

设置限速模块所使用的共享内存空间和允许的最大连接数。

对于超过阈值的连接，服务器将会直接返回错误码。



#### 示例

```nginx
limit_conn_zone $binary_remote_addr zone=addr:10m;

server {
    location /download/ {
        limit_conn addr 1;
    }
```

该配置一次仅允许每个IP地址有一个连接。

> In HTTP/2 and SPDY, each concurrent request is considered a separate connection. 
>
> 注：在HTTP/2和SPDY中，每个并发请求（对于同一个key的并发请求）都被视为一个单独的连接。



同样，limit_conn指令也可以出现多次：

```nginx
limit_conn_zone $binary_remote_addr zone=perip:10m;
limit_conn_zone $server_name zone=perserver:10m;

server {
    ...
    limit_conn perip 10;
    limit_conn perserver 100;
}
```



### 示例

在下载场景中，可能会出现很多用户同时在下载同一个资源，此时我们就可以进行一些限制。比如，同时允许在线5人下载，超过的返回503。

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=addr:10m;

    limit_conn_log_level error;
    limit_conn_status 503;

    server {
        location /download/ {
            limit_conn addr 5;
        }
    }
}
```



## 下载带宽限制

在ngx_http_core_module模块里面有 limit_rate和limit_rate_after指令，这两个指令用于限制下载带宽。



### limit_rate

#### 语法

```nginx
Syntax:	limit_rate rate;
Default:	limit_rate 0;
Context:	http, server, location, if in location
```



#### 作用

> Limits the rate of response transmission to a client. The `*rate*` is specified in bytes per second. The zero value disables rate limiting. The limit is set per a request, and so if a client simultaneously opens two connections, the overall rate will be twice as much as the specified limit.

限制向客户端传输响应数据的速度，单位为B/S。默认值为0，表示不限制传输速度。

这个限制是基于每个请求设置的，如果一个客户端同时打开两个连接，则这个客户端会获得的速度是限制值的两倍。



#### 示例

```nginx
map $slow $rate {
    1     4k;
    2     8k;
}

limit_rate $rate;
```

rate参数可以是一个变量。如上配置，基于某个条件设置不同速率限制。



### limit_rate_after

#### 语法

```nginx
Syntax:	limit_rate_after size;
Default:	limit_rate_after 0;
Context:	http, server, location, if in location
This directive appeared in version 0.8.0.
```



#### 作用

>  Sets the initial amount after which the further transmission of a response to a client will be rate limited. 

设置一个不限速的宽限大小，满足宽限大小后，再开始进行响应数据传输速度的限制。



#### 示例

```nginx
location /flv/ {
    flv;
    limit_rate_after 500k;
    limit_rate       50k;
}
```

该配置表示， 在下载完前面 500KB 数据后，对接下来的数据以每秒 50KB 速度进行限制，即前500K数据不限速，500K之后的响应数据，按每秒50K进行传输。



### 应用

 limit_rate和limit_rate_after这两个指令，在文件下载、视频播放等业务场景中应用比较多，可以避免不必要的浪费。

例如视频播放，第一个画面能够尽快看到，对用户体验来说很重要，如果用户第一个页面看不到，那他的等待忍耐程度是很差的，所以这个场景下前面的几个字节不应该去限速。在看到第一个画面之后，后面的画面是按照一定视频码率播放，所以没必要下载很快，而且快了也没用，它照样是流畅的，但却多浪费了流量资源，如果用户看到一半就关掉，整个视频下载完成，对于用户和内容提供商都是资源浪费。 



## 参考

https://www.upyun.com/opentalk/417.html

http://nginx.org/en/docs/http/ngx_http_limit_req_module.html

http://nginx.org/en/docs/http/ngx_http_limit_conn_module.html

http://nginx.org/en/docs/http/ngx_http_core_module.html#limit_rate


