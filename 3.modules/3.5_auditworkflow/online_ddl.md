# Online DDL

在 MySQL5.6 前，通过 Alter 语句直接修改表结构，如果表的数据量较大，可能造成长时间锁表的情况。为了解决这个问题，业界常用的做法是使用 Online DDL 的方式（即在修改表结构过程中，业务照常能够访问对应的表）。

通过集成优秀的开源 Online DDL 工具 [gh-ost](https://github.com/github/gh-ost)，SQLE 实现了 Online DDL 功能。

## 配置

在使用 SQLE 提供的 Online DDL 功能前，需要进行下述配置。

### 配置规则

Online DDL 对应的规则在**规则列表** -> **全局配置**中。对应的规则名为『**改表时，表空间超过指定大小(MB)时使用gh-ost上线**』。通过规则的 value 值，可以动态调整触发 Online DDL 的阈值（调整方式见[规则模板管理](../3.3_template/rule_template_management.md)）。

### 配置 gh-ost 参数
SQLE 根据 gh-ost 的官方推荐，设置了默认的参数，参数文件的位置在 `${working directory of SQL}/etc/gh-ost.ini`。当然，也可以根据自己的需求自定义修改该文件。

gh-ost [迁移模式](https://github.com/github/gh-ost/blob/master/doc/cheatsheet.md#cheatsheet)有三种：
1. connect to replica(官方推荐方式)
2. connect to master
3. migrate/test on replica

SQLE 默认使用第一种方式。这种情况下，如果数据源是单实例或者从实例，需要在配置文件里指定**allow_on_master=true**。

## 诊断

如果一条 SQL 正在通过 Online DDL 上线，通常可以通过以下方式诊断上线过程。

### SQLE 日志
通过 SQLE 的日志文件可以看到 gh-ost 执行完整流程，如下面是 dry-run 流程的部分输出日志：
```
time="2021-10-26T11:01:27+08:00" level=info msg="dry-run gh-ost" task_id=12 thread_id=50N
time="2021-10-26T11:01:27+08:00" level=info msg="--alter: ADD COLUMN `c10` INT" alter="alter table sbtest1 add column c10 int;" host=127.0.0.1 onlineddl=gh-ost port=4406 task_id=12 thread_id=50N
time="2021-10-26T11:01:27+08:00" level=info msg="Migrating `test`.`sbtest1`" alter="alter table sbtest1 add column c10 int;" host=127.0.0.1 onlineddl=gh-ost port=4406 task_id=12 thread_id=50N
...
time="2021-10-26T11:01:27+08:00" level=info msg="Done migrating `test`.`sbtest1`" alter="alter table sbtest1 add column c10 int;" host=127.0.0.1 onlineddl=gh-ost port=4406 task_id=12 thread_id=50N
time="2021-10-26T11:01:27+08:00" level=info msg="Removing socket file: /tmp/gh-ost.test.sbtest1.sock" alter="alter table sbtest1 add column c10 int;" host=127.0.0.1 onlineddl=gh-ost port=4406 task_id=12 thread_id=50N
time="2021-10-26T11:01:27+08:00" level=info msg="Tearing down inspector" alter="alter table sbtest1 add column c10 int;" host=127.0.0.1 onlineddl=gh-ost port=4406 task_id=12 thread_id=50N
time="2021-10-26T11:01:27+08:00" level=info msg="Tearing down applier" alter="alter table sbtest1 add column c10 int;" host=127.0.0.1 onlineddl=gh-ost port=4406 task_id=12 thread_id=50N
time="2021-10-26T11:01:27+08:00" level=info msg="Tearing down streamer" alter="alter table sbtest1 add column c10 int;" host=127.0.0.1 onlineddl=gh-ost port=4406 task_id=12 thread_id=50N
time="2021-10-26T11:01:27+08:00" level=info msg="Tearing down throttler" alter="alter table sbtest1 add column c10 int;" host=127.0.0.1 onlineddl=gh-ost port=4406 task_id=12 thread_id=50N
time="2021-10-26T11:01:27+08:00" level=info msg="dry-run OK!" task_id=12 thread_id=50N
```

### Systemd 日志
通过 Systemd 的日志管理器可以看到 gh-ost 执行迁移的进度：

```
journalctl -u sqled
```
