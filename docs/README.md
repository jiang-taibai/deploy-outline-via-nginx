# 0. 前言

## I. 需求

由于我希望能够有一款**可私人部署**、**多人实时协同**、**用户权限管理**、**自由分享到互联网**的**Markdown**知识库管理平台，但依次尝试了现有的腾讯文档智能文档、语雀、Docsify、Obsidian、HackMD、CodiMD、Typora、思源等等等等，都不能很完美地满足我的需求（这并非踩一捧一，我现在就是在用 Typora 本地完成这篇文章，再用 Docsify 部署到云端，看需求选择软件）

其中 HackMD 已经很接近，唯一美中不足的是不能权限管理，要么都不能协同，要么全世界人都可以协同（等同于拥有删除权限），所以用起来还是不太放心。

我也不记得在哪看到的介绍，总之在我快要放弃寻找的时候发现了超级厉害的 Github 20+K⭐ 的 **Outline**: [https://github.com/outline/outline](https://github.com/outline/outline)，说实话我在中文互联网上几乎看不到它的身影和宣传视频，即使是特意去搜，也寥寥无几，这不是因为它不优秀，我总结原因有以下几点: 

- **名称大众化**: Outline 可以是 CSS、VPN、标题中一个普普通通的动词和名词，甚至我搜 Outline 出来的是 Obsidian 的插件……

- **部署困难**: 部署起来的难度巨大，依赖服务特别多
- **第三方 OIDC 服务门槛高**: 第一道门槛是费用，第二道门槛是企业邮箱。**所以我们选择本地部署的 OIDC 服务，而要实现私人部署的单点登录系统的话是相当复杂的！**

由于该项目需要单点登录服务、对象存储服务和各种复杂的配置，即使有官方文档帮忙，也还是困难重重。而其他人分享的部署方法有些地方没有讲清楚，有些地方也不能很好地适应所有人的服务器（因为毕竟服务器可能还需要跑其他软件，80端口早就被占用了）

经过这几天的部署实战，摸索出了一套基于 Nginx All In One 的 Docker 网络架构部署模式，只要你的服务器是用 Nginx 代理，那么这套方法很适合你。

我相信你一定能在 30 分钟内完成这一切的。


## II. Outline 简介

官网: https://www.getoutline.com/

相关介绍视频: 

- Bilibili: https://www.bilibili.com/video/BV1zs4y1f7J3
- Youtube: https://www.youtube.com/watch?v=UjG4lB2u-r4&t=343s



Outline 是一个开源的知识库和团队协作工具🧠，旨在帮助团队共享、组织和协作文档📝。它提供了一个简洁的界面，使用户能够轻松创建、编辑和查看文档。

以下是 Outline 的一些主要特点: 

1. **实时协作**👥: 团队成员可以实时编辑和评论文档，提高协作效率。
2. **富文本编辑器**📄: 支持 Markdown 语法，允许用户轻松格式化文本、插入图片、链接等。
3. **文档组织**📂: 用户可以通过文件夹和集合来组织文档，使内容易于查找和管理。
4. **权限管理**🔒: 可以设置不同级别的访问权限，确保敏感信息的安全。
5. **集成第三方服务**🔗: Outline 可以与 Slack、GitHub 等第三方服务集成，方便团队协作。
6. **自托管或云服务**☁️: Outline 可以在自己的服务器上托管，也可以使用官方提供的云服务。
7. **开源**💻: Outline 是开源软件，允许开发人员根据自己的需求进行定制和扩展。

总的来说，Outline 是一个功能丰富的文档管理和团队协作平台🚀，适用于各种规模的团队和项目。

【GPT By ChatGPT4.0】

![0-outline-updated](./assets/0-outline-updated.png)



## III. 环境说明

下面是我服务器生产环境的详细说明，如有不同不代表不能运行，可能存在细微差别，比如 CentOS 和 Ubuntu

- 服务器: 腾讯云-轻量应用服务器
- 系统架构: AMD x86_64
- 系统版本: CentOS Linux release 7.6.1810 (Core)
- 实例规格: 
  - CPU: 2 核
  - 内存: 4 GB
  - 系统盘: SSD 云硬盘 50 GB
  - 流量包: 800 GB/月（带宽: 6 Mbps）



## IV. Contact

本文于2023年08月13日完成，如有问题欢迎联系我😊: 

- emailtojiang@gmail.com
- emailtojiang@163.com



# 1. 架构图

下图为整个项目的系统架构: 

![1-architecture-diagram](./assets/1-architecture-diagram-v4.png)

涉及到的 Outline 服务以及它的依赖服务: 

- **Outline**: 超级棒的团队多人协同文档管理开源项目！3000 端口为访问 Outline 的端口，但该端口并不暴露，由 Nginx 通过 Docker Network 方式访问
- **Keycloak**: 一个支持 OpenID Connect（下文简称 **OIDC**）的开源项目。用于 Outline 的单点登录服务。
- **Redis**: 非关系型数据库，Outline 使用 Redis 实现缓存、消息队列、会话存储、实时协作等功能
- **PostgreSQL**: 关系型数据库，Outline 使用 PostgreSQL 实现数据的持久化
- **Minio**: 一款本地**对象存储系统**的开源项目。用于存储 Outline 的图片等资源

网络架构主干:

- **nginx_all_in_one**: Docker Network，使用虚拟网卡实现多个容器之间的网络互通
- **Nginx**: 占用主机的 80, 443 端口并反代了四个域名，而反代的端口确实来自虚拟局域网中的端口，由图中可知整个网络只有 80 和 443 端口暴露在外。

四个域名的作用: 

- `outline.example.com`: 用于访问 Outline 的主域名
- `sso.example.com`: 提供身份权限验证服务，同时也是管理员入口
- `minio.example.com`: MinIO API 为 Outline 提供对象存储服务 OSS
- `minio.example.com`: MinIO Admin 界面

这种部署方式的优点如下: 

- **安全性更高**🔒: 所有服务均隐藏与虚拟局域网中，并在虚拟局域网中通信，不会暴露在互联网上
- **门槛低**🎁: 全文所有服务项均使用 Docker 部署，并使用 Compose 插件，俗称开箱即用，部署难度大大降低。通常情况下，你只需要新建一个 `yaml` 配置文件和执行一条命令 `docker-compose up -d` 即可。
- **非侵入式的端口友好型**🚪: 所有服务都不会占用服务器的任何一个端口。由占用 80 端口的 Nginx 负责反向代理转发到 虚拟局域网 中的服务访问点 SAP
- **非侵入式的环境友好型**🐳: 所有服务均使用 Docker 容器化部署，不会在服务器中创建一大堆的环境变量
- **后期调试友好型**🔧: 所有服务均使用 Docker Compose 插件部署，所有配置项、密码等都保存在了服务器中，较原先的纯 Docker 部署而言，对于后期调试无需翻找当时设置的所有配置项。

---

下面将正式开始完整的部署流程 🚀🛠️



# 2. Nginx 部署

## 2.1 概述

Nginx 是一个高性能的 HTTP 和反向代理 WEB 服务器。

我们使用他可以让我们用多个域名访问同一个 IP 的同一个端口，同时还可以让我们的应用支持 HTTPS 服务。

在本文中，我们使用 Nginx 反代了 Outline、Keycloak、MinIO



## 2.2 配置域名

### 2.2.1 需要了解的

**注意‼: 全文中出现的任何 `*.example.com` 一律切换成自己的域名**

**假如我的域名是 `mydomain.xyz`，那么 `outline.example.com` 应该对应着我的 `outline.mydomain.xyz`**

由架构图可知，我们共需配置四个域名: 

- **outline.example.com**: 访问笔记的主入口
- **sso.example.com**: OIDC 单点登录页面，点击 outline 首页的登陆自动跳转
- **minio.example.com**: 对象存储服务的 API 访问点
- **minio-admin.example.com**: 对象存储服务的后台管理访问点

**以下均以腾讯云服务器为例，其他服务器大致过程类似，举一反三即可**



### 2.2.2 服务器域名配置

登陆 [DNS 解析 DNSPod](https://console.cloud.tencent.com/cns) 打开你的域名，点击添加记录

![2-click-to-add](./assets/2-click-to-add.png)

添加如下四个子域名，记录值即服务器IP为你自己服务器的IP

![2-subdomain-configuration](./assets/2-subdomain-configuration.png)



## 2.3 获取 SSL 证书

腾讯云服务器为每名用户提供了 50 张免费证书的申请资格（过期或自行删除资格数恢复）

登陆 [SSL 证书](https://console.cloud.tencent.com/ssl)，依次打开 我的证书-免费证书-申请免费证书

![2-apply-free-ssl](./assets/2-apply-free-ssl.png)

注意下面域名是自己的，依次绑定`outline.example.com`、`sso.example.com`、`minio.example.com`、`minio-admin.example.com`

选择**自动DNS验证**: 会自动帮你添加域名验证记录，不用手动添加（如果其他服务器只有手动，则根据相对应提示自行添加域名验证记录）

勾选**自动删除验证**: 验证完可自由删除DNS域名验证记录（如果其他服务器没有也不要紧）

![2-apply-free-ssl-2](./assets/2-apply-free-ssl-2.png)

SSL 证书将交由后台审核，在此期间先完成Nginx的部署。



## 2.4 使用 Docker&Compose 部署 Nginx 与网络

TIP💡: SSL 申请页面先别关，之后还需要下载 SSL 证书。

如果你还没有安装 Docker 和 Docker-Compose，可以先参考 [官方文档](https://docs.docker.com/compose/) 或 其他博客。这里提醒一句如果你已经装好了的 Docker 版本过低的话，是需要考虑版本对应关系的: [版本对应关系](https://docs.docker.com/compose/compose-file/compose-versioning/)



### 2.4.1 创建 Docker Network

我们使用以下命令创建名为 `nginx_all_in_one` 的 Docker Network，当然这个名字是自定义的

```bash
# 创建名为 "nginx_all_in_one" 的 Docker Network
docker network create -d bridge nginx_all_in_one
# 也可以直接创建，因为默认网络类型就是 bridge
docker network create nginx_all_in_one
```



### 2.4.2 创建 Nginx

在这里约定本文中所有内容均放置在 `/home/docker-compose/` 目录下

首先创建文件夹和 `docker-compose.yaml`（`docker-compose` 默认以当前目录下的 `docker-compose.yaml` 启动，为了与后文中的 `yaml` 相区别，均使用相对清晰的名称以表示该 `yaml` 的作用）

```bash
# 递归地创建文件夹
mkdir -p /home/docker-compose/nginx
# 创建 nginx-docker-compose.yaml
touch /home/docker-compose/nginx/nginx-docker-compose.yaml
```

你不必新建 `data` 文件夹，因为在启动后会自动生成，所以展示忽略之

![2-where-is-nginx-docker-compose](./assets/2-where-is-nginx-docker-compose.png)

之后不再展示命令行新建文件夹和文件的方式，因为用一些 FTP 工具可能更适合，例如 XFTP 等。

在 `nginx-docker-compose.yaml` 文件中输入一下内容: 

```yaml
version: '3.8'
services:
  nginx:
    image: nginx:1.23.3
    container_name: nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./data/conf/conf.d:/etc/nginx/conf.d
      - ./data/conf/nginx.conf:/etc/nginx/nginx.conf
      - ./data/conf/ssl:/etc/nginx/ssl
      - ./data/logs:/var/log/nginx
      - ./data/www:/usr/share/nginx/html
    networks:
      - nginx_all_in_one
networks:
  nginx_all_in_one:
    external: true
```

这里解释一下 Docker-Compose 配置文件中经常出现的字段以及释义，后文将不再赘述

|        字段        |   解释   | 备注                                                         |
| :----------------: | :------: | ------------------------------------------------------------ |
|      `image`       |   镜像   | 此后的镜像都指定版本号，可自行升级新版，但不保证能适应新特性 |
|      `ports`       | 端口暴露 | `ports` 暴露给宿主机，而 `expose` 仅暴露给同一局域网的其他容器 |
|     `valumes`      |   挂载   | 在当前目录下保存 `Nginx` 数据以实现持久化                    |
|     `networks`     |   网络   | 可选择多个网络，此处将 Nginx 连入 `nginx_all_in_one` 网络中  |
| `nginx_all_in_one` | 网络名称 | 此处 `external: true` 表示该网络已经创建                     |

再执行以下命令启动 `Nginx`

```bash
# 确保进入到 "nginx-docker-compose.yaml" 同级目录
cd /home/docker-compose/nginx
# 使用 docker-compose 启动 nginx
# -f 表示指定 yaml 配置文件
# -d 表示后台运行，不需要显示日志到控制台上
# up 表示启动
docker-compose -f nginx-docker-compose.yaml up -d
```

这里推荐你安装 `Portainer`，一款 Docker 可视化管理工具，方便你查看 Docker 运行状态

安装好后，你应该会在 `/home/docker-compose/nginx/data` 下看到如下文件夹

![2-after-exec-nginx-docker-compose](./assets/2-after-exec-nginx-docker-compose.png)



##  2.5 配置 Nginx 反代 & SSL 证书

### 2.5.1 安装 SSL 证书

在 "2.3 获取 SSL 证书" 中，我们申请了四个域名的 SSL 证书，现在我们把他下载下来

![2-download-ssl](./assets/2-download-ssl.png)

一定要选择对应的 `Nginx` 证书，可以先下载到本地，然后再用譬如 `XFTP` 工具上传到服务器

![2-download-ssl-for-nginx](./assets/2-download-ssl-for-nginx.png)

将 4 个域名的 8 个证书文件放到 `/home/docker-compose/nginx/data/conf/ssl/` 文件夹下，分别是: 

- **\*.example.com.key**: 私钥文件，用于解密通过公钥加密的信息。
- **\*.example.com_bundle.crt**: 证书包文件，通常包括一个或多个证书。

![2-ssl-file-list](./assets/2-ssl-file-list.png)

### 2.5.2 配置反向代理规则

在 `/home/docker-compose/nginx/data/conf/conf.d/` 文件夹下新建以下文件，下面将依次给出四个文件的内容

**注意‼: 记得将配置文件中的所有 `*.example.com` 换成你自己的域名**

![2-nginx-config-file-list](./assets/2-nginx-config-file-list.png)

(1) minio-admin.conf

```nginx
server {
    listen 80;
    listen 443 ssl;
    charset utf-8;
    server_name minio-admin.example.com;

    #证书文件名称
    ssl_certificate ssl/minio-admin.example.com_bundle.crt; 
    #私钥文件名称
    ssl_certificate_key ssl/minio-admin.example.com.key; 
    ssl_session_timeout 5m;
    #请按照以下协议配置
    ssl_protocols TLSv1.2 TLSv1.3; 
    #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; 
    ssl_prefer_server_ciphers on;
    
    # Allow special characters in headers
    ignore_invalid_headers off;
    # Allow any size file to be uploaded.
    # Set to a value such as 1000m; to restrict file size to a specific value
    client_max_body_size 100m;
    # Disable buffering
    proxy_buffering off;
    proxy_request_buffering off;

    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-NginX-Proxy true;

        # This is necessary to pass the correct IP to be hashed
        real_ip_header X-Real-IP;

        proxy_connect_timeout 300;

        # To support websockets in MinIO versions released after January 2023
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        chunked_transfer_encoding off;
        
        proxy_pass http://minio:9001;
    }
}
```



(2) minio.conf

```nginx
server {
    listen 80;
    listen 443 ssl;
    charset utf-8;
    server_name minio.example.com;

    #证书文件名称
    ssl_certificate ssl/minio.example.com_bundle.crt; 
    #私钥文件名称
    ssl_certificate_key ssl/minio.example.com.key; 
    ssl_session_timeout 5m;
    #请按照以下协议配置
    ssl_protocols TLSv1.2 TLSv1.3; 
    #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; 
    ssl_prefer_server_ciphers on;
   
    # Allow special characters in headers
    ignore_invalid_headers off;
    # Allow any size file to be uploaded.
    # Set to a value such as 1000m; to restrict file size to a specific value
    client_max_body_size 100m;
    # Disable buffering
    proxy_buffering off;
    proxy_request_buffering off;

    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 300;
        # Default is HTTP/1, keepalive is only enabled in HTTP/1.1
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        chunked_transfer_encoding off;

        proxy_pass http://minio:9000;
    }

}
```



(3) outline.conf

```nginx
server {
    listen 80;
    listen 443 ssl;
    charset utf-8;
    server_name outline.example.com;

    #证书文件名称
    ssl_certificate ssl/outline.example.com_bundle.crt; 
    #私钥文件名称
    ssl_certificate_key ssl/outline.example.com.key; 
    ssl_session_timeout 5m;
    #请按照以下协议配置
    ssl_protocols TLSv1.2 TLSv1.3; 
    #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; 
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://outline:3000/;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;proxy_set_header Host $host;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }

}
```



(4) sso.conf

```nginx
server {
    listen 80;
    #SSL 访问端口号为 443
    listen 443 ssl;
    charset utf-8;
    server_name sso.example.com;

    #证书文件名称
    ssl_certificate ssl/sso.example.com_bundle.crt; 
    #私钥文件名称
    ssl_certificate_key ssl/sso.example.com.key; 
    ssl_session_timeout 5m;
    #请按照以下协议配置
    ssl_protocols TLSv1.2 TLSv1.3; 
    #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; 
    ssl_prefer_server_ciphers on;
   
    location / {
        proxy_pass http://keycloak:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }

}
```

**【以下内容为配置项的出处与配置出错可能出现的问题，可选择性阅读】**

**proxy_pass 的解释:** 

所有 `host` 都用了相对应容器的名称，这其实是minio、outline、sso都连入了 `nginx_all_in_one` 的虚拟局域网，同时该虚拟局域网又配置了域名转换关系，可以通过 `containner_name` 索引到相对应的 局域网IP



**outline.conf 配置项:**

来源于 Outline 的官方文档 [Outline/Configuration/Nginx](https://docs.getoutline.com/s/hosting/doc/nginx-6htaRboR57)

如果不配置，那么在登陆成功后（Keycloak显示校验成功）却被告知 `身份验证失败，我们无法让你登陆`



**minio.conf 配置项**: 

来源于 MinIO 的官方文档 [Integrations/Configure NGINX Proxy for MinIO Server](http://minio.org.cn/docs/minio/linux/integrations/setup-nginx-proxy-with-minio.html)。

如果不配置，将无法正常登录 Admin 端，也无法上传文件



**sso.conf 配置项**: 

来源于 Keycloak 的官方文档 [Guides/Server/Using a reverse proxy](https://www.keycloak.org/server/reverseproxy)

Keycloak 的某些功能依赖于这样一个假设，即连接到 Keycloak 的 HTTP 请求的远程地址是客户端机器的真实 IP 地址。

在边缘或重新加密代理模式下，Keycloak 将解析以下标头，并期望反向代理设置它们: 

- 按照 RFC7239 的 Forwarded
- 非标准的 X-Forwarded
- 非标准的 X-Forwarded-*，例如 X-Forwarded-For、X-Forwarded-Proto、X-Forwarded-Host 和 X-Forwarded-Port

### 2.5.3 导入到 Nginx 配置

在 `/home/docker-compose/nginx/data/conf/` 目录下，你应该可以看到 `nginx.conf` 文件，现在我们需要配置该文件，引入 `./conf.d/` 下的所有配置文件

![2-whereis-nginxconf](./assets/2-whereis-nginxconf.png)

这是 `nginx.conf` 的内容: 

```nginx
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    keepalive_timeout  65;
    gzip  on;
    # 导入所有配置文件
    include /etc/nginx/conf.d/*;
}
```



### 2.5.4 启用最新配置

使用如下命令进入到 Nginx 的容器内，并刷新配置: 

```bash
# 进入 Docker 容器
docker exec -it nginx /bin/bash

# 校验配置
nginx -t
# 如果输出以下内容，就说明配置文件校验成功
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful

# 再刷新 Nginx 配置
nginx -s reload

# 没什么问题的话，可以输入 exit 退出
exit
```

到此为止，你已经完成了 "1. 架构图" 中的大致框架了，接下来让我们继续部署另外几个应用吧 🚀🛠️



# 3. PostgreSQL 部署

## 3.1 概述

PostgreSQL 是一个关系型数据库，未来我们的 Outline 和 Keycloak 将使用该数据库存储数据



## 3.2 已有 PostgreSQL 服务

如果你服务器已经部署了 PostgreSQL，并且也是基于 Docker 部署的，那么可以跳过下一节

你可以直接使用下面的命令让 PostgreSQL 接入 `nginx_all_in_one` 网络（你需要将 `name_of_postgres_container` 换成对应的容器名）

```bash
docker network connect nginx_all_in_one name_of_postgres_container
```



如果你尚未部署或并非使用 Docker 部署的，那么可以考虑再部署一个 PostgreSQL 或者先导出所有数据，再切换到 Docker-Compose 部署。



## 3.3 使用 Docker-Compose 部署 PostgreSQL

在 `/home/docker-compose/postgres/` 下创建 `postgres-docker-compose.yaml`

![3-where-is-postgres-docker-compose](./assets/3-where-is-postgres-docker-compose.png)

并写入以下内容: 

关于 `expose`: 这里将 PostgreSQL 接入了 `nginx_all_in_one` 网络，该网络下的容器只能互相访问对应容器的 expose 的端口。所以这里必须设置，否则 Outline 和 Keycloak 将无法访问 `http://postgres12:5432` 实现数据持久化

```yaml
version: '3.8'
services:
  postgres12:
    image: postgres:12.10
    container_name: postgres12
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: password
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - ./data:/var/lib/postgresql/data
    expose:
      - 5432
    networks:
        - nginx_all_in_one
networks:
  nginx_all_in_one:
    external: true
```

**注意‼: 你需要修改 `POSTGRES_PASSWORD: password` 作为数据库的密码**

由于后文需要设置的密码特别多，而这些密码已经配置其实就不会再手动输入了，而且密码都保存在了 `*-docker-compose.yaml` 文件中，所以我建议直接使用随机字符串生成并填写

我使用的是 [Avast 随机密码生成器](https://www.avast.com/zh-cn/random-password-generator#pc)，但由于是在线工具，我还是不太放心，以后接触到更好的再更新(Mark as TODO)

此外再建议尽量别用特殊字符，就`小写字母`+`数字`+`30+位`的随机字符串密码应该很安全吧~有时候特殊字符会被当做命令符或者转义字符，容易出现乱七八糟的BUG

接着使用以下命令，创建 PostgreSQL

```bash
# 确保进入到 "postgres-docker-compose.yaml" 同级目录
cd /home/docker-compose/postgres
# 使用 docker-compose 启动 redis
docker-compose -f postgres-docker-compose.yaml up -d
```

## 3.4 创建用户与数据库

需要分别为 Outline 和 Keycloak 项目分配一个账号和相关数据库

**注意‼: 你需要修改 `password_of_*_user` 作为数据库的密码**

```postgresql
# 进入 PostgreSQL 数据库命令行
docker exec -it --user postgres postgres12 psql -U postgres

# 创建用户
CREATE USER outline WITH PASSWORD 'password_of_outline_user';
CREATE USER keycloak WITH PASSWORD 'password_of_keycloak_user';

# 创建数据库
CREATE DATABASE outline OWNER outline;
CREATE DATABASE outline_test OWNER outline;
CREATE DATABASE keycloak OWNER keycloak;

# 退出命令行
\q
```



# 4. Redis 部署

## 4.1 概述

Redis 是一个非关系型数据库，是基于内存的数据读写方式，比 Postgres 响应速度快，但仍然需要 PostgreSQL 完成数据持久化。二者相辅相成。

在 Outline 中，使用 Redis 实现缓存、消息队列、会话存储、实时协作等功能。



## 4.2 使用 Docker-Compose 部署 Redis

在 `/home/docker-compose/redis/` 下，创建 `redis-docker-compose.yaml`

![4-where-is-redis-docker-compose](./assets/4-where-is-redis-docker-compose.png)

该文件内容如下: 

**注意‼: 你需要修改 `password_of_redis` 作为 Redis 的密码**

```yaml
version: '3.8'
services:
  redis:
    image: redis:7.0.8
    container_name: redis
    restart: unless-stopped
    command: 
      --requirepass "password_of_redis"
    expose:
      - 6379
    volumes:
      - ./data:/data
    privileged: true
    networks:
      - nginx_all_in_one
networks:
  nginx_all_in_one:
    external: true
```

接着使用以下命令创建该容器

```bash
# 确保进入到 "redis-docker-compose.yaml" 同级目录
cd /home/docker-compose/redis
# 使用 docker-compose 启动 redis
docker-compose -f redis-docker-compose.yaml up -d
```



# 5. Keycloak 部署

## 5.1 概述

Keycloak 是一个可以提供 Outline 所需的 OIDC 单点登录服务。OIDC（OpenID Connect）是一种基于 OAuth 2.0 协议的身份层，通过允许客户端验证用户身份并获取基本个人信息，实现跨系统的单一登录。

虽然 Outline 支持多种 OIDC 在线服务，例如 Google Workspace 等，但大多数是收费、需要企业邮箱的。抛开价格门槛不谈，账户数据存储在云端总觉得会产生依赖感，所以还是选择了本地 OIDC 服务。

Keycloak 是一个非常庞大且健全的 OIDC 服务，所以部署起来也是相当困难，这几乎占据了我整个部署时间的 1/3。

下面将详细说明 Keycloak 的部署流程



## 5.2 使用 Docker-Compose 部署 Keycloak

在 `/home/docker-compose/keycloak/` 下，创建 `keycloak-docker-compose.yaml`

![5-where-is-keycloak-docker-compose](./assets/5-where-is-keycloak-docker-compose.png)

文件内容如下: 

```yaml
version: "3.8"
services:
  # keycloak
  keycloak:
    image: jboss/keycloak:15.0.2
    container_name: keycloak
    expose:
        - 8080
    volumes:
        - /etc/timezone:/etc/timezone
        - /etc/localtime:/etc/localtime
    environment:
        # 初始化密码
        - KEYCLOAK_USER=admin
        - KEYCLOAK_PASSWORD=password_of_keycloak
        # DB
        - DB_VENDOR=postgres
        - DB_ADDR=postgres12
        - DB_PORT=5432
        - DB_DATABASE=keycloak
        - DB_USER=keycloak
        - DB_PASSWORD=password_of_keycloak_user
        # 开启反向代理
        - PROXY_ADDRESS_FORWARDING=true
    labels:
        - "traefik.enable=true"
        - "traefik.docker.network=nginx_all_in_one"
        - "traefik.http.routers.halobug-sso.entrypoints=https"
        - "traefik.http.routers.halobug-sso.tls=true"
        - "traefik.http.routers.halobug-sso.rule=Host(`sso.example.com`)"
        - "traefik.http.services.halobug-sso.loadbalancer.server.scheme=http"
        - "traefik.http.services.halobug-sso.loadbalancer.server.port=8080"
    logging:
      driver: "json-file"
      options:
        max-size: "1m"
    networks:
        - nginx_all_in_one
networks:
  nginx_all_in_one:
    external: true
```

需要**更改**的字段有: 

- `password_of_keycloak`: Keycloak 的密码，是新定义的
- `password_of_keycloak_user`: 在 PostgreSQL 中定义的 Keycloak 用户密码，**已经定义好了的，别复制错了**
- `sso.example.com`: 在 labels 中，记得改成你自己的域名



需要**注意**的字段有: 

- `DB_ADDR=postgres12`: `postgres12` 是我们在部署 PostgreSQL 时 `yaml` 文件中，`container_name` 定义的名称。如果你用的别的名字或者以前就部署好了，一定要检查是什么名字！



**不需要更改**的字段有: （容易被误解的字段）

- `expose:8080`: 一般 8080 端口早就其他软件被占用了，但这里不需要更改端口，因为他只会占用自己局域网下的局域网 IP 的 8080 端口。所以请放心设置。

- `/etc/timezone:/etc/timezone`: 这里是绑定主机的时区到虚拟主机，当然你需要确保你主机有这个文件，可以使用下面这个命令生成

  - ```bash
    # 如果不存在，就会报错
    ls /etc/timezone
    # 列出所有可用的时区（输入q退出）
    timedatectl list-timezones
    # 在列表中找到 "Asia/Shanghai" 或你所需的其他中国时区
    timedatectl set-timezone Asia/Shanghai
    # 检查是否生效
    timedatectl status
    ```


- `/etc/localtime:/etc/localtime`: `/etc/localtime` 文件是一个链接到实际时区数据文件的符号链接。当你使用 `timedatectl` 或其他时区设置工具更改时区时，这个链接通常会自动更新。

  - ```bash
    # 选择时区: 确定你想要设置的时区。例如，对于中国上海时区，你可以使用 /usr/share/zoneinfo/Asia/Shanghai
    # 创建符号链接: 使用以下命令创建或更新 /etc/localtime 的符号链接
    sudo ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
    # 验证设置: 你可以使用以下命令检查 /etc/localtime 是否正确链接到所需的时区文件
    ls -l /etc/localtime
    # 检查系统时区: 你还可以使用 date 命令来检查系统时区是否正确
    date
    # 手动更改 /etc/localtime 文件可能会与系统的时区管理工具冲突。建议使用 timedatectl 或你的发行版推荐的工具来更改时区，以确保所有相关的设置都正确更新
    ```



最后的最后，使用以下命令部署 Keycloak

```bash
# 确保进入到 "keycloak-docker-compose.yaml" 同级目录
cd /home/docker-compose/keycloak
# 使用 docker-compose 启动 keycloak
docker-compose -f keycloak-docker-compose.yaml up -d
```



## 5.3 配置 Outline 的 OIDC 服务

接下来就是使用 Keycloak 配置 Outline 的 OIDC 服务



### 5.3.1 登陆

打开 `sso.example.com`，会自动跳转至 `sso.example.com/auth/`

进入左边的 **Administration Console**

![5-use-keycloak-01](./assets/5-use-keycloak-01.png)

输入刚刚定义的 `password_of_keycloak`，登陆

![5-use-keycloak-02](./assets/5-use-keycloak-02.png)



### 5.3.2 创建 Outline Realm

创建一个新的 Realm，输入名称 `outline`（注意是小写）

我这里已经创建了 Outline 的 Realm，可以看到 Select Realm 下共有两个 Realm，紧接着选中 Outline 的 Realm

![5-use-keycloak-03](./assets/5-use-keycloak-03.png)



### 5.3.3 创建 Outline Client

创建一个 Client

![5-use-keycloak-04](./assets/5-use-keycloak-04.png)

注意 Root URL 应该是你自己的域名

![5-use-keycloak-05](./assets/5-use-keycloak-05.png)



### 5.3.4 配置 Outline Client

之后自动进入到该 Client 的设置界面，需要修改以下字段

![5-use-keycloak-06](./assets/5-use-keycloak-06.png)

注意别忘了 `Valid Redirect URIs` 字段中的域名后面有 **`/*`**

![5-use-keycloak-07](./assets/5-use-keycloak-07.png)

最后保存

![5-use-keycloak-08](./assets/5-use-keycloak-08.png)

保存之后上方新添了一个选项卡 `Credentials`，这里可以获取 Outline 所需的 `OIDC_CLIENT_SECRET`。这里先记一下，后面会用到的。

![5-use-keycloak-09](./assets/5-use-keycloak-09.png)



### 5.3.5 创建 Outline Client Role

接着为 Outline Realm 创建一个用户组，该用户组用来授权哪些用户可以访问 Outline（请忽略我已经创建了 `outline-user`）

![5-use-keycloak-10](./assets/5-use-keycloak-10.png)

输入名称 `outline-user`，当然这个是自定义的，你可以取其他名字

![5-use-keycloak-11](./assets/5-use-keycloak-11.png)

### 5.3.6 创建 Outline User

紧接着创建需要访问 Outline 的用户

![5-use-keycloak-12](./assets/5-use-keycloak-12.png)

这里注意 Email、First Name、Last Name 尽量不要留空吧，我留空没有测试过，有看过说因为名称为空导致无法登陆的问题，记得这个可能的问题，为以后 Debug 找到思路。

![5-use-keycloak-13](./assets/5-use-keycloak-13.png)

然后为用户设置密码。

注意: `Temporary` 字段意思是当前设置的密码只是临时的，用户登陆时将要求用户更改密码。这个可开可不开，看自己应用需求。

![5-use-keycloak-16](./assets/5-use-keycloak-16.png)



### 5.3.7 授权 Outline User 访问 Outline Client 权限

再将用户加入到之前的 `outline-user` 的 Client Roles，以便可以访问 Outline 应用

![5-use-keycloak-14](./assets/5-use-keycloak-14.png)

设置好后应该是这样的

![5-use-keycloak-15](./assets/5-use-keycloak-15.png)



# 6. MinIO 部署

## 6.1 概述

MinIO 提供高性能、与 S3 兼容的对象存储系统，让你自己能够构建自己的云储存服务。

Outline 使用 MinIO 实现对于图片、视频、文件等对象存储管理系统。



## 6.2 使用 Docker-Compose 部署 MinIO

在 `/home/docker-compose/minio/` 下，创建 `minio-docker-compose.yaml`

![6-where-is-minio-docker-compose](./assets/6-where-is-minio-docker-compose.png)

`minio-docker-compose.yaml` 的内容如下: 

```yaml
version: '3.8'
services:
  minio:
    image: minio/minio:RELEASE.2021-09-03T03-56-13Z
    container_name: minio
    restart: unless-stopped
    expose:
      - 9000
      - 9001
    environment:
      MINIO_ROOT_USER: "username_of_minio"
      MINIO_ROOT_PASSWORD: "password_of_minio"
      MINIO_REGION_NAME: "cn-outline-1"
      MINIO_BROWSER: "on"
      MINIO_SERVER_URL: "https://minio.example.com/"
      MINIO_BROWSER_REDIRECT_URL: "https://minio-admin.example.com/"
    volumes:
      - ./data:/data
    command: server /data --console-address ":9001"
    networks:
      - nginx_all_in_one
networks:
  nginx_all_in_one:
    external: true
```

需要更改的字段如下: 

- `username_of_minio`: 用于登陆 `minio-admin.example.com` 的账户名，可以设置成例如经典的 admin、root 但是我看其他人的安装教程都是偏向于随机字符串。在官方文档中 [Deploy MinIO: Single-Node Single-Drive](https://minio.org.cn/docs/minio/container/operations/install-deploy-manage/deploy-minio-single-node-single-drive.html#) 给出的示例如下: 

  - ```properties
    # MINIO_ROOT_USER and MINIO_ROOT_PASSWORD sets the root account for the MinIO server.
    # This user has unrestricted permissions to perform S3 and administrative API operations on any resource in the deployment.
    # Omit to use the default values 'minioadmin:minioadmin'.
    # MinIO recommends setting non-default values as a best practice, regardless of environment
    
    MINIO_ROOT_USER=myminioadmin
    MINIO_ROOT_PASSWORD=minio-secret-key-change-me
    ```

- `password_of_minio`: 用于登陆 `minio-admin.example.com` 的账户密码
- `https://minio.example.com/`: 一定要换成自己的域名
- `https://minio-admin.example.com/`: 一定要换成自己的域名

需要注意的字段如下: 

- `cn-outline-1`: 这个是自定义的，可以根据自己喜好取别的名称
- `minio/minio:RELEASE.2021-09-03T03-56-13Z`: 可见该镜像版本比较老，由于我只在此版本做了测试，所以为了不出意外标明了与我本地环境相同的版本。如果你想使用新版本，那么你应该访问官方去了解新特性和可能需要设置的内容。

不需要更改的字段如下: 

- `--console-address ":9001"`: 在默认情况下，MinIO 将在每次服务器启动时为 MinIO 控制台选择一个随机端口。访问 MinIO 服务器的浏览器客户端将自动重定向到 MinIO 控制台的动态选择端口。我们这里指定了一个端口，这也与之前在 Nginx 中设置的 `minio-admin.example.com` 相对应



最后，使用以下命令部署 MinIO

```bash
# 确保进入到 "minio-docker-compose.yaml" 同级目录
cd /home/docker-compose/minio
# 使用 docker-compose 启动 minio
docker-compose -f minio-docker-compose.yaml up -d
```



## 6.3 配置 Outline 的 OSS 服务

打开 `minio-admin.example.com` 或者 `minio.example.com` (自动跳转到 admin 域名)

![6-use-minio-01](./assets/6-use-minio-01.png)

创建一个名为 `outline` 的 Bucket

![6-use-minio-02](./assets/6-use-minio-02.png)

创建一个用户

- Access Key: Outline 用于访问 MinIO API 的访问密钥，任意字符串。可以随机生成一个字符串，因为之后是写入 Outline 配置文件的，并不需要记住。我这里是 16 位的 `小写字母`+`数字`，查资料说是不少于 5  位或 3 位。
- Secret Key: Outline 用于访问 MinIO API 的密钥，任意字符串。可以随机生成一个字符串，因为之后也是写入 Outline 配置文件的，并不需要记住。我这里是 64 位的 `小写字母`+`数字`，查资料说是不少于 8 位。
- Assign Policies: 建议选择 `consoleAdmin` 或者 `readwirte`，即 Outline 用这个密钥对后的权限等级，正常来说 `readwirte` 已经能够完成 增删查改 操作了。

![6-use-minio-03](./assets/6-use-minio-03.png)

最后再创建一个组，并勾选刚刚创建的那个用户。

当然**也可以不创建**这个组，用户没有任何权限管理的情况下是可以访问所有 Bucket 的，这个组目前只是简单的分个类。如果后面你还需要接入多个应用、多个用户，此时用户组对于管理来说就很方便了。

后续的权限管理就不继续设置了，到此为止已经满足 Outline 的需求了。

![6-use-minio-04](./assets/6-use-minio-04.png)



# 7. Outline 部署

## 7.1 概述

终于，在搭建完所有的地基之后，可以开始搭建上层建筑: Outline

Outline 将接入 `nginx_all_in_one` 的 Docker Network，以此访问 PostgreSQL, Redis, Keycloak, MinIO 服务



## 7.2 创建 Outline 配置文件

在 `/home/docker-compose/outline/` 下，创建 `outline-docker-compose.yaml` 和 `outline-docker.env`

![7-where-is-outline-docker-compose](./assets/7-where-is-outline-docker-compose.png)

这里直接给出两个文件的内容，之后新开一节专门讲解每个字段的作用以及注意事项

(1) `outline-docker-compose.yaml`

```yaml
version: "3.8"
services:
  outline:
    image: docker.io/outlinewiki/outline:0.70.2
    container_name: outline
    env_file: ./outline-docker.env
    expose:
      - 3000
    networks:
      - nginx_all_in_one
networks:
  nginx_all_in_one:
    external: true
```

(2) `outline-docker.env`

```properties
# –––––––––––––––– REQUIRED ––––––––––––––––

NODE_ENV=production

# Generate a hex-encoded 32-byte random key. You should use `openssl rand -hex 32`
# in your terminal to generate a random value.
SECRET_KEY=secret_key

# Generate a unique random key. The format is not important but you could still use
# `openssl rand -hex 32` in your terminal to produce this.
UTILS_SECRET=utils_secret

# For production point these at your databases, in development the default
# should work out of the box.
DATABASE_URL=postgres://outline:password_of_outline_user@postgres12:5432/outline
DATABASE_URL_TEST=postgres://outline:password_of_outline_user@postgres12:5432/outline_test
DATABASE_CONNECTION_POOL_MIN=
DATABASE_CONNECTION_POOL_MAX=
# Uncomment this to disable SSL for connecting to Postgres
PGSSLMODE=disable

# For redis you can either specify an ioredis compatible url like this
REDIS_URL=redis://:password_of_redis@redis:6379
# or alternatively, if you would like to provide additional connection options,
# use a base64 encoded JSON connection option object. Refer to the ioredis documentation
# for a list of available options.
# Example: Use Redis Sentinel for high availability
# {"sentinels":[{"host":"sentinel-0","port":26379},{"host":"sentinel-1","port":26379}],"name":"mymaster"}
# REDIS_URL=ioredis://eyJzZW50aW5lbHMiOlt7Imhvc3QiOiJzZW50aW5lbC0wIiwicG9ydCI6MjYzNzl9LHsiaG9zdCI6InNlbnRpbmVsLTEiLCJwb3J0IjoyNjM3OX1dLCJuYW1lIjoibXltYXN0ZXIifQ==

# URL should point to the fully qualified, publicly accessible URL. If using a
# proxy the port in URL and PORT may be different.
URL=https://outline.example.com
PORT=3000

# See [documentation](docs/SERVICES.md) on running a separate collaboration
# server, for normal operation this does not need to be set.
COLLABORATION_URL=

# To support uploading of images for avatars and document attachments an
# s3-compatible storage must be provided. AWS S3 is recommended for redundancy
# however if you want to keep all file storage local an alternative such as
# minio (https://github.com/minio/minio) can be used.

# A more detailed guide on setting up S3 is available here:
# => https://wiki.generaloutline.com/share/125de1cc-9ff6-424b-8415-0d58c809a40f
#
AWS_ACCESS_KEY_ID=access_key_of_minio_outline
AWS_SECRET_ACCESS_KEY=secret_key_of_minio_outline
AWS_REGION=cn-outline-1
AWS_S3_ACCELERATE_URL=
AWS_S3_UPLOAD_BUCKET_URL=https://minio.example.com
AWS_S3_UPLOAD_BUCKET_NAME=outline
AWS_S3_UPLOAD_MAX_SIZE=26214400
AWS_S3_FORCE_PATH_STYLE=true
AWS_S3_ACL=private


# –––––––––––––– AUTHENTICATION ––––––––––––––

# Third party signin credentials, at least ONE OF EITHER Google, Slack,
# or Microsoft is required for a working installation or you'll have no sign-in
# options.

# To configure Slack auth, you'll need to create an Application at
# => https://api.slack.com/apps
#
# When configuring the Client ID, add a redirect URL under "OAuth & Permissions":
# https://<URL>/auth/slack.callback

# SLACK_CLIENT_ID=
# SLACK_CLIENT_SECRET=

# To configure Google auth, you'll need to create an OAuth Client ID at
# => https://console.cloud.google.com/apis/credentials
#
# When configuring the Client ID, add an Authorized redirect URI:
# https://<URL>/auth/google.callback

# GOOGLE_CLIENT_ID=203306213201-3ildae9if5t458eu5dq8f073pjv11rck.apps.googleusercontent.com
# GOOGLE_CLIENT_SECRET=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

# To configure Microsoft/Azure auth, you'll need to create an OAuth Client. See
# the guide for details on setting up your Azure App:
# => https://wiki.generaloutline.com/share/dfa77e56-d4d2-4b51-8ff8-84ea6608faa4

# AZURE_CLIENT_ID=
# AZURE_CLIENT_SECRET=
# AZURE_RESOURCE_APP_ID=

# To configure generic OIDC auth, you'll need some kind of identity provider.
# See documentation for whichever IdP you use to acquire the following info:
# Redirect URI is https://<URL>/auth/oidc.callback
OIDC_CLIENT_ID=outline
OIDC_CLIENT_SECRET=secret_of_keycloak_outline_client
OIDC_AUTH_URI=https://sso.example.com/auth/realms/outline/protocol/openid-connect/auth
OIDC_TOKEN_URI=https://sso.example.com/auth/realms/outline/protocol/openid-connect/token
OIDC_USERINFO_URI=https://sso.example.com/auth/realms/outline/protocol/openid-connect/userinfo

# Specify which claims to derive user information from
# Supports any valid JSON path with the JWT payload
OIDC_USERNAME_CLAIM=preferred_username

# Display name for OIDC authentication
OIDC_DISPLAY_NAME=Keycloak

# Space separated auth scopes.
OIDC_SCOPES=profile email


# –––––––––––––––– OPTIONAL ––––––––––––––––

# Base64 encoded private key and certificate for HTTPS termination. This is only
# required if you do not use an external reverse proxy. See documentation:
# https://wiki.generaloutline.com/share/1c922644-40d8-41fe-98f9-df2b67239d45
SSL_KEY=
SSL_CERT=

# If using a Cloudfront/Cloudflare distribution or similar it can be set below.
# This will cause paths to javascript, stylesheets, and images to be updated to
# the hostname defined in CDN_URL. In your CDN configuration the origin server
# should be set to the same as URL.
CDN_URL=

# Auto-redirect to https in production. The default is true but you may set to
# false if you can be sure that SSL is terminated at an external loadbalancer.
FORCE_HTTPS=false

# Have the installation check for updates by sending anonymized statistics to
# the maintainers
ENABLE_UPDATES=true

# How many processes should be spawned. As a reasonable rule divide your servers
# available memory by 512 for a rough estimate
WEB_CONCURRENCY=2

# Override the maximum size of document imports, could be required if you have
# especially large Word documents with embedded imagery
MAXIMUM_IMPORT_SIZE=5120000

# You can remove this line if your reverse proxy already logs incoming http
# requests and this ends up being duplicative
DEBUG=http

# Configure lowest severity level for server logs. Should be one of
# error, warn, info, http, verbose, debug and silly
LOG_LEVEL=info

# For a complete Slack integration with search and posting to channels the
# following configs are also needed, some more details
# => https://wiki.generaloutline.com/share/be25efd1-b3ef-4450-b8e5-c4a4fc11e02a
#
# SLACK_VERIFICATION_TOKEN=your_token
# SLACK_APP_ID=A0XXXXXXX
# SLACK_MESSAGE_ACTIONS=true

# Optionally enable google analytics to track pageviews in the knowledge base
GOOGLE_ANALYTICS_ID=

# Optionally enable Sentry (sentry.io) to track errors and performance,
# and optionally add a Sentry proxy tunnel for bypassing ad blockers in the UI:
# https://docs.sentry.io/platforms/javascript/troubleshooting/#using-the-tunnel-option)
SENTRY_DSN=
SENTRY_TUNNEL=

# To support sending outgoing transactional emails such as "document updated" or
# "you've been invited" you'll need to provide authentication for an SMTP server
SMTP_HOST=
SMTP_PORT=
SMTP_USERNAME=
SMTP_PASSWORD=
SMTP_FROM_EMAIL=hello@example.com
SMTP_REPLY_EMAIL=hello@example.com
SMTP_TLS_CIPHERS=
SMTP_SECURE=true

# The default interface language. See translate.getoutline.com for a list of
# available language codes and their rough percentage translated.
DEFAULT_LANGUAGE=zh_CN

# Optionally enable rate limiter at application web server
RATE_LIMITER_ENABLED=true

# Configure default throttling parameters for rate limiter
RATE_LIMITER_REQUESTS=1000
RATE_LIMITER_DURATION_WINDOW=60

# Iframely API config
IFRAMELY_URL=
IFRAMELY_API_KEY=
```



## 7.3 `outline-docker.env` 配置文件详解

```properties
# –––––––––––––––– REQUIRED ––––––––––––––––

# ================ Outline ================
NODE_ENV=production
# 一个十六进制编码的 32 字节随机密钥。官方推荐使用 `openssl rand -hex 32` 在你的终端生成
SECRET_KEY=secret_key
# 一个唯一的随机密钥。格式并不重要，官方推荐使用 `openssl rand -hex 32` 在你的终端生成
UTILS_SECRET=utils_secret
# 表示外部浏览器将会以什么域名连接 Outline
# 请一定要设置为 https，否则会出现登陆失败的问题
# 修改项: 
# 1. outline.example.com: 修改为你自己的域名
URL=https://outline.example.com
# 表示 Outline 使用的端口，请放心设置，因为不会占用宿主机的端口位置，仅仅作用于容器内的端口
PORT=3000
# =========================================


# ================ PostgreSQL ================
# 用于连接 PostgreSQL 的 URL
# 修改项:
# 1. password_of_outline_user: 是你当时部署 PostgreSQL 后，进入容器创建的 outline 用户的密码
DATABASE_URL=postgres://outline:password_of_outline_user@postgres12:5432/outline
DATABASE_URL_TEST=postgres://outline:password_of_outline_user@postgres12:5432/outline_test
# PostgreSQL 数据库开启 SSL 链路加密后，表示允许客户端通过 SSL 连接数据库
# 由于我们没有反代 PostgreSQL，也没有为 PostgreSQL 配置 SSL，仅仅在局域网内访问
# 因此需要将此环境变量设为 disable
PGSSLMODE=disable
# ============================================


# ================ Redis ================
# 用于连接 Redis 的 URL
# 修改项:
# 1. password_of_redis: 是当时部署 Redis 时，位于 redis-docker-compose.yaml 中的密码
REDIS_URL=redis://:password_of_redis@redis:6379
# =======================================


# ================ MinIO ================
# 这是我们 MinIO 管理界面新添加的 User 的 Access Key 字段
AWS_ACCESS_KEY_ID=access_key_of_minio_outline
# 这是我们 MinIO 管理界面新添加的 User 的 Secret Key 字段
AWS_SECRET_ACCESS_KEY=secret_key_of_minio_outline
# 位于 minio-docker-compose.yaml 的字段
AWS_REGION=cn-outline-1
# 通过 Nginx 反代后的地址
# 修改项: 
# 1. minio.example.com: 修改为你自己的域名
AWS_S3_UPLOAD_BUCKET_URL=https://minio.example.com
# 这是我们在 MinIO 管理界面新添加的 Bucket 的名称
AWS_S3_UPLOAD_BUCKET_NAME=outline
# 我们添加的 Bucket 默认是 private 的
AWS_S3_ACL=private

# 一些额外的配置项，更多的可以看文档，来更好地满足自己需求
AWS_S3_UPLOAD_MAX_SIZE=26214400
# 默认情况下，许多 S3 客户端库会使用子域名样式。然而，有时这可能会导致问题，特别是当存储桶名称包含点"."或不符合DNS子域名规范时
# 设置 AWS_S3_FORCE_PATH_STYLE 为 true 会强制客户端库使用路径样式，而不是子域名样式
AWS_S3_FORCE_PATH_STYLE=true
# =======================================

# ––––––––––––––––------------––––––––––––––




# –––––––––––––– AUTHENTICATION ––––––––––––––
# Outline 至少需要配置一个授权方式，我们在之前部署了 Keycloak 可以作为我们的 OIDC 服务

# ============== Keycloak OIDC ==============
# 这是我们在 Keycloak 管理界面新添加 Client 时，设置的 Client 名称
OIDC_CLIENT_ID=outline
# 这是我们在 Outline Client 设置界面，Credentials 选项卡中的 Secret 字段
# 你可以看本文的 `5.3.4 配置 Outline Client` 来查看图片快速找到位置
OIDC_CLIENT_SECRET=secret_of_keycloak_outline_client
# 以下是 Outline 用于完成 OIDC 验证的 API 接口，其余地方均无需修改，只需要修改域名
# 修改项
# 1. sso.example.com: 修改为你自己的域名
OIDC_AUTH_URI=https://sso.example.com/auth/realms/outline/protocol/openid-connect/auth
OIDC_TOKEN_URI=https://sso.example.com/auth/realms/outline/protocol/openid-connect/token
OIDC_USERINFO_URI=https://sso.example.com/auth/realms/outline/protocol/openid-connect/userinfo
# 这个名字将展示在 https://outline.example.com 的首页，登陆选项中将会显示 “使用 Keycloak 继续”
OIDC_DISPLAY_NAME=Keycloak
# 默认是 openid profile email，表示允许用户使用什么方式登录
# openid: 我们 Keycloak 设定的 openid 是一个 UUID，因为太长了所以用来登陆也不合适
# profile: 就是我们的账户名称 Username，注意不是 First/Last Name 字段
# email: 就是我们创建账户是，可以填入的 email 字段
OIDC_SCOPES=profile email
# ===========================================

# ––––––––––––––-----------------–––––––––––––
```



## 7.4 初始化数据库

我们在之前已经在 PostgreSQL 中创建好了 outline 和 outline-test 的数据库

如果没有创建，你可以使用以下命令快速创建

```bash
# 确保进入到 "outline-docker-compose.yaml" 同级目录
cd /home/docker-compose/outline
# 创建 outline 和 outline-test 数据库
docker-compose run \
    -f outline-docker-compose.yml \
    --rm outline \
    yarn db:create \
    --env=production-ssl-disabled
```

除了创建数据库，还需要创建表和索引

```bash
# 确保进入到 "outline-docker-compose.yaml" 同级目录
cd /home/docker-compose/outline
# 初始化数据库: 添加表和索引
docker-compose run \
    -f outline-docker-compose.yml \
    --rm outline \
    yarn db:migrate \
    --env=production-ssl-disabled
```



## 7.5 使用 Docker-Compose 部署 Outline

在此之前确保已经创建好了 `outline-docker.env`, `outline-docker-compose.yaml`，也创建好了 `outline`, `outline-test` 数据库，并且也创建好了表和索引。

```bash
# 确保进入到 "outline-docker-compose.yaml" 同级目录
cd /home/docker-compose/outline
# 使用 docker-compose 启动 outline
docker-compose -f outline-docker-compose.yaml up -d
```



# 8. 初步测试

## 8.1 登陆 Outline

打开 `outline.example.com`，再使用 Keycloak 继续

![8-use-outline-01](./assets/8-use-outline-01.png)

之后将跳转到 `sso.example.com`，完成登陆后将自动跳转到 Outline 主界面。

这里的用户名和密码都是我们在 Keycloak 中定义的

![8-use-outline-02](./assets/8-use-outline-02.png)



## 8.2 测试 PostgreSQL 基本功能

在主界面左侧新建文档，如果能够创建，基本上说明 PostgreSQL 是能够正常运行的

![8-use-outline-03](./assets/8-use-outline-03.png)





## 8.3 测试 MinIO 基本功能

在新建的文档内添加一个图片，该图片将上传到 MinIO 的 Outline Bucket 中

![8-use-outline-04-upload-pic](./assets/8-use-outline-04-upload-pic.png)

进一步的，你还可以登陆 `minio-admin.example.com` 或者 `minio.example.com` (自动跳转至 admin 域名)，登陆后你将看到。如果你的 Total Objects 为 0，请等待一段时间或者强制刷新浏览器（清除缓存）

还有就是如果你在一个文章中上传多个图片，实际上会包裹进用一个对象中去，Total Object 不等于上传的图片数量（具体的规则我没有细读官方文档，这是我观察后的想法，如有问题，欢迎纠错）

另外你还可以进入 Object Browser 来查看 Outline Bucket 中有哪些文件，具体使用可以参考左侧栏目的最后一栏有一个 [Documentation](https://min.io/docs/minio/kubernetes/upstream/index.html?ref=docs-redirect&ref=con)

![8-use-outline-04-minio-display](./assets/8-use-outline-04-minio-display.png)



## 8.4 更多功能

更多功能还待自己去尝试，这里就不剧透了。

如此界面优美、功能齐全、多人实时协同、权限管理、可私人部署的开源 Wiki 项目，我不允许大家不知道！希望 Outline 能拥有更多的用户，变得更好~



# 9. 参考资料

他人分享的部署教程系列：

- Outline 搭建：https://biteax.com/b9dce220.html
- Outline 搭建：https://github.com/soulteary/docker-outline
- Outline 搭建：https://zhuanlan.zhihu.com/p/411062890
- Keycloak 搭建：https://gitee.com/itmuch/spring-cloud-yes/tree/master/doc/keycloak-learn
- Keycloak+Outline 搭建：https://blog.yarsalabs.com/self-hosting-outline-wiki-on-cloudron/



官方文档系列：

- Outline：https://docs.getoutline.com/s/hosting/doc/docker-7pfeLP5a8t
- Nginx：https://nginx.org/en/docs/
- Keycloak：https://www.keycloak.org/documentation.html
- Keycloak：https://keycloak.org.cn/docs/latest/getting_started/index.html
- MinIO for Docker：https://min.io/docs/minio/container/index.html
- PostgreSQL 12: https://www.postgresql.org/docs/12/index.html
- Redis: https://redis.io/docs/



Github Issue 系列:

- Outline+KeyCloak出现的问题：https://github.com/outline/outline/discussions/2716



配置文件的要点系列：

- 使用 Nginx 反代 Outline: [Outline/Configuration/Nginx](https://docs.getoutline.com/s/hosting/doc/nginx-6htaRboR57)
- 使用 Nginx 反代 Minio:  [Integrations/Configure NGINX Proxy for MinIO Server](http://minio.org.cn/docs/minio/linux/integrations/setup-nginx-proxy-with-minio.html)。
- 使用 Nginx 反代 Keycloak: [Guides/Server/Using a reverse proxy](https://www.keycloak.org/server/reverseproxy)




一些字段和细节：

- 获取OIDC_CLIENT_SECRET：https://blog.csdn.net/MRLEE1212/article/details/103902379
- Nginx 反代 Outline注意事项：https://docs.getoutline.com/s/hosting/doc/nginx-6htaRboR57



# X. Change Log

---

- v1.0.0：2023年08月13日 22:03:24
  - 完成第一版文档


---

---

本文于2023年08月13日完成，如有问题欢迎联系我😊: 

- emailtojiang@gmail.com
- emailtojiang@163.com

---

---

<div style="text-align: center; margin-bottom: 100px; margin-top: 100px; font-family: 'Courier New', Courier, monospace; font-size: 24px; color: #4A90E2; padding-top: 20px;">
  <span style="background-color: #F2F2F2; padding: 10px; border-radius: 5px;">✨ The End ✨</span>
</div>


