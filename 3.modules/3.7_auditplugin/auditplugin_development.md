# 数据库审核插件开发
## 一、需要具备的能力
1. 简单 golang 语言
2. 对 SQL 审核规则有自己的理解

## 二、开始
### 1. 引入
本篇以 PostgreSQL 审核插件为例，介绍如何从头开发 SQLE 的审核插件（ps：PG 插件是我们开源的一个 SQLE 插件范例，感兴趣的可以上 GitHub [sqle-pg-plugin](https://github.com/actiontech/sqle-pg-plugin) 查看）。

总的来说，实现一个审核插件只需 3 步：
## 附录
### 1. 注册接口说明
该接口定义了该插件的名称和实现的规则，SQLE启动的时候会调用该接口获取插件名称和该插件支持的规则列表。
```go
type Registerer interface {
	Name() string
	Rules() []*model.Rule
}
```
1. `Name`: 插件名，最终会展示在 SQLE 页面的数据源类型的下拉框中；
2. `Rules`: 插件支持的规则，在启动 SQLE 时，会调用插件获取这些规则，你将在规则模板内看到它们。

### 2. 审核接口说明
该接口定义了SQLE进行审核时，由插件完成的和具体数据库底层交互的操作
```go
type Driver interface {
	Close(ctx context.Context)
	Ping(ctx context.Context) error
	Exec(ctx context.Context, query string) (driver.Result, error)
	Tx(ctx context.Context, queries ...string) ([]driver.Result, error)
	Schemas(ctx context.Context) ([]string, error)
	Parse(ctx context.Context, sqlText string) ([]Node, error)
	Audit(ctx context.Context, sql string) (*AuditResult, error)
	GenRollbackSQL(ctx context.Context, sql string) (string, string, error)
}
```
1. `Close`: 关闭审核插件使用的相关资源，通常是完成一次审核后，关闭数据库连接等资源;
2. `Ping`: 检测数据库的连接性,通常在添加数据源时，为了检测填写的数据是否正确，会调用此方法;
3. `Exec`: 执行 SQL 上线时执行此方法；
4. `Tx`: 执行 SQL 上线时执行此方法，一般当SQL是DML时且需要事务执行时会批量执行SQL；
5. `Schemas`: 返回审核插件展示给用户的 Schema 列表；
6. `Parse`: 解析审核插件支持的 SQL 格式；
7. `Audit`: 根据指定的SQL语句生成审核建议；
8. `GenRollbackSQL`: 生成 SQL 的回滚语句。

### 3. 初始化函数说明

```go
func ServePlugin(r Registerer, newDriver func(cfg *Config) Driver)
```
实现一个 Driver 实例的构造函数，函数签名为 `func(*driver.Config) driver.Driver`。`driver.Config` 是 SQLE 在准备调用插件前传给插件的一些必要参数：

```go
type Config struct {
    // 是否是静态审核。
	IsOfflineAudit bool
    // 当前审核在此 Schema 下。
	Schema         string
    // 用户选择的数据源信息。
	Inst           *model.Instance
    // 规则列表。不同数据源的规则列表可能不相同。
	Rules          []*model.Rule
}
```

### 第 3 步：注册插件

在 main 函数中调用插件层的 `ServePlugin()` 函数。该函数第一个参数传入 Registerer 接口的实现；第二个参数传入第 2 步中定义的构造函数。