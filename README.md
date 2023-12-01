# Overview

Outline 宣传图

![0-outline-updated](docs/assets/0-outline-updated.png)

本文介绍了一种基于 Nginx All In One 的网络架构部署 Outline 的方法，目录结构如下：

```
0. 前言
    I. 需求
    II. Outline 简介
    III. 环境说明
    IV. Contact
1. 架构图
2. Nginx 部署
    2.1 概述
    2.2 配置域名
        2.2.1 需要了解的
        2.2.2 服务器域名配置
    2.3 获取 SSL 证书
    2.4 使用 Docker&Compose 部署 Nginx 与网络
        2.4.1 创建 Docker Network
        2.4.2 创建 Nginx
    2.5 配置 Nginx 反代 & SSL 证书
        2.5.1 安装 SSL 证书
        2.5.2 配置反向代理规则
        2.5.3 导入到 Nginx 配置
        2.5.4 启用最新配置
3. PostgreSQL 部署
    3.1 概述
    3.2 已有 PostgreSQL 服务
    3.3 使用 Docker-Compose 部署 PostgreSQL
    3.4 创建用户与数据库
4. Redis 部署
    4.1 概述
    4.2 使用 Docker-Compose 部署 Redis
5. Keycloak 部署
    5.1 概述
    5.2 使用 Docker-Compose 部署 Keycloak
    5.3 配置 Outline 的 OIDC 服务
        5.3.1 登陆
        5.3.2 创建 Outline Realm
        5.3.3 创建 Outline Client
        5.3.4 配置 Outline Client
        5.3.5 创建 Outline Client Role
        5.3.6 创建 Outline User
        5.3.7 授权 Outline User 访问 Outline Client 权限
6. MinIO 部署
    6.1 概述
    6.2 使用 Docker-Compose 部署 MinIO
    6.3 配置 Outline 的 OSS 服务
7. Outline 部署
    7.1 概述
    7.2 创建 Outline 配置文件
    7.3 outline-docker.env 配置文件详解
    7.4 初始化数据库
    7.5 使用 Docker-Compose 部署 Outline
8. 初步测试
    8.1 登陆 Outline
    8.2 测试 PostgreSQL 基本功能
    8.3 测试 MinIO 基本功能
    8.4 更多功能
9. 问题汇总
    9.1 Outline 更新方法
    9.2 Outline 是否有桌面端？
        9.2.1 PWA 解决方案
        9.2.2 官方解决方案
10. 参考资料
Change Log
```

你可以访问在线文档查看

- GitHub Page：[https://jiang-taibai.github.io/deploy-outline-via-nginx](https://jiang-taibai.github.io/deploy-outline-via-nginx)
- Gitee Page: [https://jiang-taibai.gitee.io/deploy-outline-via-nginx](https://jiang-taibai.gitee.io/deploy-outline-via-nginx)

# Architecture Diagram

![Outline 架构图](docs/assets/1-architecture-diagram-v4.png)

涉及到的Outline服务以及它的依赖服务:

- **Outline**: 超级棒的团队多人协同文档管理开源项目！3000 端口为访问 Outline 的端口，但该端口并不暴露，由 Nginx 通过 Docker Network 方式访问
- **Keycloak**: 一个支持 OpenID Connect（下文简称 **OIDC**）的开源项目。用于 Outline 的单点登录服务。
- **Redis**: 非关系型数据库，Outline 使用 Redis 实现缓存、消息队列、会话存储、实时协作等功能
- **PostgreSQL**: 关系型数据库，Outline 使用 PostgreSQL 实现数据的持久化
- **MinIO**: 一款本地**对象存储系统**的开源项目。用于存储 Outline 的图片等资源

网络架构主干:

- **nginx_all_in_one**: Docker Network，使用虚拟网卡实现多个容器之间的网络互通
- **Nginx**: 占用主机的 80, 443 端口并反代了四个域名，而反代的端口确实来自虚拟局域网中的端口，由图中可知整个网络只有 80 和 443 端口暴露在外。

四个域名的作用:

- `outline.example.com`: 用于访问 Outline 的主域名
- `sso.example.com`: 提供身份权限验证服务，同时也是管理员入口
- `minio.example.com`: MinIO API 为 Outline 提供对象存储服务 OSS
- `minio-admin.example.com`: MinIO Admin 界面

这种部署方式的优点如下:

- **安全性更高**🔒: 所有服务均隐藏与虚拟局域网中，并在虚拟局域网中通信，不会暴露在互联网上
- **门槛低**🎁: 全文所有服务项均使用 Docker 部署，并使用 Compose 插件，俗称开箱即用，部署难度大大降低。通常情况下，你只需要新建一个 `yaml` 配置文件和执行一条命令 `docker-compose up -d` 即可。
- **非侵入式的端口友好型**🚪: 所有服务都不会占用服务器的任何一个端口。由占用 80 端口的 Nginx 负责反向代理转发到 虚拟局域网 中的服务访问点 SAP
- **非侵入式的环境友好型**🐳: 所有服务均使用 Docker 容器化部署，不会在服务器中创建一大堆的环境变量
- **后期调试友好型**🔧: 所有服务均使用 Docker Compose 插件部署，所有配置项、密码等都保存在了服务器中，较原先的纯 Docker 部署而言，对于后期调试无需翻找当时设置的所有配置项。

# Friendly Link

本文档使用了以下技术和服务：

- **Outline**: [官方网站](https://www.getoutline.com/)
- **Docker**: [官方网站](https://www.docker.com/)
- **Docker Compose**: [官方网站](https://docs.docker.com/compose/)
- **Nginx**: [官方网站](https://nginx.org/)
- **Redis**: [官方网站](https://redis.io/)
- **PostgreSQL**: [官方网站](https://www.postgresql.org/)
- **MinIO**: [官方网站](https://min.io/)
- **Keycloak**: [官方网站](https://www.keycloak.org/)
- **Docsify**: [官方网站](https://docsify.js.org/#/)

特别感谢这些项目的贡献者们！

# Change Log

---

- v1.1.0：2023年12月01日 11:16:02
  - `Outline > 0.72.0` 后以下字段为出现时不可为空，因此注释即可（已在文档中做出相应更改）
    ```properties
    # Iframely API config
    # IFRAMELY_URL=
    # IFRAMELY_API_KEY=
    ```
  - 引导用户在 Issue 提问
  - 添加问题咨询 “Outline 升级的办法”
  - 添加问题咨询 “Outline 是否有桌面端”
- v1.0.0：2023年08月13日 22:03:24
  - 完成第一版文档

---

# Contact

本文于2023年08月13日完成，最后更新时间2023年12月01日

如有问题欢迎在 Gitee 或 GitHub 上提 Issue 😊:

Gitee Issue：https://gitee.com/jiang-taibai/deploy-outline-via-nginx/issues

GitHub Issue：https://github.com/jiang-taibai/deploy-outline-via-nginx/issues


# License

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。

Copyright (c) 2023, Jiang Liu
