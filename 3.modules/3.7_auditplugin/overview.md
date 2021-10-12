# 审核插件

## 目录
* [审核插件管理](./auditplugin_management.md)

## 背景
将 SQLE 的审核插件化主要有以下目的：
* 将审核流程（业务）的代码和具体审核实现的代码进行分离，支持多数据源类型审核
* 审核插件在满足基本规范情况下，与 SQLE 独立开发、语言无关（插件可以使用非 Go 语言开发）

## 概述
SQLE 使用 [go-plugin](https://github.com/hashicorp/go-plugin) 来实现审核插件化。插件只需实现定义的 gRPC 接口，即可实现与 SQLE 主进程通信。