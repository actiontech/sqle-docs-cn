# 数据库审核插件开发

在本篇文档中，会介绍如何开发一个数据库审核插件，分为**快速开始部分**和**详细部分**。如果你是一个对 Go 语言不太了解的人，可以先通过**快速开始部分**的文档，实现一个简单的数据库审核插件。当你对插件开发有了一定的了解之后，可以通过**详细部分**的文档，通过更多自定义的方式，实现更加复杂的数据库审核插件。

## 快速开始部分

### 一、前置

#### 1. 语法
由于 SQLE 是一个用 Go 语言开发的开源项目，如果你对 Go 语言完全不了解，需要先了解 Go 语言的基础语法，建议使用官方的 Go 语言快速开始文档（[Go Tour](https://tour.golang.org/welcome/1) 或者 [Go 语言之旅](https://tour.go-zh.org/welcome/1)），如果你已经了解了 Go 语言的基础语法，可以直接跳过本部分。

#### 2. 包管理
SQLE 插件是一个独立于 SQLE 的进程，所以编写插件的方式与开发一个新的 Go 语言项目并没有什么差异。Go 语言项目使用 Go Modules 来管理包，所以在开始之前你需要先了解一下 Go Modules 的使用方法。建议参考[这篇文档](https://golang.org/doc/tutorial/getting-started)，文档中大致介绍了如果开始一个新的 Go 语言项目，并通过 Go Modules 的方式来调用其他项目的包，如果你已经了解了 Go Modules 的使用方法，可以直接跳过本部分。

#### 3. Go 语言项目构建


### 二、编写插件

这一小节会假设你已经创建了一个由 Go Modules 管理的审核插件项目。下面开始介绍插件核心代码的开发。

#### 1. 引入默认插件库

SQLE 为了方便插件的开发，在自身的插件层之上做了一层封装（Adaptor），插件开发者可以使用这个封装库来快速的开发一个数据库审核插件。这个封装库默认实现了 PostgreSQL、Oracle 与 SQL Server 的插件。

在开发之前，你需要先引入两个 SQLE 的插件库：
1. github.com/actiontech/sqle/sqle/driver
2. github.com/actiontech/sqle/sqle/pkg/driver

第一个库定义了插件规则的结构体，在你编写插件规则时需要用到这个库。第二个库实现了一些默认的插件，在引入这个库后，只需要实现相应的规则与规则处理函数即可。

假设你想要实现一个 SQL Server 的审核插件。在 main 函数中创建一个空的 SQL Server 审核插件，这时你的 main 文件应该是这样的：
```Go
package main

import (
	"github.com/actiontech/sqle/sqle/driver"
	adaptor "github.com/actiontech/sqle/sqle/pkg/driver"
)

func main() {
	plugin := adaptor.NewAdaptor(&adaptor.MssqlDialector{})
}
```

#### 2. 编写插件规则与规则处理函数
假设你需要实现一个规则，该规则检查 SQL 是否使用了 select *。

定义规则如下：

```Go
Rule{
	Name:     "aviod_select_all_column", # 规则ID，该值会与插件类型一起作为这条规则在 SQLE 的唯一标识
	Desc:     "避免查询所有的列", # 规则描述
	Category: "DQL规范", # 规则分类，用于分组，相同类型的规则会在 SQLE 的页面上展示在一起
	Level:    driver.RuleLevelError, # 规则等级，表示该规则的严重程度。在插件注册阶段，会使用所有 RuleLevelError 级别的规则创建一个默认的规则模板。
}
```

规则的处理函数如下：

```Go
func(ctx context.Context, rule *driver.Rule, sql string) (string, error) {
	if strings.Contains(sql, "select *") {
		return rule.Desc, nil
	}
	return "", nil
}
```
这里为了演示，这个处理函数只是简单的使用了字符串匹配的方式，你也可以使用正则或者 AST 语法树的方式来检查 SQL 语句（AST 的方式会在下文中介绍）。

最后将插件规则与规则处理函数通过 `plugin.AddRule()` 函数注册到 SQLE 中，注册完成后，你的 main 文件应该是这样的：
```Go
package main

import (
	"github.com/actiontech/sqle/sqle/driver"
	adaptor "github.com/actiontech/sqle/sqle/pkg/driver"
)

func main() {
	plugin := adaptor.NewAdaptor(&adaptor.MssqlDialector{})
	aviodSelectAllColumn := &driver.Rule{
		Name:     "aviod_select_all_column",
		Desc:     "避免查询所有的列",
		Category: "DQL规范",
		Level:    driver.RuleLevelError,
	}
	aviodSelectAllColumnHandler := func(ctx context.Context, rule *driver.Rule, sql string) (string, error) {
		if strings.Contains(sql, "select *") {
			return rule.Desc, nil
		}
		return "", nil
	}
	plugin.AddRule(aviodSelectAllColumn, aviodSelectAllColumnHandler)

	////////////////////////////////////////////
	// ... 编写更多规则并通过 AddRule 注册到 SQLE 中
	////////////////////////////////////////////

	// 最后关键一步，调用 `plugin.Serve()` 启动插件：
	plugin.Serve() 
}
```

#### 3. 构建并使用插件

和通常的程序编写流程一样，编写完插件代码后，需要将其构建成二进制文件，然后才能将其注册到 SQLE 中。执行 `go build -o ${二进制名} main.go` 将插件代码构建成二进制文件。最后参考[数据库审核插件使用](./auditplugin_management.md) 来使用你的自定义插件。

#### 4. 其他

##### 4.1 自定义 SQL 解析器
前面介绍的审核规则都是通过字符串匹配的方式来解析 SQL 的内容。这种方式比较适合规则较少，且不要求一些复杂场景下使用。

如果你编写的数据库插件有相应的 SQL 解析器，这时你可以通过调用 `plugin.WithSQLParser()` 来注册自己的 SQL 解析器。在添加规则时则使用 `plugin.AddRuleWithSQLParser()` 来添加带有解析器的处理函数。在处理函数中，将 interface{} 断言成具体的 AST 语法树，通过语法树级别的操作来更加精细的处理 SQL。

##### 4.2 自定义插件
如果 driver 包中默认的 PostgreSQL、Oracle 与 SQL Server 插件不能满足你的需求的话，你也可以自定义一个数据库插件。方法就是实现一个接口：
```Go
type Dialector interface {
	Dialect(dsn *driver.DSN) (driverName string, dsnDetail string)
	ShowDatabaseSQL() string
	String() string
}
```

在实现这个接口前，你需要先了解一下 Go 语言原生 Driver 的概念（以 [MySQL Driver](https://github.com/go-sql-driver/mysql) 为例）。

下面介绍 Dialector 接口的含义：
1. `Dialect`：实现该方法，通过 DSN 提供的 Host Port User Password Database 信息和你选择的数据库 driver，构造出 driverName 与 dsnDetail。driverName 是你引入的数据库 driver 名称；dsnDetail 是连接数据库驱动的必要信息。
2. `ShowDatabaseSQL`：实现该方法，可以自定义你数据源中默认展示的数据库列表，该数据库列表最终会展示在工单审核列表的数据库下拉框中，如下图：
![数据库列表](./pictures/database_list_of_instance.png)
3. `String`：实现该方法，该方法的返回值会作为你实现的数据库审核插件名展示在 SQLE 的相关下拉框中，如下图：
![审核插件列表](./pictures/instance_type_list.png)

将你的实现作为 NewAdaptor 的参数传入即可，后续的步骤与前面规则相关的介绍一致。

## 详细部分

### 一、SQLE与插件的交互图
![sqle call plugin](./pictures/sqle_call_plugin.png)

### 二、开始
下文以 PostgreSQL 审核插件为例，介绍如何从头开发 SQLE 的数据库审核插件，并添加一条规则；示例忽略SQL解析，数据库的查询和执行等细节，需要更详细的示例可以参考我们开源的 PostgreSQL 数据库审核插件：[sqle-pg-plugin](https://github.com/actiontech/sqle-pg-plugin) 。

#### 1. 引入SQLE插件库
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

#### 2. 定义插件的注册信息
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

#### 3. 实现审核驱动
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

#### 4. 注册插件
```go
func main() {
	driver.ServePlugin(&registererImpl{}, NewDriver)
}
```

### 附录
#### 1. 注册接口说明
该接口定义了该插件的名称和实现的规则，SQLE启动的时候会调用该接口获取插件名称和该插件支持的规则列表。
```go
type Registerer interface {
	Name() string
	Rules() []*model.Rule
}
```
1. `Name`: 插件名，最终会展示在 SQLE 页面的数据源类型的下拉框中；
2. `Rules`: 插件支持的规则，在启动 SQLE 时，会调用插件获取这些规则，你将在规则模板内看到它们。

#### 2. 审核接口说明
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

#### 3. 插件的配置信息说明
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

#### 4. 初始化函数说明
插件的主进程入口，由插件的 main 函数调用即可实现插件
```go
func ServePlugin(r Registerer, newDriver func(cfg *Config) Driver)
```
1. `r`: 传入 Registerer 的接口实现, 由插件侧实现； 
2. `newDriver`: 传入 Driver 的初始化函数，该函数的入参是 `Config` 是由 SQLE 向插件传递的配置信息，函数的出参是 Driver 的接口实现，由插件侧实现。

