# Docker 容器日志栈

*[English](README.md) | 简体中文*

宿主机上的所有 Docker 容器，自动发现，并可在 Grafana 中检索。

```
  宿主机上的每个容器
            |  (Docker API：服务发现 + 跟读日志)
            v
      Grafana Alloy  --推送-->  Loki  <--查询--  Grafana
     (logging-alloy)         (logging-loki)    (logging-grafana)
                                                       |
                                              http://localhost:24680
```

| 组件 | 镜像 | 职责 | 对外发布 |
| --- | --- | --- | --- |
| Loki | `grafana/loki:3.7.6` | 日志存储，保留 30 天 | 否 - 仅限内部网络 |
| Alloy | `grafana/alloy:v1.18.1` | 发现并跟读每个容器 | UI 位于 `127.0.0.1:12345` |
| Grafana | `grafana/grafana:13.1.3` | 查询界面，由文件预置 | `127.0.0.1:24680` |

面向 Docker Engine 29.x + Compose v5.x。`docker-compose.yml` 中**刻意**没有那个已废弃的顶层
`version:` 键。

## 目录结构

```
.
|-- docker-compose.yml
|-- .env                      (需要你自己创建；已被 git 忽略)
|-- .env.example
|-- .gitignore
`-- config/
    |-- loki/loki.yaml                            # 存储 + 保留期
    |-- alloy/config.alloy                        # 发现 + 跟读
    `-- grafana/
        |-- provisioning/
        |   |-- datasources/loki.yml              # Loki 数据源，uid: loki
        |   `-- dashboards/dashboards.yml         # 仪表板 provider
        `-- dashboards/
            |-- container-logs.json               # 总览：有没有出问题？
            `-- log-search.json                   # 搜索：找到并阅读具体日志行
```

两个 provisioning 子目录是各自单独挂载的，而不是挂载它们的父目录 `provisioning/`。挂载父目录会
遮盖掉 Grafana 镜像内自带的 `plugins/` 和 `alerting/` 目录，Grafana 每次启动都会为它们各打一条
ERROR 日志——而这些噪音随后会被 Alloy 采集，并被仪表板上的错误面板计入统计。

## 首次启动

### 第 0 步 - 拆除上一套栈（会销毁所有已存日志）

**不要跳过这一步。** `docker compose up -d` 会重建容器，但**绝不会**重建一个已存在的具名卷。上一次
安装遗留的 `logging_loki-data`、`logging_alloy-data`、`logging_grafana-data` 三个卷仍然留在这台
宿主机上，继承它们会以三种**无声**的方式破坏这次重建：旧的 `grafana.db` 里已经有 `admin` 用户，
于是你新设的 `GRAFANA_ADMIN_PASSWORD` 会被忽略；旧的 Loki 数据是按另一套 schema 写入的；而被保留
下来的那些流所携带的 `job` 标签，和这里每个面板所筛选的值并不一致。

```bash
cd /home/wren/codes/logging
docker rm -f logging-loki logging-alloy logging-grafana 2>/dev/null || true
docker volume rm -f logging_loki-data logging_alloy-data logging_grafana-data 2>/dev/null || true
docker network rm logging 2>/dev/null || true
```

### 第 1 步 - 给 Docker 自己的日志文件设上限（务必在首次启动前做）

参见 [宿主机层面的日志轮转](#宿主机层面的日志轮转)。不做这一步，首次启动会回放一大批积压日志，而且
`/var/lib/docker/containers/` 会永无止境地增长下去。Loki 的 30 天保留期**管不到**那些文件。

### 第 2 步 - 配置并启动

```bash
cp .env.example .env
$EDITOR .env                     # 设置 GRAFANA_ADMIN_PASSWORD - 不设置整套栈起不来
docker compose up -d
docker compose ps
```

如果你是通过网络访问这台机器、而不是坐在它前面，那还需要在 `.env` 里设置
`GRAFANA_BIND_IP=0.0.0.0` 和 `GRAFANA_ROOT_URL=http://<host>:24680`——并且要在 24680 端口前面放一道
防火墙或反向代理，因为那时 Grafana 就是横在网络和你的日志之间的唯一屏障。这台宿主机上的上一套栈
发布在 `0.0.0.0:3000`；而这里的默认值是仅监听回环地址，对远程用户来说这看起来就像整个服务挂了，
而且任何地方都不会有错误信息。

打开 <http://localhost:24680> 并登录。你会直接落在 **Container Logs** 页面上。Loki 数据源和两个
仪表板都已经就位——不需要点任何东西。

一共两个仪表板，它们回答的是不同的问题：

| | **Container Logs** | **Log Search** |
| --- | --- | --- |
| 回答什么 | 有没有出问题，出在哪？ | 那条具体的日志在哪？ |
| 展示什么 | 错误数与错误占比、按严重级别的日志量、最吵的容器、一小段最近的问题日志 | 各种筛选器、用于在时间上导航的严重级别直方图，以及一个大的日志列表 |
| 怎么进去 | 首页仪表板 | 它标题栏里的 `Log Search` 链接 |

每个页面标题栏里的链接都会把你当前的筛选条件和时间范围带到另一个页面，所以在总览页把范围收窄到
某个服务、再点进去，落地时看到的就是同一批日志行。点击"最吵的容器"表格里的容器名，效果同理。

日志行一条一行渲染，不带标签前缀。点击任意一行，会在列表旁边展开，显示完整文本以及全部标签和
结构化元数据字段（`container_id`、`image`）。

**为什么用 24680 而不是 Grafana 惯用的 3000：** 在这台宿主机上，3000 和 3001 属于开发服务器的地盘
——3000 被一个长期运行的 `rsbuild dev` 服务占着（`tokenpapa` 前端项目），而你接下来随手启动的任何
东西都会去抢 3001。一旦抢输，Compose 就会带着 `address already in use` 起不来 Grafana。24680 无人
占用，不是任何注册在案的服务端口，并且低于本机内核的临时端口范围（32768-60999），因此也不会被某个
出站连接的源端口抢走。**只有宿主机一侧的端口变了**：Grafana 在容器**内部**仍然监听 3000，这就是为什么
健康检查、以及 `ports:` 映射的右半边写的还是 3000。想换成别的端口就改 `.env` 里的 `GRAFANA_PORT`
——`GRAFANA_ROOT_URL` 必须同步改，否则分享链接和重定向会指向旧端口。

## 采集了什么

宿主机上的每一个容器，通过 Docker API 发现，每 2 秒刷新一次。启动一个新容器，它的日志就会出现，
不需要碰任何配置文件。

流标签（stream labels）——这是 `config/alloy/config.alloy` 和仪表板里 LogQL 之间的契约。改一边，
就得改另一边：

| 标签 | 来源 | 说明 |
| --- | --- | --- |
| `job` | 常量 `docker` | 每条流都带；`{job="docker"}` 就是"全部" |
| `container` | `__meta_docker_container_name` | 去掉了开头的 `/` |
| `compose_project` | `com.docker.compose.project` | 非 Compose 容器上不存在 |
| `compose_service` | `com.docker.compose.service` | 非 Compose 容器上不存在 |
| `service_name` | compose service，没有则用容器名 | Loki 3.x / Logs Drilldown 的约定 |
| `stream` | `__meta_docker_container_log_stream` | `stdout` / `stderr` |

正因为非 Compose 容器上没有 `compose_project` 和 `compose_service`，每个模板变量用的都是
`allValue: ".*"` 而不是 `".+"`。能匹配空字符串的匹配器同时也会选中那些**缺失该标签**的序列，所以
"All" 才真的意味着全部。把它们换成 `.+` 会悄无声息地隐藏掉每一个独立启动的容器。

### 结构化元数据（逐条存储，不建索引）

| 字段 | 说明 |
| --- | --- |
| `container_id` | 前 12 个字符；能告诉你某行日志是被重建过的容器的**哪一次化身**打出来的 |
| `image` | **尽力而为 - 见下文** |

在 LogQL 里这样筛选：`{job="docker"} | container_id="a1b2c3d4e5f6"`。它们存在 chunk 内部，而不在
流索引里，所以不产生任何基数（cardinality）开销。这一点对 `image` 尤其重要：如果把它做成索引标签，
每个容器每次发版都会凭空产生一条全新的流。

**`image` 是不完整的，这确实是相对需求而言的一个真实缺口。** Alloy 的 Docker 发现委托给了
Prometheus 的 moby SD，而后者只导出容器 id、名称和 network-mode，外加 `container_label_*` /
`network_*` / `port_*`——根本没有 `__meta_docker_container_image` 这个东西。配置里能捞回多少是多少，
按来源从差到好排列：

1. `com.docker.compose.image` - Compose 会把它打在自己创建的所有东西上，但值是镜像的 **digest**
   （`sha256:...`），不是可读的 tag。
2. `org.opencontainers.image.version` / `org.opencontainers.image.ref.name` - OCI 标签，Docker 会
   把它们从镜像复制到容器上。很多镜像设了，也有很多没设。
3. `logging.image` - 在你自己能控制的容器上手动打上：
   `labels: ["logging.image=myrepo/myapp:1.2.3"]`。这是唯一一个**总能**给出可读 tag 的来源。

一个用 `docker run` 直接起的、镜像又没有 OCI 标签的容器，压根不会有 `image` 值。

### 不采集的内容

出于设计和需求的明确取舍：journald、宿主机上的文件日志，以及任何外部/远程推送。这套栈只通过
Docker API 读取容器的 stdout/stderr。

### 排除某个容器

```yaml
labels:
  logging.exclude: "true"
```

**Alloy 和 Loki 两者都带着这个标签**，并且在 `config.alloy` 里还额外有一条 project+service 规则把
它们丢掉，属于双保险。这两个都处在推送链路上，所以它们自己的错误输出会反馈回自身：

- Alloy：一次推送失败会打一条错误日志，这条日志会被采集、被推送，然后再次失败。
- Loki：每一次摄入被拒（429 限流、400 数据太旧、磁盘满）都会往 Loki 的 stdout 写一条错误日志。把它
  再送回去，就会产生下一次拒绝和下一条日志——这个循环恰恰在 Loki 已经处于劣化状态、你最需要它恢复
  的时候被触发。

**Grafana 也带这个标签，但理由不同：是为了降噪，不是为了安全。** 它不在推送链路上，无法形成放大。
排除它是因为每次面板刷新都会打出好几条 `logger=tsdb.loki endpoint=queryData`，于是在一台安静的
宿主机上，Grafana 喋喋不休地讲述自己在查询 Loki，会把你真正想读的那些容器彻底淹没。想把它的日志
放回来，就从 `docker-compose.yml` 里删掉这个标签并重建容器。

所以这三个栈内容器全都被排除在采集之外，它们的日志只能通过
`docker compose logs -f {alloy,loki,grafana}` 来看。

这个标签同时也是一个逃生口：适用于日志驱动读不回来的容器，或者吵到不值得为它建索引的容器。

### 已知的采集盲区

以下都是真实存在的，没有一个能靠配置解决，而且全部是**静默**失败：

- **短命容器。** 发现机制是轮询，不是事件流。整个生命周期不到约 2 秒的容器永远不会被发现，它的日志
  一条都采不到——`docker compose run --rm` 这类一次性任务、init/迁移作业，以及快速崩溃重启的容器
  （而这些恰恰是你最想看的）。任何地方都不会报错。
- **没有网络的容器。** 上游的发现逻辑是通过遍历容器的网络来构造采集目标的；`network_mode: none`
  （或者一个解析不了的 `network_mode: container:<id>`）会得到零个目标。要做隔离，请改用
  `internal: true` 的网络。
- **读不回来的日志驱动。** `GET /containers/{id}/logs` 原生只对 `json-file`、`local` 和 `journald`
  有效。其他驱动只能通过 Docker 的双写日志缓存来读，而 `logging: {driver: none}` 会让一个容器
  **永久不可见**，*并且*在每一次发现轮询时产生一条 "could not fetch logs" 错误。在这台宿主机上
  永远不要设 `driver: none`。
- **容器重启会重读历史。** 容器停止时，它的目标随之消失，跟读器停止并**删除**自己的位点记录。因此
  一次 `docker restart` 会从头重新读一遍该容器保留下来的日志。这个范围被 `config.alloy` 里 24 小时
  的超龄丢弃规则、以及守护进程级别的 json-file 轮转所限制。
- **重启会重复约 1 秒的数据。** 位点是按秒粒度存储的，而 Docker 的 `since` 参数是闭区间，所以 Alloy
  每次重启，每个容器最多会重放 1 秒已经摄入过的日志。有界、符合预期，不是需要去追查的 bug。

去**验证**覆盖面，而不是假设它没问题：

```bash
docker ps --format '{{.Names}}' | sort
# 拿它和 Grafana Explore 里这条查询的结果对比：
#   count by (container) (count_over_time({job="docker"}[1h]))
# 差集应当恰好是 logging-alloy 和 logging-loki。
```

## 保留期

30 天，在 `config/loki/loki.yaml` 里以字面量 `retention_period: 720h` 写死，由 Loki 的 compactor
执行，它会把过期的 chunk 从 `loki-data` 卷上解链删除。`compactor.retention_enabled: true` 才是让
删除真正发生的开关；没有它，compactor 会永远压缩下去、什么都不删，而且不吭声。

删除不是瞬时的。保留检查每 20 分钟跑一次，标记出过期的 chunk，清扫器则在
`retention_delete_delay: 2h` 之后才解链。所以过期数据在 30 天线之后还会多滞留几个小时，属于正常。

这个容器**没有**带 `-config.expand-env` 运行，因此 `${...}` 在 `loki.yaml` 里没有任何含义。这是刻意
的：Loki 的展开器是 drone/envsubst，它没有为字面量 `$` 提供任何转义写法的文档，而一个解析不了的
`${...}` 会直接导致启动失败（`bad substitution`）。保留期是一个固定需求，不是随环境变化的值，所以
这层间接引用没带来任何好处，只可能把整套栈搞挂。

**保留期并不能限制头 30 天里的磁盘增长。** 在一张索引表的整个 24 小时窗口彻底过期之前，什么都不会
被删除，而且 Loki 没有基于容量的保留策略。真正的兜底是 `loki.yaml` 里的 `ingestion_rate_mb` /
`per_stream_rate_limit`，以及 `config.alloy` 里针对每个容器的 `stage.limit`。这里可持续的稳态大约
是 738GB / 30 天 ≈ 24GB/天 的磁盘占用。

## Grafana 配置即代码

Grafana 里没有任何东西是靠点鼠标配出来的。

- **数据源** - `config/grafana/provisioning/datasources/loki.yml`，`apiVersion: 1`、`uid: loki`、
  `editable: false`。`prune: true` 让这个文件成为唯一权威：在这里删掉一个条目，Grafana 下次启动时
  就会把对应的数据源删掉。
- **仪表板** - `config/grafana/provisioning/dashboards/dashboards.yml` 只是一个 *provider*；它指向
  一个目录。真正的仪表板是那个目录里的 JSON 文件。`allowUiUpdates: false` 意味着 UI 无法覆盖保存
  它们。
- `disableDeletion: false` 是刻意为之，它是仪表板层面上等价于 `prune: true` 的东西。把它设成 `true`
  **并不**意味着"以仓库为准"——它的含义是"取消预置而不是删除"，于是只要某个 JSON 文件对 Grafana
  不再可见（从 git 里删了、切了分支、挂载指到别处），那个仪表板就会作为一条普通的、可以随意编辑的
  数据库记录活下来。而这正是这套栈存在的意义所在——避免这种点出来的孤儿配置。至于在 UI 里删除一个
  被预置的仪表板，Grafana 本身就会拦住，与这个开关无关。
- provider 每 30 秒重新扫描一次，所以在宿主机上编辑 JSON 文件，不重启也会生效。

数据源文件里的 `uid: loki`，和每个仪表板面板、每个模板变量里的 `"uid": "loki"`，是同一份契约。只改
其中一边，每个面板都会显示 "Datasource loki was not found"。

数据源里的 `jsonData.maxLines: 2000` 必须小于等于 `loki.yaml` 里的
`limits_config.max_entries_limit_per_query: 10000`。一旦超过，所有面板会同时失败，报的还是 Loki
那边的错误、错误信息里点名的是一个 Loki 的限制项，而不是 Grafana 的设置。

Container Logs 带的是 `"refresh": "1m"`；Log Search 则刻意完全不设自动刷新——你在读结果的时候，
结果就应该待着别动。两个选择器都只列出 1m 及以上的选项，并且 `GF_DASHBOARDS_MIN_REFRESH_INTERVAL=1m`
在服务端强制了这个下限（Grafana 在构造时就会丢弃更短的条目，所以列出 10s 只会是一段死 JSON）。有
几个面板是锚定在 `[$__range]` 上的即时查询；在 30 天范围加 10 秒刷新的组合下，它们堆叠全量扫描
聚合的速度会超过单体 Loki 的应答速度。要做真正的实时跟读，请用 Explore -> Live。

### 新增一个仪表板

1. 在 UI 里做好。
2. **Export -> Save to file**（不要用"导出以供外部分享"那种方式——要保留硬编码的 `"uid": "loki"`，
   别让 Grafana 把它变成一个 `__inputs` 占位符）。
3. 把 JSON 放进 `config/grafana/dashboards/`，给它一个稳定的顶层 `"uid"`，然后提交。

Grafana 13 写出来的是 `"schemaVersion": 42`。更老的导出文件会在加载时被向前迁移。

## 宿主机层面的日志轮转

`docker-compose.yml` 里的 `logging:` 块只把**这三个**容器限制在 3 × 20 MB。它管不到别处定义的容器，
而且 **Loki 的 30 天保留期完全覆盖不到 Docker 自己的日志文件**——Alloy 是通过 API 去读它们的，从不
截断它们。没有轮转的话，`/var/lib/docker/containers/*/*-json.log` 会在每个容器的整个生命周期内
无限增长；文件系统一旦被写满，Docker 守护进程就会倒下，Loki、Grafana 和 Alloy 会跟着一起完蛋。

`/etc/docker/daemon.json`：

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "3" }
}
```

```bash
# log-driver / log-opts 不在 dockerd 支持 SIGHUP 热加载的配置集合里，所以
# `systemctl reload docker` 会返回 0 然后悄悄地什么都不做。必须重启。
sudo systemctl restart docker
docker info --format '{{.LoggingDriver}}'
```

这只对之后创建的容器生效；已存在的容器会保留原有设置，直到被重建。50m × 3 = 每容器 150MB，这同时
也是 Alloy 万一卡住之后的追赶预算——在 Alloy 读到之前就被轮转掉的数据就是丢了，所以别把它缩到
10m × 1。

如果这台宿主机已经跑了好几个月，而你不想在首次启动时经历一次大规模回放，可以在第一次启动前把已有
积压清零（安全操作——Docker 持有着文件描述符，会继续往后追加）：

```bash
sudo truncate -s 0 /var/lib/docker/containers/*/*-json.log
```

另外 Alloy 会丢弃任何超过 24 小时的日志条目（`stage.drop { older_than = "24h" }`），所以即便你跳过
上面的 truncate，回填量也是有界的。

## 安全决策

### Docker socket 的暴露 - 请把这一节读完

Alloy 挂载了 `/var/run/docker.sock:ro`。**`:ro` 标志不是一道安全边界。** 它让这个 socket **文件**
只读；它不会过滤 Docker API。任何能和那个 socket 对话的东西，都可以带着 `Privileged: true` 和
`Binds: ["/:/host"]` 去 `POST /containers/create`，然后彻底拿下这台宿主机。Docker socket 权限就等于
root 权限，没有余地，而且 Alloy 是以 root 运行的。

**出厂默认：直接挂载。** 理由——这是一台单节点、单租户的宿主机；Alloy 是 Grafana 官方出品的二进制，
跑的是本地编写的配置，远程配置已禁用，它那个无认证的 UI 只绑在回环地址上。而需求是可靠地采集
**所有**容器，在长连接的 `follow` 日志流前面塞一个 HTTP 代理，恰恰是那条会导致流被截断、出现日志
空洞的路。

**如果这台宿主机上跑着任何不受信任的、或者面向公网的东西，就用 socket 代理。** 它能把一次假想中的
Alloy 沦陷从宿主机 root 降级为一个只读的 Docker API。存成
`docker-compose.socket-proxy.yml`：

```yaml
services:
  docker-socket-proxy:
    image: tecnativa/docker-socket-proxy:v0.5.0
    container_name: logging-docker-socket-proxy
    restart: unless-stopped
    environment:
      CONTAINERS: 1   # /containers/json 和 /containers/{id}/logs
      NETWORKS: 1     # discovery.docker 每次刷新都会调 NetworkList - 必须开
      VERSION: 1      # 客户端启动时的 API 版本协商
      POST: 0         # 拒绝所有写操作动词 - 这才是真正的边界
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - logging
    labels:
      logging.exclude: "true"
    security_opt:
      - no-new-privileges:true
    mem_limit: 64m
    memswap_limit: 64m
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  alloy:
    # !override 是整体替换列表而非合并，这样才能把 docker.sock 那条挂载去掉。
    volumes: !override
      - ./config/alloy/config.alloy:/etc/alloy/config.alloy:ro
      - alloy-data:/var/lib/alloy/data
    depends_on:
      docker-socket-proxy:
        condition: service_started
      loki:
        condition: service_started
```

守护进程地址是刻意在 `config.alloy` 里硬编码的（否则一个未设置的环境变量会得到空主机名，然后产生
一个让人摸不着头脑的失败），所以切换过去还意味着要改 `config/alloy/config.alloy` 里**两处** `host =`：

```
discovery.docker "containers"   -> host = "tcp://docker-socket-proxy:2375"
loki.source.docker "containers" -> host = "tcp://docker-socket-proxy:2375"
```

```bash
docker compose -f docker-compose.yml -f docker-compose.socket-proxy.yml up -d
```

这个代理底层是 HAProxy，它的 `timeout server` 到期时会切断那条长连接的 `logs?follow=1` 流。Alloy 会
从存下来的位点重连，所以这属于抖动而不是丢数据——但在收工之前，请确认日志流确实能挺过那个超时窗口。
留意是否出现空洞；如果有，那还是直接挂载这笔交易更划算。

把 Alloy 以非 root 用户、加入宿主机 `docker` 组的方式运行，**不是**一个替代方案。特权在于 API，
不在于 uid。

### 匿名访问：关闭，而且值得一直关着

`GF_AUTH_ANONYMOUS_ENABLED=false`。容器日志里经常夹带 bearer token、连接串、客户标识和堆栈跟踪。
开放匿名 viewer 访问，等于把这一切交给任何能够到 24680 端口的东西。

### 对外发布的端口

Loki 从不对外发布——它只能在 `logging` 网络内部通过 `http://loki:3100` 访问。Loki 自身没有任何认证，
所以把它发布出去就等于暴露一个无认证的、可读可写的日志 API。Grafana 和 Alloy UI 默认绑定在
`127.0.0.1`；其中 Alloy UI 尤其要注意，它完全没有登录，而且会在组件图里把每一个参数值都渲染出来。

### 其他加固

每个服务都带 `no-new-privileges:true`；`GF_USERS_ALLOW_SIGN_UP=false`；Grafana 的数据回传、更新检查、
插件更新检查、新闻推送和启动时的插件预装全部关闭，这也顺带让它在一台物理隔离的宿主机上也能正常
启动。Loki 的 `analytics.reporting_enabled` 为 false，Alloy 带 `--disable-reporting` 运行。

## 为什么 Loki 没有健康检查

`grafana/loki:3.7.6` 是一个用 Bazel 构建的 distroless Debian 13 镜像（`docker history` 里能看到
`cacerts_debian13`、`os_release_debian13` 以及 trixie 的 `base-files`/`netbase`/`tzdata`）。它不带
shell、不带 `wget`、也不带 `curl`；`/usr/bin/loki` 就是 entrypoint，而 `USER 10001` 和 `/loki` 的
属主都来自 Loki 自己的层（`COPY --chown=10001:10001 /loki /loki`）——这也是为什么
`docker-compose.yml` 里的 `user: "10001:10001"` 与镜像完全吻合。

Docker 健康检查是在容器**内部**执行的，所以这里根本没有东西可以执行。该服务显式声明了
`healthcheck: disable: true` 来记录"这是刻意的"，依赖它的服务则等待 `service_started`。想要一个真答案
时，去问一个确实装了 curl 的容器：

```bash
docker compose exec -T grafana curl -fsS http://loki:3100/ready
```

Alloy 的健康检查探测的是 **`/-/healthy`**，不是 `/-/ready`。`/-/ready` 只报告初始配置加载完成了，
所以哪怕 `loki.write` 每次推送都在失败、一条日志都没发出去，它照样返回 200。`/-/healthy` 会遍历
每一个组件，并在有组件不健康时返回 500 并点名——而这正是这套栈最需要暴露出来的故障。Alloy 镜像
既没有 curl 也没有 wget，所以探针用的是 bash 的 `/dev/tcp`。

**需要预期到的后果：** 只要宿主机上有任意一个容器使用了读不回来的日志驱动，Alloy 容器就可能报
`unhealthy`，而与此同时它对其他所有容器的采集完全正常。在断定采集器挂了之前，先去
<http://127.0.0.1:12345> 看组件图。

## 持久性，以及一次 Loki 故障的代价

`loki.write` 在内存里缓冲并以指数退避重试——20 次重试，间隔从 500ms 到 5 分钟，大约能兜住一个小时。
**超过之后，那批在途数据会被永久丢弃**，而且由于跟读器早已推进了自己的读取位点，那些日志行永远不会
被重新读一遍。请针对 `loki_write_dropped_entries_total` 配告警。

Alloy 的磁盘 WAL 本可以堵上这个窗口，但 `wal` 块是一个实验性特性，要求整个进程以
`--stability.level=experimental` 运行。这里是刻意没有启用的。要启用：在 alloy 的 `command:` 里加上
`--stability.level=experimental`，在 `loki.write` 的 endpoint 块里加上
`wal { enabled = true, max_segment_age = "1h" }`，并把 Alloy 的 `stop_grace_period` 提到默认 30 秒的
排空超时之上。

在 Loki 这一侧，`ingester.wal.flush_on_shutdown: true` 意味着 SIGTERM 真的会把 head chunk 刷盘
（默认是 false，那会让 60 秒的 `stop_grace_period` 变得毫无意义）。

## 资源占用

这台宿主机有 30 GiB 内存（约 17 GiB 已用）、16 核、8 GiB swap 和 738 GiB 空闲磁盘。

| 服务 | 典型 RSS | `mem_limit` = `memswap_limit` | `cpus` | `GOMEMLIMIT` |
| --- | --- | --- | --- | --- |
| Loki | 300-900 MiB | 4 GiB | 4.0 | 3600MiB |
| Alloy | 100-250 MiB | 1 GiB | 2.0 | 900MiB |
| Grafana | 100-200 MiB | 1 GiB | 2.0 | 900MiB |

有两个设置让上面这些限制真正起作用：

- **`memswap_limit` 等于 `mem_limit`。** 这台宿主机跑的是 cgroup v2，有 8 GiB swap。只设 `mem_limit`
  的话，`memory.swap.max` 会保持在 `max`，于是一个撞到上限的容器**不会**被杀掉——它会溢出到 swap，
  而 swap 所在的 NVMe 设备正是存着 `/var/lib/docker` 的那块盘，结果整台宿主机劣化成 IO 抖动，而那个
  容器活得好好的、只是没有响应。两个值设成相等，就得到一个硬上限和一次干脆的 OOM-kill，而
  `restart: unless-stopped` 能从中恢复。
- **`GOMEMLIMIT`。** Go 从 1.25 起会读 cgroup 的 **CPU** 限制，但从来不读 cgroup 的**内存**限制，
  所以不设这个，GC 根本不知道 `mem_limit` 的存在，会一路把堆撑破它。把它设成各自限制的约 90%，让 GC
  在内核动手之前就更卖力地工作。

Loki 的缓存上限是约 640 MiB，而不是它的 `results_cache` 块表面上看起来的 128 MiB：
`cache_index_stats_results`、`cache_volume_results`、`cache_series_results` 和 `cache_label_results`
默认全为 true，而且每一个都会把 `results_cache` **克隆**成自己独立的内嵌缓存。这里刻意没有配
`chunk_store_config` 的 chunk 缓存——对象存储就是一个本地目录，那些字节本来就由内核页缓存免费提供，
还不带 GC 压力。

`ingester.wal.replay_memory_ceiling` 被钉在 2GB（默认 4GB），这样非正常重启后的 WAL 回放就不会超出
`GOMEMLIMIT`。

磁盘：Loki 存的是压缩后的 chunk，大约是原始日志量的 10-20%。一台每天产生 1 GiB 原始日志的宿主机，
整个 30 天窗口大概占 3-6 GiB。

## 日常运维

```bash
# 状态 / 健康
docker compose ps
docker compose logs -f alloy
docker compose logs -f loki        # Loki 被排除在采集之外；这是看它日志的唯一途径

# 应用配置改动（Alloy 和 Loki 只在重启时重新加载配置）
docker compose restart alloy
docker compose up -d               # 改完 docker-compose.yml 或 .env 之后

# 仪表板 30 秒内会被重新扫描；数据源改动需要重启
docker compose restart grafana

# Loki 实际拥有哪些标签？
docker compose exec -T grafana curl -fsS 'http://loki:3100/loki/api/v1/labels'
docker compose exec -T grafana curl -fsS 'http://loki:3100/loki/api/v1/label/container/values'

# 保留策略在跑吗？
docker compose exec -T grafana curl -fsS http://loki:3100/metrics \
  | grep loki_compactor_apply_retention_last_successful_run_timestamp_seconds

# Alloy 有在丢数据吗？
curl -fsS http://127.0.0.1:12345/metrics | grep -E 'loki_process_dropped_lines_total|loki_write_dropped_entries_total'

# Alloy 组件图和实时调试
xdg-open http://127.0.0.1:12345

# 升级：改 docker-compose.yml 里的镜像 tag，然后
docker compose pull && docker compose up -d

# 备份 Grafana 状态（仪表板在 git 里；这里备的是用户、偏好设置、注解）
docker run --rm -v logging_grafana-data:/data -v "$PWD:/backup" alpine \
  tar czf /backup/grafana-data.tgz -C /data .

# 彻底重置 - 会销毁所有已存日志
docker compose down -v
```

## 故障排查

**`.env` 里的管理员密码不好使** - `grafana-data` 卷是上一次安装遗留下来的，而 Grafana 只在**创建**
admin 用户的那一刻才应用这个密码。要么执行"首次启动"的第 0 步，要么就地重置：
`docker compose exec -T grafana grafana-cli admin reset-admin-password '<new password>'`。

**"Datasource loki was not found"** - 仪表板里的 `"uid": "loki"` 和 `datasources/loki.yml` 里的
`uid:` 对不上了。检查 `docker compose logs grafana | grep -i provisioning`。

**仪表板是空的，但 Alloy 是健康的** - 先确认标签存在（`/loki/api/v1/labels`）。如果 `job` 不见了，
说明 Alloy 发现了目标但没在发送；去 <http://127.0.0.1:12345> 看组件图。

**Alloy 报 `unhealthy`，但日志在正常流动** - 如果宿主机上有容器用了读不回来的日志驱动，这是预期
行为。探针是 `/-/healthy`，它会报告**任意一个**不健康的组件。打开组件图看是哪一个。

**少了某个容器** - (a) 它带着 `logging.exclude=true`；(b) 它是 `logging-alloy` 或 `logging-loki`
（两者都是设计上排除的）；(c) 它用的日志驱动不是 `json-file`/`local`/`journald`；(d) 它没有接任何
网络；或者 (e) 它活的时间还不到一个 2 秒的发现轮询周期。参见"已知的采集盲区"。

**全新启动后看不到超过 24 小时的日志** - 这是设计如此。`config.alloy` 里的
`stage.drop { older_than = "24h" }` 限制了首次启动的回填量。想要更多历史就把它放宽，上限是 Loki 的
`reject_old_samples_max_age`，即 720h。

**Compose 拒绝启动，并报了一条关于 `GRAFANA_ADMIN_PASSWORD` 的消息** - 这是按预期工作。要么 `.env`
不存在，要么密码是空的。

**Loki 没有健康状态显示** - 预期行为。参见"为什么 Loki 没有健康检查"。

**面板报错 "max entries limit per query exceeded"** - 数据源里的 `jsonData.maxLines` 超过了
`loki.yaml` 里的 `limits_config.max_entries_limit_per_query`。

**某个容器出现 429 / 日志缺失** - 它撞上了 `per_stream_rate_limit`（4MB）或者 Alloy 针对每个容器的
`stage.limit`（1000 行/秒）。检查 `loki_process_dropped_lines_total{reason="rate_limit"}`。
