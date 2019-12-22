---

layout: post
title: 常用的限流算法
category: 容量规划
tags: [容量规划]

---



### 固定窗口

固定窗口就是定义一个“固定”的统计周期，比如30秒或1分钟。然后在每个周期内，通过累加计数器统计当前周期接收到的请求总数。如果达到设定的阈值就触发流量干预，直到进入下一个周期后，计数器清零，流量接收恢复正常状态。

![image-20191222155509849](I:\Vagrantjekyll-yummyMyBlogassetsimages\image-20191222155509849.png)



### 滑动窗口

滑动窗口是对固定窗口做了进一步的细分，将原先的时间周期切的更细（比如，将1分钟的固定窗口切分为60个1秒的滑动窗口）。然后统计的时间窗口会随着时间的流逝而后移。简单来说，就是统计从`当前时间-窗口大小`到`当前时间`之间累计的请求总数。

![image-20191222171152137](I:\Vagrant\jekyll-yummy\MyBlog\assets\images\sliding-window.png)



### 漏桶

漏桶的核心是固定“出口”速率。不管进来多少量，出去的速率一直是这么多。如果进来的量多到桶都装不下了，那么就触发流量干预。

![image-20191222175104246](I:\Vagrant\jekyll-yummy\MyBlog\assets\images\leaky-bucket.png)

 

### 令牌桶

令牌桶的核心是固定“入口”速率。先拿令牌，再处理请求。如果拿不到令牌就触发流量干预。因此，当大量的流量进入时，只要令牌的生成速度大于等于请求被处理的速度，那么此刻程序的处理能力就是极限。

![image-20191222180716166](I:\Vagrant\jekyll-yummy\MyBlog\assets\images\token-bucketpng)



### 参考

https://www.infoq.cn/article/UhixHoWebU_TYJewJwcL









