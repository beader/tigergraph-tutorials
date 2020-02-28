# 安装

## 下载

下载地址: [https://www.tigergraph.com/download/](https://www.tigergraph.com/download/)

如果只是体验的话，可以下载 Developer Edition，可以体验到绝大多数功能

### Linux 系统安装 TigerGraph

```bash
$ wget https://dl.tigergraph.com/developer-edition/tigergraph-2.5.0-developer.tar.gz
$ tar xzvf tigergraph-2.5.0-developer.tar.gz
$ cd tigergraph-2.5.1-developer
$ sh ./install.sh
```

{% hint style="info" %}
安装完成之后，会在系统中创建一个 tigergraph 用户，切换至该用户，可以对系统进行一系列管理操作
{% endhint %}

```bash
$ su - tigergraph
$ gadmin status
Welcome to TigerGraph Developer Edition, free for non-production, research, or educational use.
=== zk ===[SUMMARY][ZK] process is up[SUMMARY][ZK] /data5/tigergraph/zk is ready=== kafka ===[SUMMARY][KAFKA] process is up
[SUMMARY][KAFKA] queue is ready
=== gse ===
[SUMMARY][GSE] process is up
[SUMMARY][GSE] id service is ready (online)
=== dict ===
[SUMMARY][DICT] process is up
[SUMMARY][DICT] dict server is ready
=== ts3 ===
[SUMMARY][TS3] process is up
[SUMMARY][TS3] ts3 is ready
=== graph ===
[SUMMARY][GRAPH] graph is ready
=== nginx ===
[SUMMARY][NGINX] process is up
[SUMMARY][NGINX] nginx is ready
=== restpp ===
[SUMMARY][RESTPP] process is up
[SUMMARY][RESTPP] restpp is ready
=== gpe ===
[SUMMARY][GPE] process is up
[SUMMARY][GPE] graph is ready (online)
=== gsql ===
[SUMMARY][GSQL] process is up
[SUMMARY][GSQL] gsql is ready
=== Visualization ===
[SUMMARY][VIS] process is up (VIS server PID: 2091)
[SUMMARY][VIS] gui server is up
```

### 使用 Docker 运行 TigerGraph

确认系统中已经安装好了 Docker

```bash
$ docker pull docker.tigergraph.com/tigergraph-dev:latest
$ docker run --name tigergraph -d -p 9000:9000 -p 14240:14240 --ulimit nofile=1000000:1000000 docker.tigergraph.com/tigergraph-dev:latest
```

通过 `-p 9000:9000 -p 14240:14240` 把容器中这两个端口映射至宿主机，方便后续的使用。

容器启动之后，通过 `docker exec` 进入容器，查看运行情况

```bash
$ docker exec -it tigergraph /bin/bash
$ su - tigergraph
$ gadmin status
```

如果看到各个组建均为 up 状态，则表明服务已经启动，否则，需要手动启动各个服务:

```bash
$ gadmin start
```

