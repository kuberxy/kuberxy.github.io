---

layout: post
title: OpenResty - Hello World
date:   2019-12-29 10:02:42 +0800
category: OpenResty
tags: [HTTP]
typora-root-url: ../..
---

## 环境搭建

```shell
# 安装导入 GPG 公钥时所需的几个依赖包（整个安装过程完成后可以随时删除它们）：
sudo apt-get -y install --no-install-recommends wget gnupg ca-certificates

# 导入我们的 GPG 密钥：
wget -qO - https://openresty.org/package/pubkey.gpg | sudo apt-key add -

# 安装 add-apt-repository 命令
# （之后你可以删除这个包以及对应的关联包）
sudo apt-get -y install --no-install-recommends software-properties-common

# 添加我们官方 official APT 仓库：
# sudo add-apt-repository -y "deb http://openresty.org/package/ubuntu $(lsb_release -sc) main"
sudo tee /etc/apt/sources.list.d/openresty.list <<EOF
deb http://openresty.org/package/ubuntu $(lsb_release -sc) main
# deb-src http://openresty.org/package/ubuntu $(lsb_release -sc) main
EOF

# 更新 APT 索引：
sudo apt-get update

# 安装openresty
sudo apt-get -y install openresty
```

需要注意的是，安装openresty的同时会安装openresty-opm 和 openresty-restydoc，如果不想安装这两个包请使用如下命令安装openresty：

```shell
sudo apt-get -y install --no-install-recommends openresty
```



## Hello World

### 命令行

```shell
resty -e 'print("hello, world")'
```



### HTTP Server

#### 创建工作目录

```shell
mkdir ~/work
cd ~/work
mkdir logs/ conf/
```



#### 生成配置文件

```shell
tee conf/openresty.conf <<EOF
worker_processes  1;
error_log logs/error.log;

events {
    worker_connections 1024;
}

http {
    server {
        listen 8080;
        location / {
            default_type text/html;
            content_by_lua_block {
                ngx.say("<p>hello, world</p>")
            }
        }
    }
}
EOF
```



#### 运行openresty

```shell
openresty -p `pwd`/ -c conf/openresty.conf
```



#### 访问

```shell
curl http://localhost:8080/
```

