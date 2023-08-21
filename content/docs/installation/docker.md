---
title: "Docker Compose 快速部署"
description: "使用 Docker Compose 快速部署 FastGPT"
icon: ""
draft: false
toc: true
weight: 720
---

## 准备条件

### 1. 准备好代理环境（国外服务器可忽略）

确保可以访问 OpenAI，具体方案可以参考：[Nginx 中转](/docs/installation/proxy/nginx/)

### 2. 多模型支持

推荐使用 one-api 项目来管理模型池，兼容 OpenAI 、Azure 和国内主流模型等。

具体部署方法可参考该项目的 [README](https://github.com/songquanpeng/one-api)，也可以直接通过以下按钮一键部署：

[![](https://cdn.jsdelivr.us/gh/labring-actions/templates@main/Deploy-on-Sealos.svg)](https://cloud.sealos.io/?openapp=system-fastdeploy%3FtemplateName%3Done-api)

## 安装 Docker 和 docker-compose

{{< tabs tabTotal="3" >}}
{{< tab tabName="Linux" >}}
{{< markdownify >}}
```bash
# 安装 Docker
curl -sSL https://get.daocloud.io/docker | sh
systemctl enable --now docker
# 安装 docker-compose
curl -L https://github.com/docker/compose/releases/download/2.20.3/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
# 验证安装
docker -v
docker-compose -v
```
{{< /markdownify >}}
{{< /tab >}}
{{< tab tabName="MacOS" >}}
{{< markdownify >}}
推荐直接使用 [Orbstack](https://orbstack.dev/)。可直接通过 Homebrew 来安装：

```bash
brew install orbstack
```

或者直接[下载安装包](https://orbstack.dev/download)进行安装。
{{< /markdownify >}}
 {{< /tab >}}
{{< tab tabName="Windows" >}}
{{< markdownify >}}
我们建议将源代码和其他数据绑定到 Linux 容器中时，将其存储在 Linux 文件系统中，而不是 Windows 文件系统中。

可以选择直接[使用 WSL 2 后端在 Windows 中安装 Docker Desktop](https://docs.docker.com/desktop/wsl/)。

也可以直接[在 WSL 2 中安装命令行版本的 Docker](https://nickjanetakis.com/blog/install-docker-in-wsl-2-without-docker-desktop)。
{{< /markdownify >}}
{{< /tab >}}
{{< /tabs >}}

## 创建 docker-compose.yml 文件

先创建一个目录（例如 fastgpt）并进入该目录：

```bash
mkdir fastgpt
cd fastgpt
```

创建一个 docker-compose.yml 文件，粘贴下面的内容：

```yaml
# 非 host 版本, 不使用本机代理
version: '3.3'
services:
  pg:
    image: ankane/pgvector:v0.4.2
    # image: registry.cn-hangzhou.aliyuncs.com/fastgpt/pgvector:v0.4.2 # 阿里云
    container_name: pg
    restart: always
    ports: # 生产环境建议不要暴露
      - 5432:5432
    networks:
      - fastgpt
    environment:
      # 这里的配置只有首次运行生效。修改后，重启镜像是不会生效的。需要把持久化数据删除再重启，才有效果
      - POSTGRES_USER=username
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=postgres
    volumes:
      - ./pg/data:/var/lib/postgresql/data
  mongo:
    image: mongo:5.0.18
    # image: registry.cn-hangzhou.aliyuncs.com/fastgpt/mongo:5.0.18 # 阿里云
    container_name: mongo
    restart: always
    ports: # 生产环境建议不要暴露
      - 27017:27017
    networks:
      - fastgpt
    environment:
      # 这里的配置只有首次运行生效。修改后，重启镜像是不会生效的。需要把持久化数据删除再重启，才有效果
      - MONGO_INITDB_ROOT_USERNAME=username
      - MONGO_INITDB_ROOT_PASSWORD=password
    volumes:
      - ./mongo/data:/data/db
  fastgpt:
    container_name: fastgpt
    image: ghcr.io/labring/fastgpt:latest # git
    # image: registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt:latest # 阿里云
    ports:
      - 3000:3000
    networks:
      - fastgpt
    depends_on:
      - mongo
      - pg
    restart: always
    environment:
      # root 密码，用户名为: root
      - DEFAULT_ROOT_PSW=1234
      # 中转地址，如果是用官方号，不需要管
      - OPENAI_BASE_URL=https://api.openai.com/v1
      - CHAT_API_KEY=sk-xxxx
      - DB_MAX_LINK=5 # database max link
      - TOKEN_KEY=any
      - ROOT_KEY=root_key
      # mongo 配置，不需要改
      - MONGODB_URI=mongodb://username:password@mongo:27017/?authSource=admin
      - MONGODB_NAME=fastgpt
      # pg配置
      - PG_HOST=pg
      - PG_PORT=5432
      - PG_USER=username
      - PG_PASSWORD=password
      - PG_DB_NAME=postgres
networks:
  fastgpt:
```

> 只需要改 fastgpt 容器的 3 个参数即可启动。

## 启动容器

```bash
# 在 docker-compose.yml 同级目录下执行
docker-compose up -d
```

## 访问 FastGPT

目前可以通过 `ip:3000`` 直接访问(注意防火墙)。登录用户名为 `root`，密码为刚刚环境变量里设置的 `DEFAULT_ROOT_PSW`。

如果需要域名访问，请自行安装并配置 Nginx。

## QA

### 如何更新？

执行 `docker-compose up -d` 会自动拉取最新镜像，一般情况下不需要执行额外操作。

### 如何自定义配置文件？

需要在 `docker-compose.yml` 同级目录创建一个 `config.json` 文件，内容如下：

```json
{
  "FeConfig": {
    "show_emptyChat": true,
    "show_register": false,
    "show_appStore": false,
    "show_userDetail": false,
    "show_git": true,
    "systemTitle": "FastGPT",
    "authorText": "Made by FastGPT Team.",
    "gitLoginKey": "",
    "scripts": []
  },
  "SystemParams": {
    "gitLoginSecret": "",
    "vectorMaxProcess": 15,
    "qaMaxProcess": 15,
    "pgIvfflatProbe": 20
  },
  "plugins": {},
  "ChatModels": [
    {
      "model": "gpt-3.5-turbo",
      "name": "GPT35-4k",
      "contextMaxToken": 4000,
      "quoteMaxToken": 2000,
      "maxTemperature": 1.2,
      "price": 0,
      "defaultSystem": ""
    },
    {
      "model": "gpt-3.5-turbo-16k",
      "name": "GPT35-16k",
      "contextMaxToken": 16000,
      "quoteMaxToken": 8000,
      "maxTemperature": 1.2,
      "price": 0,
      "defaultSystem": ""
    },
    {
      "model": "gpt-4",
      "name": "GPT4-8k",
      "contextMaxToken": 8000,
      "quoteMaxToken": 4000,
      "maxTemperature": 1.2,
      "price": 0,
      "defaultSystem": ""
    }
  ],
  "QAModels": [
    {
      "model": "gpt-3.5-turbo-16k",
      "name": "GPT35-16k",
      "maxToken": 16000,
      "price": 0
    }
  ],
  "VectorModels": [
    {
      "model": "text-embedding-ada-002",
      "name": "Embedding-2",
      "price": 0
    }
  ]
}
```

然后修改 `docker-compose.yml` 中的 `fastgpt` 容器内容，增加挂载选项即可：

```yaml
fastgpt:
  container_name: fastgpt
    image: ghcr.io/labring/fastgpt:latest # github
    # image: registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt:latest # 阿里云
  ports:
    - 3000:3000
  networks:
    - fastgpt
  depends_on:
    - mongo
    - pg
  restart: always
  environment:
    # root 密码，用户名为: root
    - DEFAULT_ROOT_PSW=1234
  volumes:
    - ./config.json:/app/data/config.json
```

> 参考[配置详解](/docs/installation/reference/configuration/)