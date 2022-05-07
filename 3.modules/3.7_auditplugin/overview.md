# 数据库审核插件

## 目录

* [数据库审核插件使用](./auditplugin_management.md)
* [数据库审核插件开发](./auditplugin_development.md)

## 背景

将 SQLE 的数据库审核插件化主要有以下目的：

* 将审核流程（业务）的代码和具体审核实现的代码进行分离，支持多数据源类型的审核
* 审核插件在满足基本规范情况下，与 SQLE 独立开发

## 概述

SQLE 使用 [go-plugin](https://github.com/hashicorp/go-plugin) 来实现审核插件化。插件只需实现两个接口，即可实现与 SQLE
主进程通信（详情见[审核插件开发](./auditplugin_development.md)）。

## 部分插件项目地址

* [PostgreSQL](https://github.com/actiontech/sqle-pg-plugin)
* [Oracle](https://github.com/actiontech/sqle-oracle-plugin)
* [SQL Server](https://github.com/actiontech/sqle-ms-plugin)
* [DB2](https://github.com/actiontech/sqle-db2-plugin)