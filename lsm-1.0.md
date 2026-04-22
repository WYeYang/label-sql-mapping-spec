# LS Mapping 通用规范（LSM 2.0）

## 1. 规范概述

本规范定义了标签与SQL条件之间的映射关系，用于快速定义和使用标签-SQL映射。

## 2. 设计原则

- **极简性**：只保留核心字段，无冗余信息
- **通用性**：不绑定业务场景、数据表、开发语言
- **可执行性**：SQL条件可直接在数据库执行
- **AI友好**：结构清晰，易于理解
- **可组合**：支持前置条件复用，减少重复

## 3. 核心格式定义

采用YAML格式存储，语法简洁、人类可读。字符串值在无特殊字符时可省略引号。

```yaml
version: "1.0"
name: 配置名称
id: 配置唯一标识

database:
  type: sqlite
  tables:
    - name: 表名
      alias: 别名

mappings:
  # 单值模式
  - id: field_id
    name: 字段名称
    value: 表别名.字段名

  # 单值模式 + 条件
  - id: field_with_condition
    name: 字段名称
    condition: 表别名.类型字段 = 1
    value: 表别名.字段名

  # 多值模式（items）
  - id: enum_field
    name: 枚举字段
    items:
      - condition: 表别名.字段 = 1
        value: 状态一
      - condition: 表别名.字段 = 2
        value: 状态二

  # 多值模式 + 前置条件 + 默认值
  - id: status_with_default
    name: 状态
    condition: 表别名.类型 = 1
    value: 默认状态
    items:
      - condition: 表别名.状态 = 1
        value: 状态一
      - condition: 表别名.状态 = 2
        value: 状态二
```

## 4. 字段说明

### 4.1 顶层字段

| 字段 | 必填 | 说明 |
|------|------|------|
| `version` | 是 | 规范版本 |
| `name` | 是 | 配置名称 |
| `id` | 是 | 配置唯一标识 |
| `database` | 是 | 数据库配置 |
| `mappings` | 是 | 标签映射集合 |

### 4.1.1 数据库路径配置

数据库路径在 `lsm-sdk-js.yaml` 中配置：

```yaml
# lsm-sdk-js.yaml
llm:
  apiKey: ...
  apiUrl: ...
databasePath: ./lsm-ygopro-database/cards.cdb  # 数据库文件路径
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `databasePath` | 是 | 数据库文件路径，支持相对路径或绝对路径 |

**注意**：`labels.yaml` 中的 `database.path` 字段已废弃，不再使用。

### 4.2 LabelMapping（标签映射）

| 字段 | 必填 | 说明 |
|------|------|------|
| `id` | 是 | 标签唯一ID |
| `name` | 是 | 标签类型名称 |
| `description` | 否 | 标签描述，用于 AI 理解标签含义 |
| `condition` | 条件性 | 单值模式时必须；items 模式时可选，作为所有 items 的前置条件 |
| `value` | 条件性 | 单值模式时必须（字段引用）；items 模式时可选（作为默认值） |
| `items` | 条件性 | items 模式时必须（映射项数组）；单值模式时不存在 |
| `range` | 否 | 数值范围约束（有 range 时表示为数值类型），如 `{ min: 0, max: 5000 }` |

### 4.3 MappingItem（映射项）

| 字段 | 必填 | 说明 |
|------|------|------|
| `condition` | 是 | 匹配条件 |
| `value` | 是 | 字段引用或展示值 |

## 5. 模式详解

### 5.1 单值模式

最简洁的形式，直接输出字段值。适用于无需条件判断的场景。

```yaml
- id: title            # 标签ID，程序中引用
  name: 标题            # 标签名称，用于展示
  condition: o.status != 0  # 前置条件（单值模式时必须）
  value: p.title        # 字段引用，直接输出 p.title 字段的值
```

→ SQL：
```sql
CASE WHEN o.status != 0 THEN p.title END AS title
```

→ 返回数据示例：
| order_id | title |
|----------|-------|
| 1001 | iPhone 15 Pro |
| 1002 | MacBook Air |
| 1003 | AirPods Pro |

### 5.2 单值模式 + 条件

满足条件时输出字段值，不满足时输出 NULL。适用于条件性输出。

```yaml
- id: price             # 标签ID
  name: 价格             # 标签名称
  condition: p.type = 1  # 前置条件：只有 type = 1 时才输出价格
  value: p.price        # 字段引用
```

→ SQL：
```sql
CASE WHEN p.type = 1 THEN p.price END AS price
```

→ 返回数据示例：
| product_id | type | price | 备注 |
|------------|------|-------|------|
| 1 | 1 | 8999.00 | type=1，输出价格 |
| 2 | 1 | 12999.00 | type=1，输出价格 |
| 3 | 2 | NULL | type=2，不满足条件 |

### 5.3 多值模式（items）

多个条件映射到不同值。按顺序匹配，第一个满足的条件生效。

```yaml
- id: category         # 标签ID
  name: 类别            # 标签名称
  condition: o.type > 0  # 前置条件（必须）
  items:                # 映射项数组
    - condition: c.type = 1  # 条件1
      value: 食品           # 满足条件1时输出
    - condition: c.type = 2  # 条件2
      value: 服装           # 满足条件2时输出
    - condition: c.type = 3  # 条件3
      value: 电子           # 满足条件3时输出
```

→ SQL：
```sql
CASE 
  WHEN o.type > 0 AND c.type = 1 THEN '食品'
  WHEN o.type > 0 AND c.type = 2 THEN '服装'
  WHEN o.type > 0 AND c.type = 3 THEN '电子'
END AS category
```

→ 返回数据示例：
| product_id | type | category |
|------------|------|----------|
| 1 | 1 | 食品 |
| 2 | 2 | 服装 |
| 3 | 3 | 电子 |
| 4 | 99 | NULL |

**说明**：
- 前 3 个商品匹配对应的 items，返回映射值
- 商品 4：type=99 不在 items 中，无匹配返回 NULL

### 5.4 多值模式 + 前置条件 + 默认值

前置条件被所有 items 共享，最后一个 WHEN 使用 value 作为默认值。

**使用场景**：
1. 多个 items 有共同的前置条件
2. items 都不满足时，使用 value 作为默认值

```yaml
- id: order_status     # 标签ID
  name: 订单状态        # 标签名称
  condition: o.type = 1  # 前置条件：仅 type=1 的订单有状态流
  value: 待处理          # 默认值：前置条件满足但所有 items 都不匹配时使用
  items:                # 映射项数组
    - condition: o.status = 1  # 条件1
      value: 已支付           # 满足条件1时输出
    - condition: o.status = 2  # 条件2
      value: 已发货           # 满足条件2时输出
    - condition: o.status = 3  # 条件3
      value: 已收货           # 满足条件3时输出
```

→ SQL：
```sql
CASE 
  WHEN o.type = 1 AND o.status = 1 THEN '已支付'
  WHEN o.type = 1 AND o.status = 2 THEN '已发货'
  WHEN o.type = 1 AND o.status = 3 THEN '已收货'
  WHEN o.type = 1 THEN '待处理'   -- 默认值作为最后一条 WHEN，仍需满足前置条件
END AS order_status
```

→ 返回数据示例：
| order_id | type | status | order_status |
|----------|------|--------|--------------|
| 1001 | 1 | 1 | 已支付 |
| 1002 | 1 | 2 | 已发货 |
| 1003 | 1 | 99 | 待处理 |
| 1004 | 2 | 1 | NULL |
| 1005 | 1 | -1 | 已取消 |

**说明**：
- 订单 1001/1002：匹配 items，返回对应状态
- 订单 1003：type=1 但 status=99 不在 items 中，返回默认值"待处理"
- 订单 1004：type=2 不满足前置条件，返回 NULL
- 订单 1005：type=1 且 status=-1 匹配 items，返回"已取消"

**注意**：默认值作为最后一条 WHEN 输出，所以仍需满足前置条件 `o.type = 1`。

## 6. value 类型推断

代码自动根据 value 的格式推断其类型：

| value 格式 | 类型 | SQL输出 |
|-----------|------|---------|
| `p.name` | 字段引用（包含`.`） | `p.name` |
| 其他 | 字符串 | `'value'` |

## 7. YAML 语法简化

### 7.1 引号规则

- 简单字符串（无特殊字符）可省略引号
- 包含特殊字符（如 `|`、`:`、`#`）需要引号

```yaml
# 可省略引号
name: 订单系统配置
value: 待处理
condition: o.status = 1

# 需要引号（包含 |）
value: "状态一|状态二"
```

### 7.2 布尔值

- `true` / `false` 可直接使用

```yaml
- condition: true   # 始终匹配
```

## 8. 电商订单配置示例

```yaml
version: "1.0"                        # 规范版本号，用于版本兼容性判断
name: 电商订单标签配置                  # 配置的中文名称，用于展示
id: order_labels                      # 配置的唯一标识，用于代码引用

# ==================== 数据库配置 ====================
database:
  type: mysql                         # 数据库类型，支持 mysql/postgres/sqlite/mssql
  path: ./database.db                 # 数据库文件路径（sqlite/文件数据库需要）
  tables:
    - name: orders                    # 主表：订单表
      alias: o                        # 别名，SQL中用 o 代替 orders

    - name: products                  # 子表1：商品表
      alias: p                        # 别名
      join: left                      # JOIN类型：left/inner/right，默认为 left
      on: o.product_id = p.id        # JOIN条件：订单关联商品

    - name: users                     # 子表2：用户表
      alias: u
      join: left
      on: o.user_id = u.id           # JOIN条件：订单关联用户

# ==================== 标签映射配置 ====================
mappings:
  # -------- 单值模式 --------
  # 适用场景：直接输出字段值，无需条件判断

  - id: order_id                      # 标签ID，程序中引用
    name: 订单ID                       # 标签名称，用于展示
    value: o.id                       # 字段引用，直接输出订单ID

  # -------- 单值模式 + 条件 --------
  # 适用场景：满足条件时才输出值，否则为 NULL

  - id: amount                        # 标签ID
    name: 订单金额                      # 标签名称
    condition: o.status != 0          # 前置条件：只有非取消状态的订单才显示金额
    value: o.amount                   # 字段引用

  - id: product_name                  # 标签ID
    name: 商品名称                      # 标签名称
    condition: p.name IS NOT NULL     # 条件：商品名称不为空
    value: p.name                     # 输出商品名字段

  - id: user_name                     # 标签ID
    name: 用户名称                      # 标签名称
    condition: u.name IS NOT NULL     # 条件：用户名称不为空
    value: u.name                     # 输出用户名字段

  - id: create_time                   # 标签ID
    name: 创建时间                      # 标签名称
    condition: o.create_time IS NOT NULL  # 条件：创建时间不为空
    value: o.create_time              # 输出创建时间字段

  # -------- 多值模式（items）--------
  # 适用场景：单一字段的枚举值映射为中文显示

  - id: order_type                    # 标签ID
    name: 订单类型                      # 标签名称
    items:                            # 映射项数组
      - condition: o.type = 1         # 条件：订单类型字段 = 1
        value: 普通订单                 # 映射结果：显示"普通订单"
      - condition: o.type = 2         # 条件：订单类型字段 = 2
        value: 预售订单                 # 映射结果：显示"预售订单"
      - condition: o.type = 3         # 条件：订单类型字段 = 3
        value: 团购订单                 # 映射结果：显示"团购订单"

  # -------- 多值模式 + 前置条件 + 默认值 --------
  # 适用场景：
  #   1. 多个 items 有共同的前置条件
  #   2. items 都不满足时，使用 value 作为默认值
  # 注意：value 会被放在最后一条 WHEN，仍需满足前置条件

  - id: order_status                  # 标签ID
    name: 订单状态                      # 标签名称
    condition: o.type = 1             # 前置条件：仅普通订单有完整状态流
    value: 待处理                       # 默认值：前置条件满足但所有 items 都不匹配时显示
    items:                            # 映射项数组
      - condition: o.status = 1        # 条件：状态 = 1
        value: 已支付                   # 结果：已支付
      - condition: o.status = 2        # 条件：状态 = 2
        value: 已发货                   # 结果：已发货
      - condition: o.status = 3        # 条件：状态 = 3
        value: 已收货                   # 结果：已收货
      - condition: o.status = 4        # 条件：状态 = 4
        value: 已完成                   # 结果：已完成
      - condition: o.status = -1       # 条件：状态 = -1
        value: 已取消                   # 结果：已取消

  - id: payment_method                 # 标签ID
    name: 支付方式                      # 标签名称
    condition: o.pay_type > 0         # 前置条件：已设置支付方式（>0）
    value: 其他                         # 默认值：支付方式未知时显示"其他"
    items:
      - condition: o.pay_type = 1     # 条件：支付方式 = 1
        value: 微信支付                 # 结果：微信支付
      - condition: o.pay_type = 2     # 条件：支付方式 = 2
        value: 支付宝                   # 结果：支付宝
      - condition: o.pay_type = 3     # 条件：支付方式 = 3
        value: 银行卡                   # 结果：银行卡

  - id: category                      # 标签ID
    name: 商品分类                      # 标签名称
    items:
      - condition: p.category_id = 1  # 条件：分类ID = 1
        value: 食品饮料                 # 结果：食品饮料
      - condition: p.category_id = 2  # 条件：分类ID = 2
        value: 服装鞋包                 # 结果：服装鞋包
      - condition: p.category_id = 3  # 条件：分类ID = 3
        value: 电子产品                 # 结果：电子产品
      - condition: p.category_id = 4  # 条件：分类ID = 4
        value: 家居生活                 # 结果：家居生活

  - id: user_level                    # 标签ID
    name: 用户等级                      # 标签名称
    items:
      - condition: u.level = 1        # 条件：用户等级 = 1
        value: 普通会员                 # 结果：普通会员
      - condition: u.level = 2        # 条件：用户等级 = 2
        value: 银牌会员                 # 结果：银牌会员
      - condition: u.level = 3        # 条件：用户等级 = 3
        value: 金牌会员                 # 结果：金牌会员
      - condition: u.level = 4        # 条件：用户等级 = 4
        value: 钻石会员                 # 结果：钻石会员
```

## 9. SQL 输出示例

```sql
SELECT 
  o.id,
  CASE WHEN o.status != 0 THEN o.amount END AS amount,
  CASE WHEN p.name IS NOT NULL THEN p.name END AS product_name,
  CASE WHEN u.name IS NOT NULL THEN u.name END AS user_name,
  CASE 
    WHEN o.type = 1 THEN '普通订单'
    WHEN o.type = 2 THEN '预售订单'
    WHEN o.type = 3 THEN '团购订单'
  END AS order_type,
  CASE 
    WHEN o.type = 1 AND o.status = 1 THEN '已支付'
    WHEN o.type = 1 AND o.status = 2 THEN '已发货'
    WHEN o.type = 1 AND o.status = 3 THEN '已收货'
    WHEN o.type = 1 AND o.status = 4 THEN '已完成'
    WHEN o.type = 1 AND o.status = -1 THEN '已取消'
    WHEN o.type = 1 THEN '待处理'
  END AS order_status,
  CASE 
    WHEN o.pay_type = 1 THEN '微信支付'
    WHEN o.pay_type = 2 THEN '支付宝'
    WHEN o.pay_type = 3 THEN '银行卡'
    WHEN o.pay_type > 0 THEN '其他'
  END AS payment_method,
  CASE 
    WHEN p.category_id = 1 THEN '食品饮料'
    WHEN p.category_id = 2 THEN '服装鞋包'
    WHEN p.category_id = 3 THEN '电子产品'
    WHEN p.category_id = 4 THEN '家居生活'
  END AS category,
  CASE 
    WHEN u.level = 1 THEN '普通会员'
    WHEN u.level = 2 THEN '银牌会员'
    WHEN u.level = 3 THEN '金牌会员'
    WHEN u.level = 4 THEN '钻石会员'
  END AS user_level,
  CASE WHEN o.create_time IS NOT NULL THEN o.create_time END AS create_time
FROM orders AS o
  LEFT JOIN products AS p ON o.product_id = p.id
  LEFT JOIN users AS u ON o.user_id = u.id
WHERE ...
```

### 返回数据示例

| order_id | amount | product_name | user_name | order_type | order_status | payment_method | category | user_level | create_time |
|----------|--------|--------------|-----------|------------|--------------|----------------|----------|------------|-------------|
| 10001 | 8999.00 | iPhone 15 | 张三 | 普通订单 | 已支付 | 微信支付 | 电子产品 | 金牌会员 | 2024-01-15 |
| 10002 | 299.00 | AirPods | 李四 | 普通订单 | 待处理 | 其他 | 电子产品 | 银牌会员 | 2024-01-16 |
| 10003 | 0 | NULL | 王五 | 预售订单 | NULL | NULL | 服装鞋包 | 普通会员 | NULL |
| 10004 | 599.00 | 运动鞋 | 赵六 | 普通订单 | 已发货 | 支付宝 | 服装鞋包 | 钻石会员 | 2024-01-14 |
| 10005 | 1299.00 | 机械键盘 | NULL | 普通订单 | 已取消 | 银行卡 | NULL | NULL | 2024-01-13 |

**数据说明**：
- 订单 10001：完整数据，所有标签都有值
- 订单 10002：payment_method = 5 不在 items 中，使用默认值"其他"
- 订单 10003：status=0 不满足条件（!=0），amount 为 NULL；create_time 为 NULL
- 订单 10004：数据完整，预售订单没有 order_status（前置条件 type=1 不满足）
- 订单 10005：user_name/product_name/category/user_level 为 NULL（LEFT JOIN 不匹配或 IS NOT NULL 条件不满足）

## 10. 通用性分析

### 10.1 通用能力

| 能力 | 支持 | 说明 |
|------|------|------|
| 多表关联 | ✅ | 通过 tables 配置自动生成 JOIN |
| 枚举映射 | ✅ | items 模式支持任意枚举值映射 |
| 条件过滤 | ✅ | condition 支持任意 SQL 条件 |
| 字段直出 | ✅ | value 直接输出字段或表达式 |
| 前置条件复用 | ✅ | condition 减少重复条件 |
| 默认值 | ✅ | value 作为 items 模式的兜底 |
| 位掩码 | ✅ | condition 支持位运算（`&`） |
| 字符串拼接 | ✅ | value 支持 SQL 表达式 |

### 10.2 通用场景适配

| 场景 | 适配方式 |
|------|---------|
| 电商订单 | 枚举映射 + 多表关联 |
| 内容管理 | 分类标签 + 状态枚举 |
| 物联网设备 | 设备类型 + 状态码 |
| 金融交易 | 交易类型 + 状态流 |
| 日志分析 | 日志级别 + 来源分类 |

### 10.3 数据库适配

规范本身不绑定特定数据库，关键点：

| 数据库 | 位运算语法 | 字符串拼接 |
|--------|-----------|-----------|
| MySQL | `d.type & 64` | `CONCAT(a, b)` |
| PostgreSQL | `d.type & 64` | `a \|\| b` |
| SQLite | `d.type & 64` | `a \|\| b` |
| SQL Server | `d.type & 64` | `CONCAT(a, b)` |

**建议**：将数据库特定语法封装在配置层，SDK 只负责结构解析。

### 10.4 扩展性思考

| 潜在扩展 | 方案 |
|---------|------|
| 聚合函数 | 新增 `aggregate` 字段：`SUM(d.amount)` |
| 分组统计 | 在 SDK 层处理，不在配置层 |
| 自定义函数 | value 支持函数调用 |
| 国际化 | 新增 `i18n` 字段或外部引用 |

## 11. 使用方式

1. **标签展示**：通过 mappings 展示标签类型
2. **值映射**：通过 items 中的 condition 和 value 将数据库值映射为可读名称
3. **前置条件**：通过 condition 字段提取公共条件，减少重复
4. **默认值**：通过 value 字段作为 items 模式的兜底值
5. **数据库适配**：根据 database 配置适配不同类型的数据库
6. **自动表连接**：系统根据 tables 配置自动生成 join 语句
7. **AI对接**：将配置嵌入AI提示词，作为理解业务的标准规则

## 12. 优势

- **极简设计**：快速上手，易于维护
- **性能最优**：所有逻辑下推数据库执行
- **无依赖**：可接入任意系统
- **跨平台通用**：支持多场景使用
- **前置条件复用**：减少重复条件编写
- **默认值支持**：简化兜底逻辑
- **AI友好**：结构清晰，易于AI理解和使用
- **YAML简洁**：字符串可省略引号

## 13. 扩展标签配置

扩展标签用于定义额外的查询标签，通过 `extensions/` 目录下的 YAML 文件配置。

### 13.1 文件结构

```
extensions/
├── category.yaml      # 分类标签
├── tag.yaml           # 标签
└── ...
```

### 13.2 扩展标签格式

```yaml
# extensions/category.yaml
id: category                    # 标签ID
name: 分类                       # 标签名称
description: 商品分类，用于区分不同类型的商品  # 描述，用于AI理解
items:
  - condition: p.type & 0x1 = 1
    value: 食品
  - condition: p.type & 0x2 = 2
    value: 服装
  # ...
```

### 13.3 简化格式（传给 AI）

扩展标签传给 AI 时，采用简化格式以节省 token：

```
id: category
name: 分类
description: 商品分类，用于区分不同类型的商品
values: [食品, 服装, 电子产品, 家居用品, ...]

id: tag
name: 标签
description: 商品标签
values: [热门, 新品, 促销, 限时, ...]
```

**说明**：
- 只传递 `id`、`name`、`description`、`values`
- 不传递 `condition`（AI 不需要理解底层 SQL 逻辑）
- `values` 来自配置中的 `value` 字段

### 13.4 AI 返回格式

AI 返回扩展标签时使用以下格式：

```json
{
  "sql": "SELECT ...",
  "explanation": "...",
  "extensions": [
    {"id": "category", "values": ["食品", "电子产品"]},
    {"id": "tag", "values": ["热门"]}
  ]
}
```

**说明**：
- `extensions` 是额外标签查询，不是 SQL 的组成部分
- `id` 对应扩展标签配置的 ID
- `values` 用于在扩展配置中查找对应的 `condition` 生成额外查询
