# 权限管理

## 目录
* [角色管理](./role_management.md)
* [用户管理](./user_management.md)

## 背景
SQLE 的权限设计有两个初衷：
* 除 admin 用户外，所有用户只能看到自己应该看到的资源
* 除 admin 用户外，所有用户只能操作自己应该操作的资源

SQLE 中的资源包括：
* 数据源
* 审核工单
* 审核计划

基于以上的描述，SQLE 实现了基于 [RBAC](https://en.wikipedia.org/wiki/Role-based_access_control) 的权限管理系统。

![SQLE RBAC](./pictures/sqle_rbac.png)