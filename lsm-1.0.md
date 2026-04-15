# LS Mapping 通用规范（LSM 1.0）

## 1. 规范概述

本规范定义了标签与SQL条件之间的映射关系，用于快速定义和使用标签-SQL映射。

## 2. 设计原则

- **极简性**：只保留核心字段，无冗余信息
- **通用性**：不绑定业务场景、数据表、开发语言
- **可执行性**：SQL条件可直接在数据库执行
- **AI友好**：结构清晰，易于理解

## 3. 核心格式定义

采用YAML格式存储，语法简洁、人类可读：

```yaml
# 规范版本
version: "1.0"
# 配置名称
name: "配置名称"
# 配置唯一标识
id: "配置唯一标识"
# 数据库类型配置（可选）
database:
  # 数据库类型
  type: "数据库类型"  # 如 mysql, postgres, sqlite 等
  # 表名配置（数组）
  # 第一个表默认为主表，后续表为子表
  tables:
    # 主表
    - name: "表名"  # 数据库表名
      alias: "表别名"  # 可选，SQL语句中的表别名，默认使用表名
    # 子表1
    - name: "表名2"
      alias: "表别名2"  # 可选，默认使用表名
      join: "join类型"  # 可选，如 inner, left, right 等，默认为 left
      on: "join条件"  # 可选，join时的关联条件，默认使用 `主表别名.id = 子表别名.id`
    # 子表2
    - name: "表名3"
      alias: "表别名3"  # 可选，默认使用表名
      join: "join类型2"  # 可选，默认为 left
      on: "join条件2"  # 可选，默认使用 `主表别名.id = 子表别名.id`

# 标签映射集合
mappings:
  # 标签1
  - id: "标签唯一ID"
    # 标签类型名称
    name: "标签类型名称"
    # 映射项
    items:
      # 映射1
      - condition: "SQL条件"
        name: "显示名称"  # 支持 {value} 占位符，用于显示字段值
      # 映射2
      - condition: "SQL条件2"
        name: "显示名称2"
  # 标签2
  - id: "标签唯一ID2"
    name: "标签类型名称2"
    items:
      - condition: "SQL条件3"
        name: "显示名称3"

# 插值说明
# 1. {value} 占位符：用于在显示名称中插入字段的实际值
# 2. 示例：
#    - 对于数值字段：name: "{value}元" → 显示为 "100元"
#    - 对于百分比字段：name: "{value}%" → 显示为 "50%"
#    - 对于等级字段：name: "等级{value}" → 显示为 "等级5"
# 3. 适用场景：数值型字段的格式化显示，如价格、百分比、等级等
```

## 4. 示例

### 4.1 游戏卡牌场景

```yaml
version: "1.0"
name: "游戏王卡片标签配置"
id: "ygopro_card_labels"
database:
  type: "sqlite"
  tables:
    - name: "datas"
      alias: "d"
    - name: "texts"
      alias: "t"
      join: "left"  # 使用left join以主表为准
      # 未指定on条件，默认使用 d.id = t.id

mappings:
  # 卡片类型
  - id: "card_type"
    name: "卡片类型"
    items:
      - condition: "(d.type & 1) AND ((d.type & 32) = 0) AND ((d.type & 64) = 0)"
        name: "通常怪兽"
      - condition: "(d.type & 1) AND (d.type & 32)"
        name: "效果怪兽"
      - condition: "(d.type & 1) AND (d.type & 64)"
        name: "融合怪兽"
      - condition: "(d.type & 1) AND (d.type & 8192)"
        name: "同调怪兽"
      - condition: "(d.type & 1) AND (d.type & 8388608)"
        name: "XYZ怪兽"
      - condition: "(d.type & 1) AND (d.type & 67108864)"
        name: "连接怪兽"
      - condition: "(d.type & 1) AND (d.type & 128)"
        name: "仪式怪兽"
      - condition: "(d.type & 1) AND (d.type & 16777216)"
        name: "灵摆怪兽"
      - condition: "(d.type & 1) AND (d.type & 4096)"
        name: "调整怪兽"
      - condition: "(d.type & 2) AND ((d.type & 128) = 0) AND ((d.type & 65536) = 0) AND ((d.type & 131072) = 0) AND ((d.type & 262144) = 0) AND ((d.type & 524288) = 0)"
        name: "通常魔法"
      - condition: "(d.type & 2) AND (d.type & 131072)"
        name: "永续魔法"
      - condition: "(d.type & 2) AND (d.type & 262144)"
        name: "装备魔法"
      - condition: "(d.type & 2) AND (d.type & 65536)"
        name: "速攻魔法"
      - condition: "(d.type & 2) AND (d.type & 524288)"
        name: "场地魔法"
      - condition: "(d.type & 2) AND (d.type & 128)"
        name: "仪式魔法"
      - condition: "(d.type & 4) AND ((d.type & 131072) = 0) AND ((d.type & 1048576) = 0)"
        name: "通常陷阱"
      - condition: "(d.type & 4) AND (d.type & 131072)"
        name: "永续陷阱"
      - condition: "(d.type & 4) AND (d.type & 1048576)"
        name: "反击陷阱"
```

### 4.2 电商场景

```yaml
version: "1.0"
name: "商品标签配置"
id: "product_labels"
database:
  type: "mysql"
  tables:
    - name: "products"
      alias: "p"
    - name: "categories"
      alias: "c"
      join: "left"
      on: "p.category_id = c.id"

mappings:
  # 价格区间
  - id: "price_range"
    name: "价格区间"
    items:
      - condition: "p.price >= 0 AND p.price < 100"
        name: "0-100元"
      - condition: "p.price >= 100 AND p.price < 500"
        name: "100-500元"
      - condition: "p.price >= 500"
        name: "500元以上"
  # 商品状态
  - id: "status"
    name: "商品状态"
    items:
      - condition: "p.status = 0"
        name: "下架"
      - condition: "p.status = 1"
        name: "上架"
      - condition: "p.status = 2"
        name: "缺货"
```

## 5. 使用方式

1. **标签展示**：通过labels展示标签类型
2. **值映射**：通过mappings中的condition和name将数据库值映射为可读名称
3. **数据库适配**：根据database配置适配不同类型的数据库
4. **自动表连接**：系统根据tables配置自动生成join语句
5. **AI对接**：将配置嵌入AI提示词，作为理解业务的标准规则
6. **AI调用SDK工具**：
   - **AI检索**：使用LSM配置作为知识库，辅助AI理解业务逻辑
   - **标签对应SQL**：将标签转换为对应的SQL条件
   - **SQL数据查询**：执行生成的SQL查询获取数据
   - **主标签查询**：查询单项数据的主标签信息

## 6. 优势

- **极简设计**：快速上手，易于维护
- **性能最优**：所有逻辑下推数据库执行
- **无依赖**：可接入任意系统
- **跨平台通用**：支持多场景使用
- **值映射**：支持通过SQL条件将数据库值映射为可读名称
- **数据库适配**：支持不同类型数据库的配置
- **自动表连接**：根据配置自动生成表连接语句
- **AI友好**：结构清晰，易于AI理解和使用
- **标签对应SQL**：支持标签到SQL条件的转换
- **主标签查询**：支持单项数据的主标签查询

## 7. 文件目录配置结构

### 7.1 目录结构

```
lsm-ygopro-database/                # 游戏王卡片配置目录（LSM规范实现）
└── main.yaml                      # 主标签文件（包含数据库描述和完整映射）
```

### 7.2 主标签文件 (lsm-ygopro-database/main.yaml)

主标签文件包含数据库描述和完整的映射配置：

```yaml
version: "1.0"
name: "游戏王卡片标签配置"
id: "ygopro_card_labels"
database:
  type: "sqlite"
  tables:
    - name: "datas"
      alias: "d"
    - name: "texts"
      alias: "t"
      join: "left"
      # 未指定on条件，默认使用 d.id = t.id

mappings:
  # 卡片类型
  - id: "card_type"
    name: "卡片类型"
    items:
      - condition: "(d.type & 1) AND ((d.type & 32) = 0) AND ((d.type & 64) = 0)"
        name: "通常怪兽"
      - condition: "(d.type & 1) AND (d.type & 32)"
        name: "效果怪兽"
      # 其他映射...
  # 属性
  - id: "attribute"
    name: "属性"
    items:
      - condition: "attribute = 0"
        name: "无"
      - condition: "attribute = 1"
        name: "地"
      # 其他映射...
  # 种族
  - id: "race"
    name: "种族"
    items:
      - condition: "race = 0"
        name: "无"
      - condition: "race = 1"
        name: "战士"
      # 其他映射...
  # OCG/TCG
  - id: "ocg_tcg"
    name: "OCG/TCG"
    items:
      - condition: "ocg_tcg = 1"
        name: "OCG"
      - condition: "ocg_tcg = 2"
        name: "TCG"
      # 其他映射...
  # 等级/阶级/连接数
  - id: "level"
    name: "等级"
    items:
      - condition: "(d.type & 8388608)"
        name: "{value}阶"
      - condition: "(d.type & 67108864)"
        name: "LINK-{value}"
      - condition: "(d.type & 1)"
        name: "★{value}"
  # 攻击力
  - id: "atk"
    name: "攻击力"
    items:
      - condition: "(d.type & 1)"
        name: "{value}ATK"
  # 防御力
  - id: "def"
    name: "防御力"
    items:
      - condition: "(d.type & 1) AND ((d.type & 67108864) = 0)"
        name: "{value}DEF"
```