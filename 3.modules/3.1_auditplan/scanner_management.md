# Scanner管理

**Scanner** 是**审核计划管理**中负责解析并上传 SQL 的组件。

不同场景下的 **Scanner** 会有不同的解析行为，但它们最终都需要通过统一的 API 接口将 SQL 上传到 SQLE。

## MyBatis Scanner

### 概述
**MyBatis Scanner** 通过指定代码目录，扫描并解析目录中的 MyBatis XML 文件得到 SQL 语句。**MyBatis Scanner** 将解析出的 SQL 上传至 SQLE Server 后，触发审核并得到审核结果。如果审核结果中包含 Error 级别的错误（指触发了 Error 级别的[审核规则](TODO)），则将退出码（Exit Code）置为非 0。

### 场景
常见的使用场景是将 **MyBatis Scanner** 与 CI/CD 集成。通过持续的审核代码仓库中的 SQL 以及变更，可以提早发现问题。

### 使用
由于 Scanner 是**审核计划管理**下的功能组件，所以在使用前需要创建审核计划，具体操作步骤见[审核计划](./auditplan.md)。

Scanner 打包在 SQLE RPM 中。在部署完 SQLE Server后（部署方式见[快速开始](TODO)）, Scanner 的二进制放在 `${SQLE的工作目录}/bin` 目录下。如下：

```sh
[root@sqle-server bin]# pwd
/opt/sqle/bin
[root@sqle-server bin]# ll
total 62816
-rwxr-x--- 1 actiontech-universe actiontech 25064378 Sep 27 13:16 scannerd
-rwxr-x--- 1 actiontech-universe actiontech 39254854 Sep 27 13:16 sqled
```

将目录中的 scannerd 文件拷贝至 CI/CD 服务器上，进行相应的配置。不同的 CI/CD（如 GoCD、Jenkins）配置方式不同，具体可参考对应的官方文档，这里不逐一介绍。

下面介绍 Scanner 相关参数：

```sh
[root@sqle-server bin]# ./scannerd --help
SQLE Scanner

Usage:
  SQLE [command]

Available Commands:
  help        Help about any command
  mybatis     Parse MyBatis XML file
  slowquery   Parse slow query

Flags:
  -h, --help           help for SQLE
  -H, --host string    sqle host (default "127.0.0.1")
  -N, --name string    audit plan name
  -P, --port string    sqle port (default "10000")
  -A, --token string   sqle token

Use "SQLE [command] --help" for more information about a command.
```

可以看到 scannerd 二进制支持两种不同类型的 Scanner，分别是 mybatis 和 slowquery。由于都需要上传 SQL 到 SQle Server，所以它们有一些公共参数：
* host：SQLE Server 所在的主机 IP 地址（默认是当前主机）
* port：SQLE Server 提供 HTTP 服务的端口（默认是 10000）
* name：审核计划名（表示 Scanner 将 SQL 上传至哪个审核计划的 **SQL 池**）
* token：审核计划上传凭证（具体值可到审核计划列表页中**访问凭证**列中获取）

除了上面的公共参数，MyBatis Scanner 还需要一个独有的参数：

```sh
[root@sqle-server bin]# ./scannerd mybatis --help
Parse MyBatis XML file

Usage:
  SQLE mybatis [flags]

Flags:
  -D, --dir string   xml directory
  -h, --help         help for mybatis

Global Flags:
  -H, --host string    sqle host (default "127.0.0.1")
  -N, --name string    audit plan name
  -P, --port string    sqle port (default "10000")
  -A, --token string   sqle token
```

* dir: 通常 CI/CD 会将项目代码拉取至本地做集成，这个需要填写的是项目拉取到本地的目录

## SlowQuery Scanner（企业版功能）

### 概述

**SlowQuery Scanner** 通过指定慢日志文件，扫描并解析文件中的慢 SQL 语句。**SlowQuery Scanner** 将解析出的 SQL 上传至 SQLE Server 通过。通过审核计划中配置的 Cron 定时触发审核。

### 场景
常见的使用场景是 DBA 发现运维中的数据库产生了慢日志，想知道慢日志中是否有不符合数据库使用规范的 SQL。这时，就可以使用 **SlowQuery Scanner** 扫描并审核指定的慢日志。

一旦 **SlowQuery Scanner** 启动，它会全量解析慢日志文件，在**审核计划详情**页面可以看到它上传的 SQL，这时可以点击页面上的**立即审核**按钮手工触发一次审核。

与 **MyBatis Scanner** 不同的是，**SlowQuery Scanner** 是一个常驻进程，它会持续的监控 Scanner 启动时指定的慢日志文件，一旦产生新的慢 SQL，它会增量解析并上传这些 SQL。通过审核计划配置的 Cron 表达式，可以对 SQL 池定时触发审核。

### 使用

**SlowQuery Scanner** 的使用方式与 **MyBatis Scanner** 类似。需要指定一些公共参数（公共参数的设置见**MyBatis Scanner**节，这里不再赘述）和独有的参数。

```sh
[root@sqle-server bin]# ./scannerd slowquery --help
Parse slow query

Usage:
  SQLE slowquery [flags]

Flags:
  -h, --help              help for slowquery
      --log-file string   log file absolute path

Global Flags:
  -H, --host string    sqle host (default "127.0.0.1")
  -N, --name string    audit plan name
  -P, --port string    sqle port (default "10000")
  -A, --token string   sqle token
```

* log-file：慢日志文件的绝对路径

**SlowQuery Scanner** 启动后进程不会退出，即默认以增量解析方式启动。这种情况下，需要将二进制以后台进程方式启动。