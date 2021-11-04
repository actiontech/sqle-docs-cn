# 审核插件开发

本篇以 PostgreSQL 审核插件为例，介绍如何从头开发 SQLE 的审核插件（ps：PG 插件是我们开源的一个 SQLE 插件范例，感兴趣的可以上 GitHub [sqle-pg-plugin](https://github.com/actiontech/sqle-pg-plugin) 查看）。

总的来说，实现一个审核插件只需 3 步：

## 第 1 步：实现 Registerer 接口
```go
type Registerer interface {
    // 插件名。最终会展示在 SQLE 页面的数据源类型的下拉框中。
	Name() string

    // 插件支持的规则。在启动 SQLE 时，会调用插件拿到这些规则，并存储在 SQLE 的数据库中。
	Rules() []*model.Rule
}
```

## 第 2 步：实现 Driver 接口及其构造函数
```go
type Driver interface {
    // 关闭审核插件使用的相关资源。通常是完成一次审核后，关闭数据库连接等资源。
	Close(ctx context.Context)
    // 检测数据库的连接性。通常在添加数据源时，为了检测填写的数据是否正确，会调用此方法。
	Ping(ctx context.Context) error
    // 执行 DDL 上线时执行此方法。
	Exec(ctx context.Context, query string) (driver.Result, error)
    // 执行 DML 上线时在一个事物中执行此方法。
	Tx(ctx context.Context, queries ...string) ([]driver.Result, error)
    // 返回审核插件展示给用户的 Schema 列表。
	Schemas(ctx context.Context) ([]string, error)
    // 解析审核插件支持的 SQL 格式。
	Parse(ctx context.Context, sqlText string) ([]Node, error)
    // 审核 SQL 并给出审核建议。
	Audit(ctx context.Context, sql string) (*AuditResult, error)
    // 生成 SQL 的回滚语句。
	GenRollbackSQL(ctx context.Context, sql string) (string, string, error)
}
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

## 第 3 步：注册插件

在 main 函数中调用插件层的 `ServePlugin()` 函数。该函数第一个参数传入 Registerer 接口的实现；第二个参数传入第 2 步中定义的构造函数。