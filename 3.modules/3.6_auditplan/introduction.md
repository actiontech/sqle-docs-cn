# 审核任务介绍
## 一、背景
在[审核工单](../3.3_auditworkflow/overview.md)中我们介绍了如何通过 SQLE 进行 SQL 审核并上线的流程。在这流程中，通常是审核一些 DDL，如建库（create database）、建表（create table）和改表（alter table）等语句。
审核工单管理，主要解决 SQL 上线的规范化流程化的问题，它能够帮助 DBA 自动化处理整个 SQL 上线过程中一些重复繁琐的工作。

**审核工单管理**也有它的局限性。

* 第一，通常**审核工单管理** 中流转的是 DDL 一类的 SQL，它们的上线通常是一次性操作。DDL 上线后，通常还会有业务 SQL（通常是 DML）访问数据库。这时可能会遇到一些执行效率较低的业务 SQL 造成数据库的性能问题。这类 SQL 是**审核工单管理**无法覆盖到的。
* 第二，通常在使用**审核工单管理**审核需要上线的 SQL 时，项目已经临近发版，功能都已经实现完毕。如果这时审核出 SQL 存在一些问题，是否修复这些问题，可能会收到很多因素的影响（如 SQL 问题的影响面大小，项目发版的紧急程度等）。虽然这类 SQL **审核工单管理**可以覆盖到，但因为外部因素，会导致问题 SQL 的存在。

SQLE 使用**审核任务**来解决上面的两个问题。

**审核任务**是基于 Cron 的 SQL 自动审核系统。一个**审核任务**的成功运行需要两方参与，分别是 SQLE Server 和 **Scanner**。在 SQLE Server 中创建审核任务。Scanner 将 SQL 上传到 SQLE Server 对应的审核任务的 **SQL 池**，SQLE 根据配置对 SQL 池触发审核，并生成**审核报告**。

## 二、审核任务介绍
### 分类
|类型|支持的数据库|SQL采集模式|
|---|---|---|
|库表元数据|MySQL|SQLE 自动抓取|
|TopSQL|Oracle|SQLE 自动抓取|
|慢日志|MySQL|Scanner 抓取|
|Mybatis扫描|ALL|Scanner 抓取|
|应用程序SQL抓取|ALL|OpenAPI 推送|
|自定义|ALL|OpenAPI 推送|

### SQL采集模式
#### 1. SQLE 自动抓取
由 SQLE 后台服务定期抓取，创建完审核任务后无需再做其他操作，即可再界面看到已经采集的SQL并进行审核。
#### 2. Scanner 抓取
scanner 是SQLE支持的实现对特定场景进行SQL采集的一个客户端工具，通过调用 SQLE 的OpenAPI将SQL推送到SQLE，例如：scanner支持扫描MySQL慢日志
并实时解析文件中的慢SQL将SQL推送到SQLE。具体的使用参考：[Scanner 使用说明](./scanner_management.md)
#### 3. OpenAPI 推送
SQLE 对外提供标准的OpenAPI接口，任何人都可以通过调用接口来将特定场景的SQL收集并推送到SQLE，来使用审核任务的审核能力。用户可以自行实现scanner（可以是任何形式，bash脚本、python程序等）将SQL推送到审核任务类型为“自定义”的审核任务中并进行审核。
