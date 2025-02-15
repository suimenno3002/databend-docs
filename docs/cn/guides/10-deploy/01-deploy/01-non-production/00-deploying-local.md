---
title: Docker 和本地部署
---

import FunctionDescription from '@site/src/components/FunctionDescription';
import GetLatest from '@site/src/components/GetLatest';
import DetailsWrap from '@site/src/components/DetailsWrap';
import StepsWrap from '@site/src/components/StepsWrap';
import StepContent from '@site/src/components/Steps/step-content';

为了快速访问 Databend 功能并获得实践经验，您有以下部署选项：

- [在 Docker 上部署 Databend](#deploying-databend-on-docker)：您可以在 Docker 上部署 Databend 和 [MinIO](https://min.io/)，以实现容器化设置。
- [部署本地 Databend](#deploying-a-local-databend)：如果对象存储不可用，您可以选择本地部署，并使用文件系统作为存储后端。

:::note 仅限非生产用途

- 对象存储是使用 Databend 的生产环境的要求。文件系统只应用于评估、测试和非生产场景。

- 本章节涉及的 MinIO 部署只适合开发演示使用，单机环境资源有限，不建议用于生产环境或性能测试目的。
:::

## 在 Docker 上部署 Databend {#deploying-databend-on-docker}

### 视频演示

<iframe width="853" height="505" className="iframe-video" src="//player.bilibili.com/player.html?aid=1651684130&bvid=BV1Aj421f74e&cid=1467602365&p=1&autoplay=0" frameBorder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowFullScreen> </iframe>

### 操作步骤

开始之前，请确保您的系统上安装了 Docker。

<StepsWrap>
<StepContent number="1" title="部署 MinIO">

1. 使用以下命令作为容器拉取并运行 MinIO 镜像：

```shell
mkdir -p ${HOME}/minio/data

docker run -d \
   --name minio \
   --user $(id -u):$(id -g) \
   --net=host \
   -e "MINIO_ROOT_USER=ROOTUSER" \
   -e "MINIO_ROOT_PASSWORD=CHANGEME123" \
   -v ${HOME}/minio/data:/data \
   minio/minio server /data --console-address ":9001"
```

:::note
我们在这里将控制台地址更改为 `:9001`，以避免端口冲突。
:::

请注意，上述命令还设置了根用户凭据（ROOTUSER/CHANGEME123），您将需要在后续步骤中提供这些凭据以进行身份验证。如果您在此时更改了根用户凭据，请确保在整个过程中保持一致性。

您可以通过在终端中检查以下消息来确认 MinIO 容器已成功启动：

```shell
❯ docker logs minio
Formatting 1st pool, 1 set(s), 1 drives per set.
WARNING: Host local has more than 0 drives of set. A host failure will result in data becoming unavailable.
MinIO Object Storage Server
Copyright: 2015-2024 MinIO, Inc.
License: GNU AGPLv3 <https://www.gnu.org/licenses/agpl-3.0.html>
Version: RELEASE.2024-01-05T22-17-24Z (go1.21.5 linux/arm64)

Status:         1 Online, 0 Offline.
S3-API: http://192.168.106.3:9000  http://172.17.0.1:9000  http://192.168.5.1:9000  http://127.0.0.1:9000
Console: http://192.168.106.3:9001 http://172.17.0.1:9001 http://192.168.5.1:9001 http://127.0.0.1:9001

Documentation: https://min.io/docs/minio/linux/index.html
Warning: The standard parity is set to 0. This can lead to data loss.
```

2. 打开您的网络浏览器并访问 http://127.0.0.1:9001/（登录凭据：ROOTUSER/CHANGEME123）。创建一个名为 **databend** 的存储桶。

</StepContent>

<StepContent number="2" title="部署 Databend">

使用以下命令作为容器拉取并运行 Databend 镜像：

```shell
docker run -d \
    --name databend \
    --net=host \
    -v meta_storage_dir:/var/lib/databend/meta \
    -v log_dir:/var/log/databend \
    -e QUERY_DEFAULT_USER=databend \
    -e QUERY_DEFAULT_PASSWORD=databend \
    -e QUERY_STORAGE_TYPE=s3 \
    -e AWS_S3_ENDPOINT=http://${IP}:9000 \
    -e AWS_S3_BUCKET=databend \
    -e AWS_ACCESS_KEY_ID=ROOTUSER \
    -e AWS_SECRET_ACCESS_KEY=CHANGEME123 \
    datafuselabs/databend
```

> 这里的 ${IP} 是 192.168.106.3 或 192.168.5.1，应用程序需要访问 s3。所以如果您不知道 ${IP}，可以参考 `docker logs minio` 的输出

启动 Databend Docker 容器时，您可以使用环境变量 QUERY_DEFAULT_USER 和 QUERY_DEFAULT_PASSWORD 指定用户名和密码。如果没有提供这些变量，将创建一个没有密码的默认 root 用户。上述命令创建了一个 SQL 用户（databend/databend），您将需要在下一步中使用它来连接到 Databend。如果您在此时更改了 SQL 用户，请确保在整个过程中保持一致性。

您可以通过在终端中检查以下消息来确认 Databend 容器已成功启动：

```shell
❯ docker logs databend
==> QUERY_CONFIG_FILE is not set, using default: /etc/databend/query.toml
==> /tmp/std-meta.log <==
Databend Metasrv

Version: v1.2.287-nightly-8930689add-simd(1.75.0-nightly-2024-01-07T22:13:53.249351145Z)
Working DataVersion: V002(2023-07-22: Store snapshot in a file)

Raft Feature set:
    Server Provide: { append:v0, install_snapshot:v0, install_snapshot:v1, vote:v0 }
    Client Require: { append:v0, install_snapshot:v0, vote:v0 }

On Disk Data:
    Dir: /var/lib/databend/meta
    DataVersion: V0(2023-04-21: compatible with openraft v07 and v08, using openraft::compat)
    In-Upgrading: None

Log:
    File: enabled=true, level=INFO, dir=/var/log/databend, format=json
    Stderr: enabled=false(To enable: LOG_STDERR_ON=true or RUST_LOG=info), level=WARN, format=text
    OTLP: enabled=false, level=INFO, endpoint=http://127.0.0.1:4317, labels=
    Tracing: enabled=false, capture_log_level=INFO, otlp_endpoint=http://127.0.0.1:4317
Id: 0
Raft Cluster Name: foo_cluster
Raft Dir: /var/lib/databend/meta
Raft Status: single

HTTP API
   listening at 127.0.0.1:28002
gRPC API
   listening at 127.0.0.1:9191
   advertise:  -
Raft API
   listening at 127.0.0.1:28004
   advertise:  colima:28004

Upgrade on-disk data
    From: V0(2023-04-21: compatible with openraft v07 and v08, using openraft::compat)
    To:   V001(2023-05-15: Get rid of compat, use only openraft v08 data types)
Begin upgrading: version: V0, upgrading: V001
Write header: version: V0, upgrading: V001
tree raft_state not found
tree raft_log not found
Upgraded 0 records
Finished upgrading: version: V001, upgrading: None
Write header: version: V001, upgrading: None
Upgrade on-disk data
    From: V001(2023-05-15: Get rid of compat, use only openraft v08 data types)
    To:   V002(2023-07-22: Store snapshot in a file)
Begin upgrading: version: V001, upgrading: V002
Write header: version: V001, upgrading: V002
tree raft_state not found
tree raft_log not found
Found state machine trees: []
Found min state machine id: 18446744073709551615
No state machine tree, skip upgrade
Finished upgrading: version: V002, upgrading: None
Write header: version: V002, upgrading: None
Wait for 180s for active leader...
Leader Id: 0
    Metrics: id=0, Leader, term=1, last_log=Some(3), last_applied=Some(1-0-3), membership={log_id:Some(1-0-3), voters:[{0:{EmptyNode}}], learners:[]}

Register this node: {id=0 raft=colima:28004 grpc=}

    Register-node: Ok

Databend Metasrv started

==> /tmp/std-query.log <==
Databend Query

Version: v1.2.287-nightly-8930689add(rust-1.75.0-nightly-2024-01-07T22:05:46.363097970Z)

Logging:
    file: enabled=true, level=INFO, dir=/var/log/databend, format=json
    stderr: enabled=false(To enable: LOG_STDERR_ON=true or RUST_LOG=info), level=WARN, format=text
    otlp: enabled=false, level=INFO, endpoint=http://127.0.0.1:4317, labels=
    query: enabled=false, dir=, otlp_endpoint=, labels=
    tracing: enabled=false, capture_log_level=INFO, otlp_endpoint=http://127.0.0.1:4317
Meta: connected to endpoints [
    "0.0.0.0:9191",
]
Memory:
    limit: unlimited
    allocator: jemalloc
    config: percpu_arena:percpu,oversize_threshold:0,background_thread:true,dirty_decay_ms:5000,muzzy_decay_ms:5000
Cluster: standalone
Storage: s3 | bucket=databend,root=,endpoint=http://127.0.0.1:9000
Cache: none
Builtin users: databend

Admin
    listened at 0.0.0.0:8080
MySQL
    listened at 0.0.0.0:3307
    connect via: mysql -u${USER} -p${PASSWORD} -h0.0.0.0 -P3307
Clickhouse(http)
    listened at 0.0.0.0:8124
    usage:  echo 'create table test(foo string)' | curl -u${USER} -p${PASSWORD}: '0.0.0.0:8124' --data-binary  @-
echo '{"foo": "bar"}' | curl -u${USER} -p${PASSWORD}: '0.0.0.0:8124/?query=INSERT%20INTO%20test%20FORMAT%20JSONEachRow' --data-binary @-
Databend HTTP
    listened at 0.0.0.0:8000
    usage:  curl -u${USER} -p${PASSWORD}: --request POST '0.0.0.0:8000/v1/query/' --header 'Content-Type: application/json' --data-raw '{"sql": "SELECT avg(number) FROM numbers(100000000)"}'
```

</StepContent>
<StepContent number="3" title="连接到 Databend">

在此步骤中，您将使用 BendSQL CLI 工具建立与 Databend 的连接。有关如何安装和操作 BendSQL 的说明，请参见 [BendSQL](../../../30-sql-clients/00-bendsql/index.md)。

1. 使用 SQL 用户（databend/databend）建立与 Databend 的连接，请运行以下命令：

```shell
❯ bendsql -udatabend -pdatabend
Welcome to BendSQL 0.12.1-homebrew.
Connecting to localhost:8000 as user databend.
Connected to DatabendQuery v1.2.287-nightly-8930689add(rust-1.75.0-nightly-2024-01-07T22:05:46.363097970Z)

databend@localhost:8000/default>
```

2. 要验证部署，您可以使用 BendSQL 创建一个表并插入一些数据：

```shell
databend@localhost:8000/default> CREATE DATABASE test;
0 row written in 0.042 sec. Processed 0 row, 0 B (0 row/s, 0 B/s)

databend@localhost:8000/default> use test;
0 row read in 0.028 sec. Processed 0 row, 0 B (0 row/s, 0 B/s)

databend@localhost:8000/test> CREATE TABLE mytable(a int);
0 row written in 0.053 sec. Processed 0 row, 0 B (0 row/s, 0 B/s)

databend@localhost:8000/test> INSERT INTO mytable VALUES(1);
1 row written in 0.108 sec. Processed 1 row, 5 B (9.27 row/s, 46 B/s)

databend@localhost:8000/test> INSERT INTO mytable VALUES(2);
1 row written in 0.102 sec. Processed 1 row, 5 B (9.81 row/s, 49 B/s)

databend@localhost:8000/test> INSERT INTO mytable VALUES(3);
1 row written in 0.120 sec. Processed 1 row, 5 B (8.33 row/s, 41 B/s)

databend@localhost:8000/test> select * from mytable;
┌─────────────────┐
│        a        │
│ Nullable(Int32) │
├─────────────────┤
│               1 │
│               2 │
│               3 │
└─────────────────┘
3 rows read in 0.066 sec. Processed 3 rows, 15 B (45.2 rows/s, 225 B/s)
```

由于表数据存储在桶中，您将注意到桶大小从 0 开始增加。

![Alt text](@site/docs/public/img/deploy/minio-deployment-verify.png)

</StepContent>
</StepsWrap>

## 部署本地 Databend {#deploying-a-local-databend}

以下步骤将指导您完成本地部署 Databend 的过程。

<StepsWrap>
<StepContent number="1" title="下载 Databend">

1. 从 [下载](/download) 页面下载适合您平台的 Databend 安装包。

2. 将安装包解压到本地目录，并进入解压后的目录。

</StepContent>
<StepContent number="2" title="启动 Databend">

1. 配置管理员用户。您将使用此账户连接到 Databend。有关更多信息，请参见 [配置管理员用户](../../04-references/01-admin-users.md)。作为示例，取消在 **configs** 文件夹的 `databend-query.toml` 中以下行的注释，以启用 `root` 账户：

```sql title="databend-query.toml"
[[query.users]]
name = "root"
auth_type = "no_password"
```

2. 打开终端并导航到存储已解压文件和文件夹的文件夹。

3. 运行位于 **scripts** 文件夹中的脚本 **start.sh**：
   
   MacOS 可能会提示错误“_databend-meta 无法打开，因为 Apple 无法检查其是否包含恶意软件。_”。要继续，请在您的 Mac 上打开 **系统设置**，在左侧菜单中选择 **隐私与安全**，然后在右侧的 **安全性** 部分为 databend-meta 点击 **仍要打开**。对于 databend-query 上的错误也执行相同操作。

    ```shell
    ./scripts/start.sh
    ```

:::tip
如果在尝试启动 Databend 时遇到以下错误消息：

```shell
==> query.log <==
: No getcpu support: percpu_arena:percpu
: option background_thread currently supports pthread only
Databend Query start failure, cause: Code: 1104, Text = failed to create appender: Os { code: 13, kind: PermissionDenied, message: "Permission denied" }.
```

运行以下命令，然后再次尝试启动 Databend：

```shell
sudo mkdir /var/log/databend
sudo mkdir /var/lib/databend
sudo chown -R $USER /var/log/databend
sudo chown -R $USER /var/lib/databend
```
:::

4. 运行以下命令以验证 Databend 是否已成功启动：

```shell
ps aux | grep databend

---
eric             12789   0.0  0.0 408495808   1040 s003  U+    2:16pm   0:00.00 grep databend
eric             12781   0.0  0.5 408790416  38896 s003  S     2:15pm   0:00.05 bin/databend-query --config-file=configs/databend-query.toml
eric             12776   0.0  0.3 408654368  24848 s003  S     2:15pm   0:00.06 bin/databend-meta --config-file=configs/databend-meta.toml
```

</StepContent>
<StepContent number="3" title="连接到 Databend">

在此步骤中，您将使用 BendSQL CLI 工具建立与 Databend 的连接。你也可以参考 [BendSQL](../../../30-sql-clients/00-bendsql/index.md) 获得关于 BendSQL 的更多相关信息。

1. 从 [下载](/download) 页面下载适合您平台的 BendSQL 安装包。

2. 将安装包解压到本地目录，并进入解压后的目录。

3. 要与本地 Databend 建立连接，请执行以下命令：

```shell
❯ ./bendsql 
Welcome to BendSQL 0.13.3-25b1195(2024-03-01T11:33:39.167314799Z).
Connecting to localhost:8000 as user root.
Connected to Databend Query v1.2.371-a95ac62303(rust-1.77.0-nightly-2024-03-11T01:07:23.093484068Z)

root@localhost:8000/default>
```

4. 查询 Databend 版本以验证连接：

```sql
root@localhost:8000/default> SELECT VERSION();

SELECT
  VERSION()

┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                        version()                                       │
│                                         String                                         │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ Databend Query v1.2.371-a95ac62303(rust-1.77.0-nightly-2024-03-11T01:07:23.093484068Z) │
└────────────────────────────────────────────────────────────────────────────────────────┘
1 row read in 0.013 sec. Processed 1 row, 1 B (74.7 row/s, 74 B/s)
```

</StepContent>

<StepContent number="4" title="停止 Databend">

1. 打开终端并导航到在第 2 步中解压的 Databend 相关文件夹。

2. 运行位于 **scripts** 文件夹中的脚本 **stop.sh**：

    ```shell
    # 该脚本使用了 `killall` 命令，若您尚未安装该命令，请安装适用于您系统环境的 [`psmisc`](https://gitlab.com/psmisc/psmisc) 包。
    # 以 CentOS 系统为例：`yum install psmisc` 。
    ./scripts/stop.sh
    ```

</StepContent>
</StepsWrap>

## 下一步

部署 Databend 后，您可能需要了解以下主题：

- [加载与卸载数据](/guides/load-data)：在 Databend 中管理数据的导入/导出。
- [可视化](/guides/visualize)：将 Databend 与可视化工具集成以获得洞察。
