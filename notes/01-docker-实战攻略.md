# 01 Docker 实战攻略（B站 · 技术爬爬虾）

> 配套视频：**[40分钟的Docker实战攻略，一期视频精通Docker](https://www.bilibili.com/video/BV1THKyzBER6/)**（约 40 分钟）
> UP 主：B 站 **@技术爬爬虾** ｜ 主线：核心概念 → 安装 → 拉镜像 → run 容器 → 挂载卷 → 调试 → Dockerfile → 网络 → Compose
> 本章命令均在 **Ubuntu 22.04 + Docker 3.3.6** 环境下整理；视频里的语音识别错别字（"速度"=sudo、"N减X"=nginx 等）已统一修正。

| # | 视频章节 | 时间点 | 对应小节 |
|---|---------|--------|---------|
| 1 | 核心概念 | 00:00 | [1.1](#11-核心概念0000) |
| 2 | 安装 | 01:20 | [1.2](#12-安装0120) |
| 3 | 下载镜像 | 04:09 | [1.3](#13-下载镜像-docker-pull0409) |
| 4 | 运行容器 | 09:33 | [1.4](#14-运行容器-docker-run0933) |
| 5 | 挂载卷 | 13:01 | [1.5](#15-挂载卷-v1301) |
| 6 | run 其他参数 | 16:56 | [1.6](#16-run-的其他参数1656) |
| 7 | 调试容器 | 19:34 | [1.7](#17-调试容器1934) |
| 8 | 构建镜像 | 24:45 | [1.8](#18-构建镜像-dockerfile2445) |
| 9 | Docker 网络 | 29:16 | [1.9](#19-docker-网络2916) |
| 10 | Docker Compose | 33:31 | [1.10](#110-docker-compose3331) |

---

## 1.1 核心概念（00:00）

**一句话：Docker = 用容器化技术给应用封装独立的运行环境，比虚拟机更轻、启动更快。**

| 概念 | 大白话 | 类比 |
|------|--------|------|
| 宿主机 | 运行容器的那台计算机 | 厨房 |
| 镜像（Image） | 容器的模板，只读 | 模具 |
| 容器（Container） | 用镜像跑起来的实例 | 糕点 |
| 仓库（Registry） | 存放/分享镜像的地方，官方的是 Docker Hub | 分享模具的货架 |

- **容器 vs 虚拟机**：多个容器**共用宿主机的系统内核**；每个虚拟机都要装一整个操作系统 → 容器更轻更小、启动秒级。
- **底层原理**（视频在调试章节补充）：Docker 基于 Linux 内核两大机制——
  - **cgroups**：限制和隔离进程的资源（CPU/内存/带宽上限），一个容器抢资源不会拖垮别人；
  - **namespaces**：隔离进程的资源视图，容器只能看到自己内部的进程 ID、网络和文件目录。
  - 所以**容器本质上就是一个特殊的进程**，只是从里面看像一个独立的 Linux 系统。

---

## 1.2 安装（01:20）

Docker 基于 Linux 容器化技术，Windows/Mac 都是虚拟了一个 Linux 子系统来跑，**命令行在各系统间完全一致**，通用性最强。

| 系统 | 安装要点 |
|------|---------|
| Linux（Ubuntu） | 官网 get.docker.com 执行第 1 步 + 第 4 步两条脚本命令；非 root 用户所有 docker 命令前要加 `sudo` |
| Windows | 「启用或关闭 Windows 功能」勾选 **虚拟机平台** + **WSL** → 重启 → 管理员 CMD：`wsl --set-default-version 2`、`wsl --update`（国内网络可加 `--web-download`）→ 下载 Docker Desktop（一般选 AMD64）一路下一步 |
| Mac | 按芯片（Apple Silicon / Intel）下载对应 Docker Desktop 安装包，最简单 |

验证安装：

```bash
docker --version    # 能打印版本号即成功
```

---

## 1.3 下载镜像：docker pull（04:09）

### 镜像名的完整结构

```
docker.io / library / nginx : latest
└─ registry 注册表地址 └─ 命名空间 └─ 镜像名 └─ 标签(版本)
```

| 部分 | 说明 | 能否省略 |
|------|------|---------|
| Registry（如 `docker.io`） | 仓库地址，docker.io 是 Docker Hub 官方 | 官方可省略 |
| 命名空间（如 `library`） | 作者名，防止大家镜像重名 | 官方命名空间可省略 |
| 镜像名 | 如 `nginx` | 不能省 |
| 标签（如 `latest`） | 版本号 | 不写默认 latest |

**Registry + 命名空间 + 镜像名 = repository（镜像库）**，一个镜像库存放同一镜像的不同版本；整个 Docker Hub 网站是一个 registry。

### 常用命令

```bash
docker pull nginx              # 拉官方最新版 nginx
docker pull nginx:1.28.0       # 拉指定版本
docker images                  # 列出所有下载过的镜像
docker rmi <镜像名或ID>         # 删除镜像（rm + image）
docker pull --platform=arm64 xxx   # 拉特定 CPU 架构的镜像
```

### 国内网络问题 → 配置镜像站

- Linux：修改 `/etc/docker/daemon.json` 加入 `registry-mirrors`，然后 `systemctl restart docker` 重启；
- Windows/Mac：Docker Desktop → Settings → Docker Engine，在 JSON 里同样加 `registry-mirrors` → Apply and Restart；
- UP 主的安装/镜像站文档：https://github.com/tech-shrimp/docker_installer

### CPU 架构注意

- 默认 `docker pull` 自动选最适合当前宿主机架构的镜像；
- ARM 小主机（如香橙派 arm64）要先去 Docker Hub 确认镜像有没有 arm64 版本（如 RAGFlow 只有 AMD64 就跑不了）；
- Mac（arm64）的 Docker Desktop 会用 **QEMU 模拟 x86**，能兼容部分 AMD64 镜像，但有兼容性/性能开销。

---

## 1.4 运行容器：docker run（09:33）

**本期视频最重要的命令**：用镜像（模具）创建并运行容器（糕点）。

```bash
docker run nginx               # 本地没有镜像会自动先 pull 再运行
docker run -d nginx            # detached 后台运行，不阻塞终端
docker ps                      # 查看正在运行的容器（ps = process status）
docker ps -a                   # 查看所有容器（含已停止的）
docker stop <容器ID/名字>       # 停止容器
docker start <容器ID/名字>      # 重新启动（之前的参数都记住了，不用重写）
docker rm -f <容器ID/名字>      # 删除容器（运行中的要加 -f）
docker create nginx            # 只创建不启动，之后用 docker start 启动
docker logs -f <容器>           # 查看容器日志，-f 滚动刷新
```

### 端口映射 `-p`（先外后内）

容器网络与宿主机是**隔离**的，直接访问不到容器内的服务，需要 `-p` 把宿主机端口转发给容器：

```bash
docker run -d -p 80:80 nginx   # 宿主机80端口 → 容器80端口
# 浏览器访问 localhost:80 就等于访问容器内的 nginx
```

### 环境变量 `-e`

把账号密码等配置传进容器，比如 MongoDB：

```bash
docker run -d -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=123456 \
  mongo
```

有哪些环境变量可以传？去 **Docker Hub 镜像页文档**或项目 **GitHub 仓库**里查。

---

## 1.5 挂载卷 `-v`（13:01）

**作用：数据持久化。** 删除容器时容器内所有数据会一起没了；挂载卷把宿主机目录和容器目录绑定，数据存在宿主机上就删不掉了。

| 方式 | 写法 | 特点 |
|------|------|------|
| 绑定挂载 | `-v /宿主机路径:/容器路径` | 直接指定目录；**宿主机空目录会覆盖容器内目录**（网页变 403）；无初始化同步 |
| 命名卷挂载 | `-v 卷名:/容器路径` | Docker 管理存储空间；**首次使用自动把容器内文件同步过来做初始化** |

```bash
docker volume create nginxhtml          # 创建命名卷
docker volume inspect nginxhtml         # 查看卷的真实存放路径（需 root 进入）
docker volume list                      # 列出所有卷
docker volume rm <卷名>                  # 删除一个卷
docker volume prune -a                  # 删除所有未被容器使用的卷
```

---

## 1.6 run 的其他参数（16:56）

| 参数 | 作用 | 说明 |
|------|------|------|
| `--name xxx` | 给容器起自定义名字 | 宿主机内必须唯一；之后用名字就能 stop/rm，比 ID 好记 |
| `-it` | 进入容器的交互式控制台 | 常配合 `--rm` 临时调试用 |
| `--rm` | 容器一停止就自动删除 | `-it --rm alpine` = 进去玩完就走，不留垃圾 |
| `--restart=always` | 只要容器停了就立刻重启 | 含崩溃、宿主机断电重启等场景 |
| `--restart=unless-stopped` | 同上，但**手动 stop 的不再自动拉起** | 生产环境更常用 |
| `--network xxx` | 指定加入哪个网络/子网 | 见 [1.9](#19-docker-网络2916) |

> 注意：每次 `docker run` 都会创建一个**全新**的容器；只是想启停原有容器用 `docker stop` / `docker start`。

---

## 1.7 调试容器（19:34）

```bash
docker exec <容器> ps -ef         # 在容器内执行 Linux 命令（看进程）
docker exec -it <容器> /bin/bash  # 进入容器内部，拿到交互式终端
docker inspect <容器>             # 查看容器全部详细配置（端口映射/挂载卷/环境变量）
```

- 忘了容器当初 run 时填了什么参数？`docker inspect` 一把梭，输出太长可以直接**贴给 AI 帮你分析**。
- 容器内是极简系统，很多工具缺失（比如没有 vi）。先 `cat /etc/os-release` 看发行版（如 Debian），再 `apt update && apt install vim` 临时装。

---

## 1.8 构建镜像：Dockerfile（24:45）

**类比升级：镜像=模具，容器=糕点，Dockerfile=制作模具的图纸。**

以一个 FastAPI 的 Python 程序为例，本地跑起来要三步：装 Python → `pip install -r requirements.txt` → `python main.py`。Dockerfile 把这三步写进图纸：

```dockerfile
FROM python:3.13-slim        # 基础镜像（自带 Python 的精简 Debian）
WORKDIR /app                 # 设置工作目录，后续命令都在这里执行
COPY . .                     # 把本机当前目录的代码拷进镜像工作目录（点 空格 点）
RUN pip install -r requirements.txt   # 构建镜像时执行的命令（装依赖）
EXPOSE 8000                  # 声明服务端口（只是提示，不强制，实际以 -p 为准）
CMD ["python", "main.py"]    # 容器启动时默认执行的命令，建议数组写法，只能有一个
```

| 指令 | 作用 | 易错点 |
|------|------|--------|
| `FROM` | 选基础镜像 | 第一行必须 |
| `WORKDIR` | 切换镜像内工作目录 | 类似 cd |
| `COPY` | 拷贝文件进镜像 | 两个点的含义：本机目录 → 镜像目录 |
| `RUN` | **构建时**执行 | 装依赖用这个 |
| `EXPOSE` | 声明端口 | 可省略 |
| `CMD` | **容器运行时**的默认命令 | 只能一个；优先级低于 ENTRYPOINT |
| `ENTRYPOINT` | 同 CMD 但优先级更高 | 不容易被 run 后面的命令覆盖 |

文件名必须叫 `Dockerfile`（D 大写、无后缀名）。

### 构建、运行、推送

```bash
docker build -t docker-test:1.0 .        # 在当前目录构建，-t 起名:版本，最后的点指当前目录
docker run -d -p 8000:8000 docker-test   # 用自己的镜像跑容器

docker login                              # 登录 Docker Hub（设备验证码流程）
docker build -t 用户名/docker-test:1.0 .   # 推送前镜像名必须带上自己的用户名（命名空间）
docker push 用户名/docker-test:1.0        # 推送到 Docker Hub，别人就能 docker pull 了
```

---

## 1.9 Docker 网络（29:16）

`docker network list` 能看到三种**默认网络**（不可删除）：bridge、host、none；`docker network remove` 只能删自定义子网。

| 模式 | 特点 | 典型用途 |
|------|------|---------|
| **bridge**（默认） | 容器分配独立内网 IP（172.17 开头），容器间可用 IP 互访；与宿主机网络隔离 | 一般场景 |
| **bridge + 自定义子网** | 同子网容器**可以用容器名互相访问**（内置 DNS 把名字解析成 IP），跨子网不通；容器间互通无需 `-p` | 多容器协作，如 web 连数据库 |
| **host** | 直接共享宿主机网络，**无需 -p 端口映射**，服务直接跑在宿主机端口上 | 解决棘手的网络需求 |
| **none** | 完全断网 | 安全隔离场景 |

子网实战（MongoDB + 网页客户端 mongo-express）：

```bash
docker network create network1              # 创建子网
docker run -d --name mongo --network network1 mongo        # 数据库不映射端口，仅子网内可达
docker run -d --name mongo-express --network network1 \
  -p 8081:8081 \
  -e ME_CONFIG_MONGODB_SERVER=mongo \       # 只写容器名，不写 IP，靠子网 DNS 解析
  mongo-express
# 浏览器访问 服务器IP:8081 → mongo-express → 通过子网内部访问 MongoDB
```

---

## 1.10 Docker Compose（33:31）

**解决什么问题**：一个应用 = 前端 + 后端 + 数据库多个容器。都塞一个大容器 → 一崩全崩、没法精准扩容；分开多个容器 → 要手动 run 多次、配网络、容易错。**Compose 用一个 YML 文件把这些容器（服务）集中管理。**

> 可以把 compose 文件理解成：把多条 docker run 命令按格式写进一个文件。docker 还会自动为每个 compose 文件创建一个子网，里面的服务自动加入，不用手动配网络。

```yaml
services:
  mongo:
    image: mongo
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
    volumes:
      - mongo-data:/data/db
  mongo-express:
    image: mongo-express
    ports:
      - "8081:8081"                    # -p 的等价写法
    environment:
      ME_CONFIG_MONGODB_SERVER: mongo  # -e 的等价写法
    depends_on:                        # 先启动 mongo，再启动 mongo-express
      - mongo
volumes:
  mongo-data:
```

| run 命令写法 | compose 文件写法 |
|-------------|-----------------|
| `--name xxx` | service 名 |
| 镜像名 | `image:` |
| `-e` | `environment:` |
| `-v` | `volumes:` |
| `-p` | `ports:` |

### 常用命令

```bash
docker compose up -d        # 一键启动文件里所有容器（-d 后台）
docker compose down         # 停止并删除所有容器
docker compose stop         # 只停止不删除
docker compose start        # 重新启动
docker compose up           # 容器已在运行时重复执行无效果，不会起第二份
docker compose -f my.yml up # 文件名不是标准的 docker-compose.yml 时用 -f 指定
```

- compose 会自动识别当前目录下**严格叫 docker-compose 的文件**，改名后要用 `-f`；
- **AI 时代偷懒法**：把想要的 docker 命令丢给 AI，让它生成等价的 compose 文件；
- Compose 是**轻量级、单机**编排；企业级服务器集群的大规模编排要用 **Kubernetes（K8s）**。

---

## 1.11 命令速查表

| 命令 | 作用 |
|------|------|
| `docker pull 镜像` | 下载镜像 |
| `docker images` | 列出本地镜像 |
| `docker rmi 镜像` | 删除镜像 |
| `docker run -d -p 外:内 -v 宿:容 -e K=V --name 名 镜像` | 创建并运行容器 |
| `docker ps` / `docker ps -a` | 看运行中 / 所有容器 |
| `docker stop` / `start` / `rm -f 容器` | 停止 / 启动 / 删除容器 |
| `docker logs -f 容器` | 滚动看日志 |
| `docker exec -it 容器 /bin/bash` | 进入容器内部 |
| `docker inspect 容器` | 查看容器完整配置 |
| `docker volume create/inspect/list/rm/prune` | 管理挂载卷 |
| `docker network create/list/remove` | 管理网络 |
| `docker build -t 名:版本 .` | 按 Dockerfile 构建镜像 |
| `docker login` / `docker push 用户名/镜像` | 登录 / 推送镜像到 Docker Hub |
| `docker compose up -d` / `down` / `stop` / `start` | 一键编排多容器 |

---

## ✍️ 说明

- 笔记按视频章节顺序整理，是看视频时的理解与补充，**不是字幕原文复制**；视频里的语音识别错别字（速度→sudo、N减X→nginx、乌邦图→Ubuntu、cooper nice→Kubernetes、待变→Debian、reg flow→RAGFlow 等）已修正。
- 视频版权归 UP 主「技术爬爬虾」所有，本笔记仅作个人学习用途。
- 配套安装/镜像站文档：https://github.com/tech-shrimp/docker_installer
