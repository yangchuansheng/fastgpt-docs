---
weight: 500
title: "开发指南"
description: "对 FastGPT 进行开发调试"
icon: "developer_guide"
draft: false
images: []
---

第一次开发，需要先部署数据库，建议本地开发可以随便找一台 2C2G 的轻量小数据库实践。数据库部署教程：[Docker 快速部署](/docs/installation/docker/)

client 目录下为 FastGPT 核心代码。NextJS 框架前后端放在一起，API 服务位于 `src/pages/api` 目录内。

## 初始配置

**1. 环境变量**

复制.env.template 文件，生成一个.env.local 环境变量文件夹，修改.env.local 里内容才是有效的变量。变量说明见 .env.template

**2. config 配置文件**

复制 data/config.json 文件，生成一个 data/config.local.json 配置文件。

这个文件大部分时候不需要修改。只需要关注 SystemParams 里的参数：

+ `vectorMaxProcess`: 向量生成最大进程，根据数据库和 key 的并发数来决定，通常单个 120 号，2c4g 服务器设置10~15。
+ `qaMaxProcess`: QA 生成最大进程
+ `pgIvfflatProbe`: PostgreSQL vector 搜索探针，没有添加 vector 索引时可忽略。

## 运行

```bash
cd client
pnpm i
pnpm dev
```

## 镜像打包

```bash
docker build -t dockername/fastgpt .
```