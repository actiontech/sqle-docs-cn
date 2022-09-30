# SQL工作台

## 目录

* [配置SQL工作台](./sql_query_config.md)
* [SQL工作台使用](./use_sql_query.md)

## 背景

SQL查询是DBA日常工作中的一部分，但有些场景可能给DBA带来困扰，例如：业务人员需要对数据库进行数据检索，DBA很难对业务人员的行为进行追踪和管控，无法限制有问题的SQL在生产环境执行，一些查询SQL可能会造成数据库性能问题甚至hang死。SQL质量管理平台如果提供SQL查询能力，
就可以对业务人员的SQL进行审计，拒绝不合规的SQL运行，也可以做较为细化的权限控制。

## 概述

SQLE为用户提供SQL工作台功能，通过集成CloudBeaver的方式实现在SQLE中方便的操作数据库. SQL工作台会自动同步用户信息和实例信息, 有效减少数据库密码泄露的风险,
同时可通过参数配置及SQL查询审核对SQL做限制和筛查，通过权限设置管理可操作人员，有效避免不合规的查询和避免无权限的人进行操作。

## CloudBeaver 简介

[CloudBeaver Community](https://github.com/dbeaver/cloudbeaver) 是一个开源的 Web 数据库可视化管理工具，前端基于 TypeScript 和 React 编写，支持
PostgreSQL, MySQL, MariaDB, SQL Server, Oracle, DB2, Firebird, H2, Trino 等数据库。

## 注意事项

### 1. CloudBeaver支持版本

当前CloudBeaver仅支持22.2.0版本

### 2. 需要禁用CloudBeaver原地址的使用

当CloudBeaver接入SQLE后, 应当使用SQLE的地址访问CloudBeaver, 而不应该继续使用CloudBeaver之前的地址访问, SQLE SQL工作台地址: http://{SQLE IP}:
{SQLE端口}/sql_query#/