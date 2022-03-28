# 流程模板管理（企业版功能）

[工作流](https://en.wikipedia.org/wiki/Workflow)要解决的主要问题是：为实现某个业务目标，在多个参与者之间按某种预定规则自动传递文档、信息或者任务。

SQLE 使用工作流来解决 SQL 上线的流程化问题。在 SQLE 中称为流程模板。

## 创建流程模板

在左侧导航栏的「平台管理」中的「流程管理」页面中，点击「创建流程模板」，填写流程模板相关信息。

### 步骤 1：

![create workflow template step1](./pictures/create_workflow_template_step1.png)

在这一步中, 指定了流程的基本信息

* 新建的流程名称为 `workflow_template_a`, 这个名称在流程模板中是唯一的
* 审核流程模板的描述
* 应用这个流程的工单必须所有SQL都没有触发大于 `告警` 级别的规则才可以提交工单
* `asdf` `db2` 这两个数据源将会使用此审核流程

### 步骤 2：

![create workflow template step2](./pictures/create_workflow_template_step2.png)

在这一步中，添加了两个审核步骤：

* 工单创建后，流转到 DBA 初审
* DBA 初审后，流转到部门领导复审

最后，部门领导复审后，流转到 DBA 执行上线。