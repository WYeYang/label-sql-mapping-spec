# Label-SQL Mapping (LSM) 1.0 规范

## 简介

Label-SQL Mapping (LSM) 是一种用于定义标签与 SQL 查询条件之间映射关系的规范。它允许用户通过标签来查询数据库，而不需要直接编写 SQL 语句。

## 规范文件

- [lsm-1.0.md](lsm-1.0.md) - LSM 1.0 完整规范文档

## 主要特性

- **标签映射**：将标签 ID 与 SQL 查询条件关联
- **值映射**：将原始值与显示名称关联
- **数据库配置**：支持多种数据库类型和表结构
- **表关联**：支持多表关联查询
- **轻量级设计**：简洁的 YAML 配置格式

## 应用场景

- 游戏卡片查询系统
- 商品分类查询
- 内容管理系统
- 任何需要通过标签进行数据查询的场景

## 如何使用

1. 按照 LSM 规范创建配置文件
2. 使用 LSM SDK 加载配置文件
3. 通过标签 ID 和值进行查询

## 相关项目

- [label-sql-mapping-sdk](https://github.com/user/label-sql-mapping-sdk) - LSM SDK 和 CLI 工具
- [lsm-ygopro-database](https://github.com/user/lsm-ygopro-database) - 游戏王数据库配置
- [ygopro-card-cli](https://github.com/user/ygopro-card-cli) - 游戏王卡片查询 CLI 工具