# SQL工作台

## 目录

* [配置SQL工作台](./sql_query_config.md)
* [SQL工作台使用](./use_sql_query.md)

## 背景

SQL查询是DBA日常工作中的一部分，但有些场景可能给DBA带来困扰，例如：业务人员需要对数据库进行数据检索，DBA很难对业务人员的行为进行追踪和管控，无法限制有问题的SQL在生产环境执行，一些查询SQL可能会造成数据库性能问题甚至hang死。SQL质量管理平台如果提供SQL查询能力，
就可以对业务人员的SQL进行审计，拒绝不合规的SQL运行，也可以做较为细化的权限控制。

## 概述

SQLE为用户提供SQL查询功能，可通过参数配置及SQL查询审核对SQL做限制和筛查，通过权限设置管理可操作人员，有效避免不合规的查询。同时支持的历史记录功能可以追溯过往的查询

## 注意事项

### 1. CloudBeaver支持版本

当前CloudBeaver仅支持22.2.0版本

### 2. 需要禁用CloudBeaver原地址的使用

当CloudBeaver接入SQLE后, 应当使用SQLE的地址访问CloudBeaver, 而不应该继续使用CloudBeaver之前的地址访问, SQLE SQL工作台地址: http://{SQLE IP}:
{SQLE端口}/sql_query#/