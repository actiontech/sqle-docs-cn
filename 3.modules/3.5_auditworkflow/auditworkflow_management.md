# 审核工单管理
[toc]

在这一节中，会用一个具体的案例来逐一介绍「审核工单管理」中的各个功能模块。

## 创建审核工单

在左侧导航栏的「工单」中的「工单列表」页面中，点击「创建工单」，审核工单相关信息，如下图：

![create auditworkflow fill form](./pictures/create_auditworkflow_fill_form.png)

* 工单名称：该字段需要保证工单全局唯一性。
* 工单描述：略
* 数据源：表示修改最终会应用到哪个数据源。
* 数据库：表示 orders 表所在的 schema，相当于执行 use 语句。
* SQL 语句：填写需要上线的 SQL 语句，该语句会被审核。

在确认需要上线的 SQL 语句后，点击页面下方的「审核」按钮：

![create auditworkflow press audit](./pictures/create_auditworkflow_press_audit.png)

在**审核结果**列表中，可以看到被审核的 SQL 语句（**alter table orders add column create_date TIMESTAMP**）不符合该数据源上绑定的审核规则（或者说这条 SQL 触发了该数据源上的审核规则）。

按照**审核结果**给出的提示修改 SQL 语句，再次点击「审核」按钮：

![create auditworkflow after change](./pictures/create_auditworkflow_after_change.png)

可以看到，修改后的 SQL 已经完全符合规范，审核通过率也从 0% 变成了 100%。点击「创建工单」，回到工单列表，即可看到该工单显示为**处理中**：

![auditworkflow list](./pictures/auditworkflow_list.png)

## 删除审核工单