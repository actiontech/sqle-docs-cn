# 数据库审核插件开发
## 一、本篇文章的目标
1. 你可以新建一个数据库审核插件；
2. 你可以编写数据库审核规则。

## 二、开始
下文以 PostgreSQL 审核插件为例，介绍如何从头开发 SQLE 的数据库审核插件，并添加一条规则；示例忽略SQL解析，数据库的查询和执行等细节，需要更详细的示例可以参考我们开源的 PostgreSQL 数据库审核插件：[sqle-pg-plugin](https://github.com/actiontech/sqle-pg-plugin) 。

### 1. 引入SQLE插件库
```bash
go get github.com/actiontech/sqle@v1.2111.0-pre1 # 此版本为该文档编辑时的最新版本
```
``` go
import (
	"github.com/actiontech/sqle/sqle/driver"
	"github.com/actiontech/sqle/sqle/model"
)
```
1. `github.com/actiontech/sqle/sqle/driver` 导入插件的接口，初始化函数，参考附录；
2. `github.com/actiontech/sqle/sqle/model` 导入审核规则的定义。

### 2. 定义插件的注册信息
定义一个结构体`registererImpl`实现 `driver.Registerer` 接口，该接口对应的方法将插件名和该数据库插件的审核规则注册到SQLE，接口说明参考附录。
```go
type registererImpl struct{}

func (s *registererImpl) Name() string {
    return "PostgreSQL"
}

const (
    RuleTypeDMLConvention = "DML规范"
)

func (s *registererImpl) Rules() []*model.Rule {
    return []*model.Rule{
        &model.Rule{
            Name:      "pg_rule_1",             // 定义规则的名称，该名称全局唯一；
            Desc:      "不建议使用select *",     // 规则的描述，会展示在SQLE的规则页面上；
            Level:     model.RuleLevelNotice,   // 规则的默认影响级别，可以在配置规则模板时灵活修改；
            Typ:       RuleTypeDMLConvention,   // 规则的分类，可以自由定义，页面的规则分类会按此进行展示；
            IsDefault: true,                    // 是否加入默认规则模板。为true时，SQLE平台的默认规则模板会加入该条规则。
        },
    }
}
```
1. 定义该插件名称`PostgreSQL`，该名称会展示在SQLE页面上，在添加数据库或者审核时可以选择；
2. 定义一个审核规则 `不建议使用select *`, 可以定义多个规则，通过`Rules` 函数返回即可。

### 3. 实现审核驱动
定义一个结构体`driverImpl`实现 `driver.Driver` 接口，接口说明参考附录。
```go
type driverImpl struct {
	cfg         *driver.Config 
	// 省略其他字段 
}

func NewDriver(cfg *driver.Config) driver.Driver {
	return &driverImpl{
		cfg:    cfg, 
		// 省略其他字段
	}
}

// Audit 接收从SQLE传递的SQL语句，输出审核计划 `driver.AuditResult`
func (i *driverImpl) Audit(ctx context.Context, sql string) (*driver.AuditResult, error) {
    result := driver.NewInspectResults() // 新建审核建议
    
    // i.cfg.Rules 为本次审核开启的审核规则
    for _, rule := range i.cfg.Rules {
        switch rule.Name {
        case "pg_rule_1": // "不建议使用select *"
            
            // 假设这里做了 select * 的检查
            
            result.Add(rule.Level, "不建议使用select *") // 将”不建议使用select *“加入到审核建议内
        }
    }
    return result, nil
}
```
1. 定义`driverImpl`结构体实现 `Driver` 接口，上述实例中仅展示`Driver`的`Audit`的接口，用来介绍如何实现一个基于规则的审核

### 4. 注册插件
```go
func main() {
	driver.ServePlugin(&registererImpl{}, NewDriver)
}
```

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

### 3. 插件的配置信息说明
```go
type Config struct {
	IsOfflineAudit bool
	Schema         string
	Inst           *model.Instance
	Rules          []*model.Rule
}
```
1. `IsOfflineAudit`: 是否使用静态审核，此时数据源为空；
2. `Schema`: 当前审核在此 Schema 下；
3. `Inst`: 数据源信息, 待审核的数据库；
4. `Rules`: 本次审核制定的规则列表。

### 4. 初始化函数说明
插件的主进程入口，由插件的 main 函数调用即可实现插件
```go
func ServePlugin(r Registerer, newDriver func(cfg *Config) Driver)
```
1. `r`: 传入 Registerer 的接口实现, 由插件侧实现； 
2. `newDriver`: 传入 Driver 的初始化函数，该函数的入参是 `Config` 是由 SQLE 向插件传递的配置信息，函数的出参是 Driver 的接口实现，由插件侧实现。

