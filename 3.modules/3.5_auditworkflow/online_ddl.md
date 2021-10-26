# Online DDL

在 MySQL5.6 前，通过 Alter 语句直接修改表结构，如果表的数据量较大，可能造成长时间锁表的情况。为了解决这个问题，业界常用的做法是使用 Online DDL 的方式（即在修改表结构过程中，业务照常能够访问对应的表）。

通过集成优秀的开源 Online DDL 工具 [gh-ost](https://github.com/github/gh-ost)，SQLE 实现了 Online DDL 功能。

## 配置

在使用 SQLE 提供的 Online DDL 功能前，需要进行下述配置。

### 配置规则

Online DDL 对应的规则在**规则列表** -> **全局配置**中。对应的规则名为『**改表时，表空间超过指定大小(MB)时使用gh-ost上线**』。通过规则的 value 值，可以动态调整触发 Online DDL 的阈值（调整方式见[规则模板管理](../3.3_template/rule_template_management.md)）。

### 配置 gh-ost 参数

SQLE 根据 gh-ost 的官方推荐，使用了如下默认配置（相应配置说明见[官方文档](https://github.com/github/gh-ost/blob/master/doc/command-line-flags.md)）：
```sh
mysql_timeout=0.0
assume_master_host=""
master_user=""
master_password=""
exact_rowcount=false
concurrent_rowcount=true
allow_on_master=false
allow_master_master=false
allow_nullable_unique_key=false
approve_renamed_columns=false
skip_renamed_columns=false
tungsten=false
discard_foreign_keys=false
skip_foreign_key_checks=false
;skip_strict_mode=false //fixme
aliyun_rds=false
gcp=false
azure=false
test_on_replica=false
test_on_replica_skip_replica_stop=false
migrate_on_replica=false
ok_to_drop_table=false
initially_drop_old_table=false
initially_drop_ghost_table=false
timestamp_old_table=false
cut_over=""
force_named_cut_over=false
force_named_panic=false
switch_to_rbr=false
assume_rbr=false
cut_over_exponential_backoff=false
exponential_backoff_max_interval=64
chunk_size=1000
dml_batch_size=10
default_retries=120
cut_over_lock_timeout_seconds=3
nice_ratio=0
max_lag_millis=1500
throttle_control_replicas=""
throttle_query=""
throttle_http=""
ignore_http_errors=false
heartbeat_interval_millis=100
throttle_flag_file=""
throttle_additional_flag_file="/tmp/gh-ost.throttle"
postpone_cut_over_flag_file=""
panic_flag_file=""
initially_drop_socket_file=false
serve_socket_file="/tmp/gh-ost.${库名}.${表明}.sock"
serve_tcp_port=0
replica_server_id=${随机数方式重复}
max_load="Threads_running=80,Threads_connected=1000"
critical_load="Threads_running=80,Threads_connected=1000"
critical_load_interval_millis=0
critical_load_hibernate_seconds=0
force_table_names=""
```

可以根据自己的需求自定义配置。配置文件的位置在 `/opt/sqle/etc/gh-ost.ini`。注意单词间使用下划线分割。

### 其他配置
1. 开启数据源 binlog，并设置 binlog 格式为 ROW
2. 如果数据源为单实例或者从实例，需要在配置文件里指定**allow_on_master=true**

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
