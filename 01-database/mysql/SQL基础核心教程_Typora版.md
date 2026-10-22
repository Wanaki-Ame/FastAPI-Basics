# MySQL 8 基础核心教程：从读懂 SQL 到独立写查询

> **适用对象**：已学过 SQL 基础、但看到分组、连接、子查询时仍需要解释的学习者。  
> **学习目标**：掌握常用的 **查、增、改、删、JOIN、GROUP BY、基础子查询和事务**，并能作为以后开发 FastAPI / Agent 应用时的基础复习资料。  
> **运行环境**：MySQL 8.x、SQLyog（其他 MySQL 客户端也适用）。  
> **阅读方式**：按章节运行 SQL → 观察结果 → 修改条件再运行 → 关闭示例自己重写。

---

## 目录

[TOC]

> Typora 提示：在「偏好设置 → Markdown → Markdown 扩展语法」中按需开启目录；`[TOC]` 在支持此语法的 Typora 版本中会生成可点击的目录。本文使用标准 Markdown 标题、表格和 `sql` 代码块，避免复杂 HTML 与特殊插件。

## 第 0 章：学习路线和最重要的 SQL 思维

### 0.1 先学什么，后学什么

| 阶段 | 内容 | 学会后的标志 |
|---|---|---|
| 1 | `SELECT`、`WHERE`、排序 | 能从单表筛选需要的数据 |
| 2 | 聚合函数、`GROUP BY`、`HAVING` | 能写分类统计、筛选分组的 SQL |
| 3 | `INSERT`、`UPDATE`、`DELETE` | 能安全地维护数据 |
| 4 | `INNER JOIN`、`LEFT JOIN` | 能把两张表的信息组合起来 |
| 5 | 标量、集合、关联子查询 | 能使用其他查询的结果作为条件 |
| 6 | 事务、`COMMIT`、`ROLLBACK` | 能把多步业务操作作为一个整体 |

**原则：不是背住每段 SQL，而是先说清楚问题，再确定语句结构。**

例如题目：「查询每类商品中价格高于该类平均价的商品。」

1. 最终显示什么？`product_type`、`product_name`、`sale_price`。
2. 从哪张表取？`product`。
3. 过滤依据是什么？每条商品售价 > **它所属类别的平均售价**。
4. 「所属类别的平均售价」依赖当前外层商品，因此适合关联子查询，或者先分组再 JOIN。

### 0.2 表、行、列、主键

把关系表想成一张 Excel 表：

- **表（Table）**：一类数据，例如商品表 `product`。
- **行（Row）**：一个商品，例如商品编号 `0001`。
- **列（Column）**：一项属性，例如 `sale_price`。
- **主键（PRIMARY KEY）**：唯一标识一条记录，例如 `product_id`。
- **外键（FOREIGN KEY）**：引用另一张表的主键，用来维护表之间的关系。

### 0.3 SQL 的书写顺序与理解顺序

常用书写模板：

```sql
SELECT 要显示的列或计算结果
FROM 数据来源
JOIN 另一张表 ON 连接条件
WHERE 行筛选条件
GROUP BY 分组列
HAVING 分组筛选条件
ORDER BY 排序列 DESC
LIMIT 10;
```

帮助理解的**逻辑处理顺序**（是概念模型，不等于 MySQL 的实际物理执行计划）：

```text
FROM / JOIN  →  WHERE  →  GROUP BY  →  HAVING
                                    ↓
                  SELECT  →  DISTINCT  →  ORDER BY  →  LIMIT
```

据此记住两件事：

- `WHERE` 负责**分组前过滤行**；`HAVING` 负责**分组后过滤组**。
- 同一查询层中，`SELECT` 给出的别名通常不能直接写到 `WHERE` 中，因为 `WHERE` 的逻辑处理阶段更早。

---

## 第 1 章：准备练习数据库（只需运行一次）

> **安全提醒**：下面使用新数据库 `sql_review_lab`，不会覆盖你已在学习中使用的 `shop`。不要在 `shop` 上照抄后面的 `UPDATE` 和 `DELETE`。如果你已经创建过这些表，请不要重复执行 `INSERT` 初始化语句，否则会出现主键重复错误。

### 1.1 创建并选择数据库

```sql
CREATE DATABASE IF NOT EXISTS sql_review_lab
  CHARACTER SET utf8mb4;

USE sql_review_lab;

SELECT DATABASE();           -- 检查当前数据库
SHOW TABLES;                 -- 查看当前库中的表
```

`USE` 是选择当前数据库，通常在建立连接后执行一次；重新建立连接后，应再确认当前使用哪个数据库。

### 1.2 建立三张表

**商品表**：保存每个商品的基本信息。

```sql
CREATE TABLE product (
    product_id CHAR(4) PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    product_type VARCHAR(32) NOT NULL,
    sale_price DECIMAL(10, 2) NOT NULL,
    purchase_price DECIMAL(10, 2),
    regist_date DATE
) ENGINE = InnoDB;
```

**商品类别表**：保存「类别 → 负责人」的对应关系。

```sql
CREATE TABLE category (
    product_type VARCHAR(32) PRIMARY KEY,
    manager_name VARCHAR(30) NOT NULL
) ENGINE = InnoDB;
```

**库存表**：同一商品可出现在不同仓库，因此由 `product_id + warehouse` **联合主键**标识一条库存记录。

```sql
CREATE TABLE product_stock (
    product_id CHAR(4) NOT NULL,
    warehouse VARCHAR(20) NOT NULL,
    quantity INT NOT NULL,
    PRIMARY KEY (product_id, warehouse),
    FOREIGN KEY (product_id) REFERENCES product(product_id),
    CHECK (quantity >= 0)
) ENGINE = InnoDB;
```

表的关系：

```text
category（商品类别）     product（商品）            product_stock（库存）
product_type       ←    product_type
                         product_id             → product_id
                                                 warehouse
                                                 quantity

一个类别 → 多个商品；一个商品 → 0 / 1 / 多条仓库库存。
```

`DECIMAL(10, 2)` 适合保存精确的小数金额：最多 10 位数字，其中小数部分 2 位；金额通常不建议用浮点型 `FLOAT`。

### 1.3 插入初始化数据

```sql
INSERT INTO product
    (product_id, product_name, product_type,
     sale_price, purchase_price, regist_date)
VALUES
    ('0001', 'T恤',     '衣服',     1000,  500, '2009-09-20'),
    ('0002', '运动T恤', '衣服',     4000, 2800, '2009-09-11'),
    ('0003', '铅笔',     '办公用品',  100,   30, '2009-09-11'),
    ('0004', '打孔器',   '办公用品',  500,  320, '2009-09-11'),
    ('0005', '菜刀',     '厨房用具', 3000, 1800, '2009-09-20'),
    ('0006', '高压锅',   '厨房用具', 6800, 5000, '2009-01-15'),
    ('0007', '毛巾',     '衣服',      500,  250, '2009-04-28'),
    ('0008', '圆珠笔',   '办公用品',  200, NULL, NULL);

INSERT INTO category (product_type, manager_name)
VALUES
    ('衣服', '服装组'),
    ('办公用品', '文具组'),
    ('厨房用具', '厨具组'),
    ('玩具', '待分配');

INSERT INTO product_stock (product_id, warehouse, quantity)
VALUES
    ('0001', 'A仓', 20),
    ('0001', 'B仓', 5),
    ('0002', 'A仓', 10),
    ('0003', 'A仓', 100),
    ('0004', 'B仓', 8),
    ('0005', 'A仓', 6),
    ('0006', 'A仓', 3),
    ('0007', 'B仓', 30);
```

检查：

```sql
SELECT COUNT(*) AS product_count FROM product;  -- 8
SELECT COUNT(*) AS type_count FROM category;     -- 4
SELECT COUNT(*) AS stock_rows FROM product_stock;-- 8
```

请留意数据里刻意设置的 4 个特点：

1. `product` 有 3 种商品类别，而 `category` 多了一个**没有商品的玩具类别**。
2. `0001 T恤` 在 A 仓、B 仓各有库存，体现**一对多**。
3. `0008 圆珠笔` 没有库存记录，方便观察 `LEFT JOIN`。
4. `0008` 的进货价和登记日期为 `NULL`，方便练习空值。

---

## 第 2 章：SELECT——最常用的查询

### 2.1 选择列与所有列

```sql
SELECT *
FROM product;
```

`*` 表示所有列。学习时可以使用；写应用程序时通常**显式列出需要的字段**，避免返回大量无用数据。

```sql
SELECT product_name, product_type, sale_price
FROM product;
```

**SQL 语法阅读法**：

- `SELECT`：要哪些列。
- `FROM`：从哪张表。
- `;`：语句结束。

### 2.2 WHERE：过滤符合条件的行

```sql
SELECT product_name, sale_price
FROM product
WHERE sale_price > 1000;
```

比较运算符：

| 运算符 | 含义 | 示例 |
|---|---|---|
| `=` | 等于 | `sale_price = 500` |
| `<>`、`!=` | 不等于（MySQL 支持两者） | `product_type <> '衣服'` |
| `>`、`>=` | 大于、大于等于 | `sale_price >= 500` |
| `<`、`<=` | 小于、小于等于 | `sale_price <= 1000` |

```sql
SELECT product_name, regist_date
FROM product
WHERE regist_date > '2009-04-28';
```

日期推荐使用 `YYYY-MM-DD`，避免不规范日期字符串引发歧义。

### 2.3 AND、OR、NOT 和括号

```sql
SELECT product_name, product_type, sale_price
FROM product
WHERE product_type = '办公用品'
   OR product_type = '厨房用具';
```

```sql
SELECT product_name, product_type, sale_price
FROM product
WHERE (product_type = '办公用品' OR product_type = '厨房用具')
  AND sale_price > 300;
```

- `AND`：两个条件都成立。
- `OR`：至少一个条件成立。
- `NOT`：对条件取反。
- **优先级：`NOT` > `AND` > `OR`**。混合使用时，建议加括号。

错误思路：

```sql
-- 容易误读为：限定两种类别且价格 > 300
WHERE product_type = '办公用品'
   OR product_type = '厨房用具'
  AND sale_price > 300;
```

实际上，`AND` 优先执行，因此「办公用品」即使价格不大于 300，也可能被选中。括号能明确表达意图。

### 2.4 IN、BETWEEN、LIKE

**IN：在多个指定值中选择。**

```sql
SELECT product_name, product_type
FROM product
WHERE product_type IN ('办公用品', '厨房用具');
```

等价于 `product_type = '办公用品' OR product_type = '厨房用具'`。

**BETWEEN：闭区间，包含两端。**

```sql
SELECT product_name, sale_price
FROM product
WHERE sale_price BETWEEN 500 AND 3000;
```

**LIKE：匹配字符串模式。**

```sql
SELECT product_name
FROM product
WHERE product_name LIKE '%T恤%';
```

| 通配符 | 含义 |
|---|---|
| `%` | 任意长度的字符串，也可以是 0 个字符 |
| `_` | 恰好 1 个字符 |

`LIKE '%T恤%'` 表示「名称中包含 T恤」。若需匹配字面的 `%` 或 `_`，应进行适当转义。

### 2.5 NULL：必须用 IS NULL / IS NOT NULL

```sql
SELECT product_name, purchase_price
FROM product
WHERE purchase_price IS NULL;
```

```sql
SELECT product_name, purchase_price
FROM product
WHERE purchase_price IS NOT NULL;
```

**不能写** `purchase_price = NULL` 或 `purchase_price <> NULL`。`NULL` 表示未知/缺失，普通比较的结果是 `UNKNOWN`，不能作为 `WHERE` 中为真的条件。

另一个高频坑：

```sql
-- 如果子查询结果中可能出现 NULL，NOT IN 需要格外小心
SELECT product_id
FROM product
WHERE product_id NOT IN (SELECT product_id FROM product_stock);
```

上面示例因 `product_stock.product_id` 定义为 `NOT NULL`，可以正常使用；但如果子查询列可能含 `NULL`，`NOT IN` 可能得不到预期结果。更稳妥的「找不存在」写法是 `NOT EXISTS`（见第 7 章）。

### 2.6 AS：计算列与别名

```sql
SELECT product_name,
       sale_price,
       purchase_price,
       (sale_price * 0.9 - purchase_price) AS profit
FROM product;
```

含义：打九折后的利润 = 售价 × 0.9 − 进货价。

- `(sale_price * 0.9 - purchase_price)` 是**计算表达式**。
- `AS profit` 是**输出列的别名**，不是修改表结构。
- 括号可以加在计算表达式上，但 `AS profit` 应写在表达式括号外。
- 对进货价为 `NULL` 的圆珠笔，利润结果也是 `NULL`。

结合条件：

```sql
SELECT product_name,
       product_type,
       (sale_price * 0.9 - purchase_price) AS profit
FROM product
WHERE product_type IN ('办公用品', '厨房用具')
  AND (sale_price * 0.9 - purchase_price) > 100;
```

为什么 `WHERE` 不写 `profit > 100`？因为在**同一层 SELECT** 中，`WHERE` 通常不能直接引用 `SELECT` 新定义的别名。如果不想重复表达式，可以改为派生表查询（见 7.3）。

### 2.7 DISTINCT：结果去重

```sql
SELECT DISTINCT product_type
FROM product;
```

返回各不相同的商品种类。

```sql
SELECT DISTINCT product_type, regist_date
FROM product;
```

**多列 DISTINCT 看的是整行组合是否重复**，不是分别给每一列去重。

### 2.8 ORDER BY：排序

```sql
SELECT product_name, sale_price
FROM product
ORDER BY sale_price DESC;
```

- `ASC`：升序，默认。
- `DESC`：降序。
- 多个排序条件从左到右依次生效。

```sql
SELECT product_name, product_type, sale_price
FROM product
ORDER BY product_type ASC, sale_price DESC;
```

MySQL 中想把 `NULL` 放最后的一种写法：

```sql
SELECT product_name, regist_date
FROM product
ORDER BY regist_date IS NULL ASC, regist_date DESC;
```

`IS NULL` 为真时可按 1 理解，假时为 0；先将非空放前，再按日期降序。

### 2.9 LIMIT：取部分结果

```sql
SELECT product_name, sale_price
FROM product
ORDER BY sale_price DESC, product_id ASC
LIMIT 3;
```

按价格排序后取前三条。若涉及分页，**先使用稳定、确定的 ORDER BY**，否则每页结果可能不稳定。

```sql
SELECT product_name
FROM product
ORDER BY product_id
LIMIT 5 OFFSET 3;   -- 跳过前 3 行，再取 5 行
```

MySQL 也支持 `LIMIT 3, 5`，其中第一个数字是偏移量，不是「第 3 到第 5 行」。

> SQLyog 可能在执行日志里追加 `LIMIT 0, 1000`：这是查询工具为结果集设置的行数限制，不一定是你写错了 SQL。

### 2.10 本章小练习

1. 查询所有办公用品的名称和售价，并按售价从高到低排序。
2. 查询 2009-04-28 之后登记、且售价不少于 1000 的商品。
3. 查询商品名称包含 `T恤` 的商品。
4. 查询「办公用品或厨房用具」中打九折后利润大于 100 的商品，输出 `product_name`、`product_type`、`profit`。

---

## 第 3 章：聚合函数、GROUP BY 和 HAVING

这是从「取出记录」到「统计结果」的关键过渡。

### 3.1 五种最常用聚合函数

| 函数 | 用途 | 记忆 |
|---|---|---|
| `COUNT(*)` | 统计行数 | 这组有几行 |
| `SUM(col)` | 求和 | 总金额 |
| `AVG(col)` | 平均值 | 平均售价 |
| `MAX(col)` | 最大值 | 最高售价 |
| `MIN(col)` | 最小值 | 最低售价 |

```sql
SELECT COUNT(*) AS total_count,
       COUNT(purchase_price) AS known_cost_count,
       SUM(sale_price) AS total_price,
       AVG(sale_price) AS avg_price,
       MAX(sale_price) AS max_price,
       MIN(sale_price) AS min_price
FROM product;
```

**注意 NULL：**

- `COUNT(*)` 统计全部行，这里是 8。
- `COUNT(purchase_price)` 只统计该列非 `NULL` 的行，这里是 7。
- `SUM`、`AVG` 忽略该列中的 `NULL`；如果所有值都是 `NULL`，结果通常为 `NULL`。

### 3.2 GROUP BY：一组一组地统计

问题：「每个商品类别有多少种商品？」

```sql
SELECT product_type, COUNT(*) AS product_count
FROM product
GROUP BY product_type;
```

思考步骤：

1. 先看 `FROM product`：原始 8 行。
2. `GROUP BY product_type`：把相同类别放进同一组。
3. `COUNT(*)`：对每一组分别统计。
4. `SELECT`：最终**每个类别显示一行**。

在初始化数据中，逻辑结果为：

| product_type | product_count |
|---|---:|
| 衣服 | 3 |
| 办公用品 | 3 |
| 厨房用具 | 2 |

没有 `ORDER BY` 时，返回行顺序**不保证**和上表一样。

### 3.3 一个类别同时统计多个指标

```sql
SELECT product_type,
       COUNT(*) AS product_count,
       SUM(sale_price) AS total_sale_price,
       ROUND(AVG(sale_price), 2) AS avg_sale_price
FROM product
GROUP BY product_type;
```

`ROUND(数值, 2)` 表示把结果保留到小数点后 2 位。

### 3.4 WHERE 和 HAVING 的区别

题目 A：「先排除低于 500 的商品，再计算每类平均售价。」

```sql
SELECT product_type, AVG(sale_price) AS avg_price
FROM product
WHERE sale_price >= 500
GROUP BY product_type;
```

`WHERE` **过滤原始行**，然后才分组。

题目 B：「找平均售价超过 2000 的商品类别。」

```sql
SELECT product_type, AVG(sale_price) AS avg_price
FROM product
GROUP BY product_type
HAVING AVG(sale_price) > 2000;
```

`HAVING` **过滤聚合后的组**。

可以同时用：

```sql
SELECT product_type,
       COUNT(*) AS product_count,
       AVG(sale_price) AS avg_price
FROM product
WHERE purchase_price IS NOT NULL
GROUP BY product_type
HAVING COUNT(*) >= 2
ORDER BY avg_price DESC;
```

翻译为自然语言：

> 先去掉进货价未知的商品 → 按类别分组 → 只保留至少两种商品的类别 → 按平均售价降序排列。

### 3.5 GROUP BY 最常见错误：选了不该选的列

**错误示例**：

```sql
SELECT product_type, product_name, COUNT(*)
FROM product
GROUP BY product_type;
```

每组可能有多个 `product_name`，数据库无法明确选择哪一个。MySQL 8 通常默认启用 `ONLY_FULL_GROUP_BY`，在无法证明列由分组列唯一确定时，会拒绝此类查询。

**正确示例**：

```sql
SELECT product_type, COUNT(*) AS product_count
FROM product
GROUP BY product_type;
```

简单记法：在常规分组查询中，`SELECT` 里的列应当是**分组列**，或者使用 `COUNT/SUM/AVG/MAX/MIN` 等**聚合后的结果**。若涉及函数依赖，MySQL 有更细致的规则，入门阶段先遵守这条即可。

### 3.6 高频综合题：总售价超过总进价 1.5 倍的类别

```sql
SELECT product_type,
       SUM(sale_price) AS total_sale,
       SUM(purchase_price) AS total_purchase
FROM product
GROUP BY product_type
HAVING SUM(sale_price) > SUM(purchase_price) * 1.5;
```

请区分：这里要求的是**各组的合计值比较**，不是逐条商品比较，所以用 `HAVING`，不是 `WHERE`。

> 业务提醒：`SUM(purchase_price)` 忽略缺失值。真实业务若不能接受缺失进价，需先校验数据完整性，而不是把缺失值默认为 0。

### 3.7 本章小练习

1. 统计每个类别的商品个数、最高售价和最低售价。
2. 找出商品数量至少为 3 的类别。
3. 找出「售价不低于 500」的商品中，平均售价超过 1500 的类别。
4. 解释为什么 `SELECT product_name, COUNT(*) ... GROUP BY product_type` 不可靠。

---

## 第 4 章：INSERT、UPDATE、DELETE——安全修改数据

> **本章会改变数据。** 请只在练习库中运行。建议每次修改前，先写一条对应的 `SELECT` 确认目标行。

### 4.1 INSERT：插入一条记录

```sql
INSERT INTO product
    (product_id, product_name, product_type,
     sale_price, purchase_price, regist_date)
VALUES
    ('0009', '鼠标', '办公用品', 800, 450, '2026-10-19');
```

然后验证：

```sql
SELECT *
FROM product
WHERE product_id = '0009';
```

**建议明确列名**，不要过度依赖建表时的列顺序。

一次插入多行：

```sql
-- 以下仅展示语法，避免与前面的主键重复，请不要直接再次执行
-- INSERT INTO product (product_id, ...) VALUES (...), (...);
```

### 4.2 UPDATE：更新已有记录

```sql
-- 第一步：先确认目标
SELECT product_id, product_name, sale_price
FROM product
WHERE product_id = '0009';

-- 第二步：再修改
UPDATE product
SET sale_price = 850
WHERE product_id = '0009';
```

多列同时更新：

```sql
UPDATE product
SET sale_price = 900,
    purchase_price = 500
WHERE product_id = '0009';
```

在 MySQL 中，你也可以修改为：`SET regist_date = NULL`，前提是该列允许 `NULL`。

**极其重要：**

```sql
UPDATE product SET sale_price = 900;
```

这条语句会把**整张表**所有商品售价改成 900，而不是只改某个商品。生产环境中，养成 `SELECT → UPDATE → 检查受影响行数` 的流程。

### 4.3 DELETE：删除记录

```sql
-- 先检查
SELECT * FROM product WHERE product_id = '0009';

-- 再删除
DELETE FROM product
WHERE product_id = '0009';
```

与 `UPDATE` 一样，遗漏 `WHERE` 会造成严重后果：

```sql
-- 危险：不要执行
-- DELETE FROM product;
```

### 4.4 DELETE 与 DROP / TRUNCATE 不一样

| 语句 | 意义 | 入门阶段的安全提醒 |
|---|---|---|
| `DELETE FROM t WHERE ...` | 删除满足条件的行 | `WHERE` 至关重要 |
| `DELETE FROM t` | 删除表中的所有行 | 事务性表中通常可回滚，前提是尚未提交 |
| `TRUNCATE TABLE t` | 清空表数据 | MySQL 中是 DDL，通常隐式提交、不能按普通 DML 事务回滚 |
| `DROP TABLE t` | 删除整张表和表结构 | 危险，通常隐式提交，不能按普通 DML 回滚 |

> `TRUNCATE` / `DROP` 可能还受外键约束等限制。练习时不要把它们当成普通 `DELETE`。

### 4.5 UPSERT 是什么（知道即可）

实际应用中经常遇到「存在则更新，不存在则插入」。MySQL 支持 `INSERT ... ON DUPLICATE KEY UPDATE`，但本阶段优先学好普通 `INSERT`、`UPDATE`、`DELETE`，后续写接口时再深入。

---

## 第 5 章：JOIN——把两张表连接起来

JOIN 是应用开发中非常高频的能力，因为实际业务数据往往存放在多张表里。

### 5.1 为什么需要 JOIN

目前有两张表：

`product`：

| product_id | product_name | product_type |
|---|---|---|
| 0001 | T恤 | 衣服 |
| 0003 | 铅笔 | 办公用品 |

`category`：

| product_type | manager_name |
|---|---|
| 衣服 | 服装组 |
| 办公用品 | 文具组 |
| 玩具 | 待分配 |

如果想显示「商品名称 + 类别负责人」，一个表里没有全部信息，需要关联。

### 5.2 INNER JOIN：只保留成功匹配的记录

```sql
SELECT p.product_name,
       p.product_type,
       c.manager_name
FROM product AS p
INNER JOIN category AS c
    ON p.product_type = c.product_type;
```

逐句解释：

- `FROM product AS p`：指定左边的数据来源，别名为 `p`。
- `INNER JOIN category AS c`：引入另一张表，别名为 `c`。
- `ON p.product_type = c.product_type`：两张表怎样对应。
- `p.product_name`：明确该列来自 `p` 表。

**结果特征**：只有 `ON` 条件成立的组合才会出现。因为玩具类别没有商品，所以这里不会产生单独的「玩具」记录。

`INNER JOIN` 也可简写为 `JOIN`。

### 5.3 LEFT JOIN：左表的行都要保留

```sql
SELECT p.product_id,
       p.product_name,
       s.warehouse,
       s.quantity
FROM product AS p
LEFT JOIN product_stock AS s
    ON p.product_id = s.product_id
ORDER BY p.product_id, s.warehouse;
```

观察三个现象：

1. `0001 T恤` 在两个仓库有库存，因此**出现两行**。
2. `0008 圆珠笔` 没有库存，但因为是 `LEFT JOIN`，仍**保留这一行**。
3. 对于圆珠笔，`s.warehouse` 和 `s.quantity` 显示为 `NULL`。

**一句话**：`INNER JOIN` 只要匹配项；`LEFT JOIN` 左边全部保留，右边匹配不上填 `NULL`。

### 5.4 RIGHT JOIN 和为什么通常只需熟练 LEFT JOIN

```sql
SELECT c.product_type, p.product_name
FROM product AS p
RIGHT JOIN category AS c
    ON p.product_type = c.product_type;
```

这会保留右边 `category` 的所有类别，包括玩具。

相同思路可交换表顺序，用 `LEFT JOIN` 表达：

```sql
SELECT c.product_type, p.product_name
FROM category AS c
LEFT JOIN product AS p
    ON c.product_type = p.product_type;
```

实际书写时，通常先确定「要保留谁」，把它放左边，用 `LEFT JOIN`。

### 5.5 非常重要：ON 和 WHERE 的位置

需求：「列出所有商品，并展示 A 仓库存；即使没有 A 仓库存也要保留商品。」

**正确**：筛选右表记录的条件放进 `ON`。

```sql
SELECT p.product_name, s.warehouse, s.quantity
FROM product AS p
LEFT JOIN product_stock AS s
    ON p.product_id = s.product_id
   AND s.warehouse = 'A仓';
```

容易写错：

```sql
SELECT p.product_name, s.warehouse, s.quantity
FROM product AS p
LEFT JOIN product_stock AS s
    ON p.product_id = s.product_id
WHERE s.warehouse = 'A仓';
```

后者的 `WHERE` 会排除右表未匹配时产生的 `NULL` 行，结果不再保留那些没有 A 仓库存的商品。

**记忆原则**：

- `ON`：规定**怎样匹配**，对 `LEFT JOIN` 也决定右表哪些行能参与匹配。
- `WHERE`：规定**最终保留哪些结果行**。

### 5.6 JOIN 为什么会产生「重复行」

```sql
SELECT p.product_id, p.product_name, s.warehouse
FROM product AS p
LEFT JOIN product_stock AS s
    ON p.product_id = s.product_id;
```

一个商品可以对应多个仓库，即「一对多」。所以 `0001` 会出现多次；不是数据库错误，也不应随意用 `DISTINCT` 掩盖。

如果目标是「每种商品的库存总量」，需要分组汇总：

```sql
SELECT p.product_id,
       p.product_name,
       COALESCE(SUM(s.quantity), 0) AS total_stock
FROM product AS p
LEFT JOIN product_stock AS s
    ON p.product_id = s.product_id
GROUP BY p.product_id, p.product_name
ORDER BY p.product_id;
```

`COALESCE(a, b)`：如果 `a` 不是 `NULL`，取 `a`；否则取 `b`。因此没有库存记录的圆珠笔会得到 0，而不是 `NULL`。

### 5.7 三表 JOIN（进阶但常用）

```sql
SELECT p.product_name,
       c.manager_name,
       COALESCE(SUM(s.quantity), 0) AS total_stock
FROM product AS p
JOIN category AS c
    ON p.product_type = c.product_type
LEFT JOIN product_stock AS s
    ON p.product_id = s.product_id
GROUP BY p.product_id, p.product_name, c.manager_name
ORDER BY p.product_id;
```

读 SQL 时，先逐步构造：

1. `product JOIN category`：取得负责组。
2. 再 `LEFT JOIN product_stock`：取得库存明细。
3. `GROUP BY`：每个商品重新收敛为一行。
4. `SUM`：计算商品总库存。

**额外提醒**：多个一对多关系同时 JOIN 可能使行数相乘，导致 `SUM` 重复累计。真实业务里最好先分别聚合，再 JOIN 聚合结果。

### 5.8 本章小练习

1. 查出所有商品的名称、类别与负责组。
2. 查出所有商品及其仓库库存，没有库存的商品也要显示。
3. 查出完全没有库存记录的商品。
4. 按商品统计总库存，包括库存为 0 的商品。
5. 找出未被任何商品使用的类别。

---

## 第 6 章：SQL 子查询——把一个查询放到另一个查询里

子查询最容易造成理解障碍。你只需按这个思路拆解：**子查询返回什么 → 外层怎样使用返回值。**

### 6.1 标量子查询：返回一个值

问题：「找出售价高于全体商品平均售价的商品。」

第一步，只算平均售价：

```sql
SELECT AVG(sale_price)
FROM product;
```

第二步，把这一结果放进外层条件：

```sql
SELECT product_name, sale_price
FROM product
WHERE sale_price > (
    SELECT AVG(sale_price)
    FROM product
);
```

这里子查询返回**一行一列**，因此可放在 `>` 右侧。若一个预期标量的子查询返回多行，MySQL 会报错（例如 `Subquery returns more than 1 row`）。

### 6.2 IN 子查询：返回一个值的集合

问题：「查出有库存记录的商品。」

```sql
SELECT product_id, product_name
FROM product
WHERE product_id IN (
    SELECT product_id
    FROM product_stock
);
```

内层可返回多行商品 ID；外层用 `IN` 判断自己的 ID 是否位于这些值中。

注意：这里 `0001` 虽有两条库存记录，外层仍只返回**一条商品记录**，因为 `IN` 是成员判断，不是 JOIN 展开行。

### 6.3 FROM 子查询：把结果当作临时表

问题：「显示各商品类别的商品数量，并筛选数量不少于 3 的类别。」

```sql
SELECT t.product_type, t.product_count
FROM (
    SELECT product_type, COUNT(*) AS product_count
    FROM product
    GROUP BY product_type
) AS t
WHERE t.product_count >= 3;
```

拆解：

1. 内层产生一张临时结果表：`product_type`、`product_count`。
2. `AS t` 给这张**派生表**设置别名（MySQL 要求此类派生表有别名）。
3. 外层对结果再筛选。此处 `t.product_count` 已经是内层输出列，所以在外层 `WHERE` 可以使用。

同样需求也可以更简洁地写成：

```sql
SELECT product_type, COUNT(*) AS product_count
FROM product
GROUP BY product_type
HAVING COUNT(*) >= 3;
```

**经验**：能用简单 `GROUP BY / HAVING` 写清楚时，不必强行嵌套。

### 6.4 关联子查询：内层依赖外层当前行

最重要的例题：「查询**每一类**中，售价高于**本类平均售价**的商品。」

```sql
SELECT p1.product_type,
       p1.product_name,
       p1.sale_price
FROM product AS p1
WHERE p1.sale_price > (
    SELECT AVG(p2.sale_price)
    FROM product AS p2
    WHERE p2.product_type = p1.product_type
);
```

**这段 SQL 最关键的只有一行：**

```sql
WHERE p2.product_type = p1.product_type
```

它的意思是：算平均价时，只看「与外层当前商品**同类别**」的那些记录。

以外层正在检查 `T恤` 为例：

1. `p1.product_type` 是「衣服」。
2. 内层只选择 `p2.product_type = '衣服'` 的记录。
3. 内层得到衣服类的平均售价。
4. 比较 T恤售价 `1000` 是否高于这一平均价。
5. 外层再检查下一种商品，比较基准随类别变化。

这是**理解模型**，不代表数据库引擎必然真的逐行执行内层；MySQL 优化器可能改写执行方案。

与「全体商品平均价」的区别：

```sql
-- 全局平均价：不依赖外层行
WHERE p1.sale_price > (SELECT AVG(sale_price) FROM product)

-- 本类平均价：依赖外层商品类型
WHERE p1.sale_price > (
    SELECT AVG(p2.sale_price)
    FROM product AS p2
    WHERE p2.product_type = p1.product_type
)
```

之前教程里的内层 `GROUP BY product_type` 在这个例子中**可以省略**：内层已经限定为一个商品类别，直接 `AVG()` 即可。

### 6.5 用 GROUP BY + JOIN 改写关联子查询

同一道题有另一种常见写法：

```sql
SELECT p.product_type, p.product_name, p.sale_price
FROM product AS p
JOIN (
    SELECT product_type, AVG(sale_price) AS avg_price
    FROM product
    GROUP BY product_type
) AS a
    ON p.product_type = a.product_type
WHERE p.sale_price > a.avg_price;
```

这时逻辑变为：

1. 先做「每类平均价表」`a`。
2. 将商品 `p` 与各类平均价 `a` 连接。
3. 比较当前商品售价与 `a.avg_price`。

**两种方式都应认识；基础阶段能独立写出其中一种即可。**

### 6.6 EXISTS / NOT EXISTS：检查是否存在

问题：「找出没有任何库存记录的商品。」

```sql
SELECT p.product_id, p.product_name
FROM product AS p
WHERE NOT EXISTS (
    SELECT 1
    FROM product_stock AS s
    WHERE s.product_id = p.product_id
);
```

解释：

- 对外层商品，检查库存表里**是否存在**匹配记录。
- `EXISTS`：至少找到一条，条件为真。
- `NOT EXISTS`：一条也找不到，条件为真。
- `SELECT 1` 只是习惯写法，因为这里关心**有没有行**，不是需要返回什么字段。

这条查询在初始化数据中返回 `0008 圆珠笔`。

### 6.7 子查询选型速查

| 题目里的信号 | 优先想到 |
|---|---|
| 「高于全体平均值」 | 标量子查询 `> (SELECT AVG(...))` |
| 「属于某些 ID 集合」 | `IN (SELECT ...)` |
| 「先统计，再筛选结果表」 | `FROM (SELECT ...) AS t`，或 `HAVING` |
| 「高于本类平均值」 | 关联子查询，或分组后 JOIN |
| 「有没有对应记录」 | `EXISTS` / `NOT EXISTS` |

### 6.8 本章小练习

1. 找出售价高于所有商品平均售价的商品。
2. 找出有库存记录的商品（不要重复显示同一商品）。
3. 找出每个类别中高于本类别平均售价的商品。
4. 找出没有库存的商品，并用 `LEFT JOIN` 再写一次。

---

## 第 7 章：事务——让多步操作一起成功或一起撤销

假设用户下单后要完成两件事：

1. 商品库存减少 1。
2. 新增一条订单记录。

如果库存已经扣了，但订单写入失败，就会发生数据不一致。**事务（Transaction）**把有关联的数据库操作组织成一个整体。

### 7.1 最重要的三个命令

```sql
START TRANSACTION;  -- 开启事务

-- 中间执行一条或多条 INSERT / UPDATE / DELETE

COMMIT;             -- 提交，确认本次事务的修改
```

```sql
START TRANSACTION;

-- 中间执行修改

ROLLBACK;           -- 回滚，撤销本次事务里尚未提交的修改
```

事务执行示意：

```text
START TRANSACTION
       │
       ▼
  修改记录（尚未提交）
       │
       ├── 检查通过 ──→ COMMIT   → 保留修改
       │
       └── 检查失败 ──→ ROLLBACK → 撤销修改
```

### 7.2 实验一：先修改，再回滚

请在**同一个 SQLyog 连接/会话**里，依次运行：

```sql
-- 看原库存
SELECT quantity
FROM product_stock
WHERE product_id = '0001' AND warehouse = 'A仓';

START TRANSACTION;

UPDATE product_stock
SET quantity = quantity - 2
WHERE product_id = '0001'
  AND warehouse = 'A仓';

-- 在当前事务里可以看到暂时的新库存
SELECT quantity
FROM product_stock
WHERE product_id = '0001' AND warehouse = 'A仓';

ROLLBACK;

-- 回滚后应该恢复为原来的数量
SELECT quantity
FROM product_stock
WHERE product_id = '0001' AND warehouse = 'A仓';
```

若该仓初始库存是 20，则事务内查到 18，回滚后又是 20。

> 运行时应确保 SQLyog 没有在中途断开或切换连接。事务属于**当前数据库会话**；另外，不要在实验事务中间执行 `CREATE TABLE`、`ALTER TABLE` 等可能导致隐式提交的 DDL。

### 7.3 实验二：提交后修改保留

```sql
START TRANSACTION;

UPDATE product_stock
SET quantity = quantity + 1
WHERE product_id = '0001'
  AND warehouse = 'A仓';

COMMIT;

SELECT quantity
FROM product_stock
WHERE product_id = '0001' AND warehouse = 'A仓';
```

这条示例会使库存永久增加 1。**不要反复运行**，否则每次都会继续增加。为了恢复原来的数字，可再开启事务并减 1、提交。

### 7.4 为什么需要 InnoDB

MySQL 常见存储引擎 `InnoDB` 支持事务。初始化练习表时使用了 `ENGINE = InnoDB`。不支持事务的表不能依赖 `ROLLBACK` 来撤销已经生效的修改。

查看表引擎：

```sql
SHOW TABLE STATUS FROM sql_review_lab LIKE 'product_stock';
```

### 7.5 autocommit：为什么平时 UPDATE 立即生效

MySQL 默认通常为自动提交：

```sql
SELECT @@autocommit;
```

若返回 `1`，则不在显式事务中的普通修改语句通常执行成功后自动提交。

这意味着：

```sql
UPDATE product_stock SET quantity = 9 WHERE product_id = '0001' AND warehouse = 'A仓';
ROLLBACK;
```

在默认 `autocommit=1` 且没有显式事务的情况下，**后面的 `ROLLBACK` 不能撤销前面已自动提交的 UPDATE**。

因此，练习事务时优先用：

```sql
START TRANSACTION;
-- ...
ROLLBACK; -- 或 COMMIT
```

### 7.6 业务型示例：扣库存 + 创建订单

先**建立一次**订单表（在事务实验之外执行 DDL）：

```sql
CREATE TABLE sales_order (
    order_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    product_id CHAR(4) NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES product(product_id),
    CHECK (quantity > 0)
) ENGINE = InnoDB;
```

一个**人工演示事务**：

```sql
START TRANSACTION;

-- 只在库存足够时扣减 2 件
UPDATE product_stock
SET quantity = quantity - 2
WHERE product_id = '0001'
  AND warehouse = 'A仓'
  AND quantity >= 2;

-- 必须立即查看 UPDATE 到底成功更新了几行
SELECT ROW_COUNT() AS affected_rows;

-- 只有 affected_rows = 1 时，才运行下面 INSERT，再 COMMIT。
-- 若 affected_rows = 0，直接 ROLLBACK，不要 INSERT。
INSERT INTO sales_order (product_id, quantity, unit_price)
VALUES ('0001', 2, 1000);

COMMIT;
```

**重要边界：这是一段供人工分步操作的示例，不是可以无条件一次性执行的生产代码。**

如果 `ROW_COUNT()` 为 0（库存不足或找不到商品），应该**停止执行后续语句并回滚**。真实应用中，这种条件判断、异常处理通常放在 Python 后端事务控制里；不能把 `SELECT ROW_COUNT()` 当作自动的 `IF`。

### 7.7 事务的 ACID（理解即可）

| 字母 | 特性 | 直观理解 |
|---|---|---|
| A | Atomicity 原子性 | 一组操作作为一个整体提交或回滚 |
| C | Consistency 一致性 | 事务前后满足数据库约束与业务规则 |
| I | Isolation 隔离性 | 并发事务之间的可见性与相互影响受隔离级别控制 |
| D | Durability 持久性 | 成功提交的结果应可靠保存 |

注意：事务本身不会自动替你检查所有业务规则。例如「库存不能卖成负数」，还需要条件更新、约束或后端逻辑。

### 7.8 SAVEPOINT：只撤销部分操作（选学）

```sql
START TRANSACTION;

UPDATE product_stock
SET quantity = quantity + 1
WHERE product_id = '0001' AND warehouse = 'A仓';

SAVEPOINT step1;

UPDATE product_stock
SET quantity = quantity + 3
WHERE product_id = '0001' AND warehouse = 'A仓';

ROLLBACK TO SAVEPOINT step1; -- 撤销 +3，保留前面的 +1
ROLLBACK;                    -- 最终撤销整个事务，包括 +1
```

### 7.9 事务使用的 5 条底线

1. 修改数据前确认 `WHERE`，防止误更新整表。
2. 需要多步同时成功的操作，用 `START TRANSACTION` 包裹。
3. 提交前检查关键条件与影响行数，失败则回滚。
4. 不要把 `CREATE / DROP / ALTER / TRUNCATE` 与希望回滚的 DML 随意混在同一事务里；MySQL 的很多 DDL 会**隐式提交**。
5. 在应用开发中由框架/数据库驱动管理提交与回滚，发生异常及时回滚，不要长时间持有未提交事务。

---

## 第 8 章：综合练习（先独立完成，再看答案）

> 推荐方式：每道题先用一句中文写清楚「查哪些列、从哪张表、筛选什么、要不要分组/连接」，再动手写 SQL。练习默认基于第 1 章**初始化数据**；若你已执行第 4、7 章修改操作，某些数字会不同。

### 8.1 练习题

**题 1｜SELECT + WHERE**  
查询售价超过 1000 的商品名称、类别和价格，按售价降序。

**题 2｜NULL**  
查出进货价未知的商品名称。

**题 3｜计算列 + 别名**  
查询办公用品与厨房用具中，打九折利润大于 100 的商品，返回 `product_name`、`product_type` 和 `profit`。

**题 4｜GROUP BY**  
统计各商品类别的商品数量及平均售价，平均售价显示两位小数。

**题 5｜HAVING**  
查询商品数量至少为 3 的类别。

**题 6｜INNER JOIN**  
查询每个商品的名称和负责组。

**题 7｜LEFT JOIN + GROUP BY**  
查询每个商品的库存总量；没有库存的商品显示 0。

**题 8｜NOT EXISTS**  
查询没有任何库存记录的商品名称。

**题 9｜标量子查询**  
查出高于全部商品平均售价的商品。

**题 10｜关联子查询**  
查出高于自身商品类别平均售价的商品。

**题 11｜事务**  
在一个事务中把 `0001` 的 A 仓库存临时增加 5，然后撤销，验证最终库存没有变化。

**题 12｜安全更新**  
如何仅把 `0003 铅笔` 的售价调整为 120，并先确认目标行？（不要漏掉 `WHERE`。）

### 8.2 参考答案

> 建议先独立完成，再展开此章节查看。以下 SQL 用于对照思路，**不是要求死记每个字符**。

**题 1**

```sql
SELECT product_name, product_type, sale_price
FROM product
WHERE sale_price > 1000
ORDER BY sale_price DESC;
```

**题 2**

```sql
SELECT product_name
FROM product
WHERE purchase_price IS NULL;
```

**题 3**

```sql
SELECT product_name,
       product_type,
       (sale_price * 0.9 - purchase_price) AS profit
FROM product
WHERE product_type IN ('办公用品', '厨房用具')
  AND (sale_price * 0.9 - purchase_price) > 100;
```

**题 4**

```sql
SELECT product_type,
       COUNT(*) AS product_count,
       ROUND(AVG(sale_price), 2) AS avg_price
FROM product
GROUP BY product_type;
```

**题 5**

```sql
SELECT product_type, COUNT(*) AS product_count
FROM product
GROUP BY product_type
HAVING COUNT(*) >= 3;
```

**题 6**

```sql
SELECT p.product_name, c.manager_name
FROM product AS p
INNER JOIN category AS c
    ON p.product_type = c.product_type;
```

**题 7**

```sql
SELECT p.product_id,
       p.product_name,
       COALESCE(SUM(s.quantity), 0) AS total_stock
FROM product AS p
LEFT JOIN product_stock AS s
    ON p.product_id = s.product_id
GROUP BY p.product_id, p.product_name;
```

**题 8**

```sql
SELECT p.product_name
FROM product AS p
WHERE NOT EXISTS (
    SELECT 1
    FROM product_stock AS s
    WHERE s.product_id = p.product_id
);
```

**题 9**

```sql
SELECT product_name, sale_price
FROM product
WHERE sale_price > (
    SELECT AVG(sale_price)
    FROM product
);
```

**题 10**

```sql
SELECT p1.product_name, p1.product_type, p1.sale_price
FROM product AS p1
WHERE p1.sale_price > (
    SELECT AVG(p2.sale_price)
    FROM product AS p2
    WHERE p2.product_type = p1.product_type
);
```

**题 11**

```sql
SELECT quantity FROM product_stock
WHERE product_id = '0001' AND warehouse = 'A仓';

START TRANSACTION;
UPDATE product_stock
SET quantity = quantity + 5
WHERE product_id = '0001' AND warehouse = 'A仓';
ROLLBACK;

SELECT quantity FROM product_stock
WHERE product_id = '0001' AND warehouse = 'A仓';
```

**题 12**

```sql
SELECT product_id, product_name, sale_price
FROM product
WHERE product_id = '0003';

UPDATE product
SET sale_price = 120
WHERE product_id = '0003';
```

如需不影响其他练习结果，可以再次用带 `WHERE` 的 `UPDATE` 恢复成 100。

---

## 第 9 章：报错速查与排错顺序

### 9.1 常见报错

| 现象 / 错误码 | 常见原因 | 优先检查 |
|---|---|---|
| `1054 Unknown column` | 列名写错，如 `scale_price` | `DESC product;`，确认是 `sale_price` |
| `1055 ... GROUP BY` | 查询了既未分组、又未聚合且不满足函数依赖规则的列 | `SELECT` 中是否混入 `product_name` |
| `1062 Duplicate entry` | 插入了重复的主键或唯一键 | 是否重复运行初始化 `INSERT` |
| `1146 Table doesn't exist` | 表不存在，或当前库不对 | `SELECT DATABASE(); SHOW TABLES;` |
| `1046 No database selected` | 没有选择数据库 | `USE sql_review_lab;` |
| `1064 ... syntax` | SQL 语法错误 | 逗号、括号、引号、关键字位置 |
| `1242 Subquery returns more than 1 row` | 用 `=`、`>` 等要求单值的位置，子查询返回多行 | 改成 `IN` 或让子查询只返回一个值 |
| `1451/1452` 外键错误 | 删除被引用记录，或引用不存在的父记录 | 查看表间关系与删除/插入顺序 |

### 9.2 按这个顺序调试 SQL

1. **检查当前库**：`SELECT DATABASE();`
2. **查看表和字段**：`SHOW TABLES;`、`DESC product;`
3. **删掉复杂条件**：先让 `SELECT ... FROM ...` 跑通。
4. **单独运行子查询**：检查它返回几行、几列、什么类型的值。
5. **检查 JOIN 匹配行数**：是否存在一对多导致行数增加。
6. **检查 GROUP BY**：`SELECT` 中的非聚合列是否合理。
7. **检查 NULL 和括号**：是否误用 `= NULL`，或 `AND / OR` 优先级不符合预期。
8. **修改数据前先运行 SELECT**：先看将被更新或删除的是哪些行。

### 9.3 「看懂 SQL」的实用翻译法

对任何复杂查询，按以下顺序用中文说出：

```text
1. FROM：从哪一张（或哪几张）表取数据？
2. JOIN/ON：按什么条件把表连起来？
3. WHERE：先删掉不符合条件的哪些行？
4. GROUP BY：按什么维度统计？
5. HAVING：保留什么样的组？
6. SELECT：最后显示哪些字段 / 计算值？
7. ORDER BY / LIMIT：按什么排序，取前几行？
```

读关联子查询时**再多问一句**：子查询有没有引用外层表别名？有的话，内层结果可能依赖当前外层行。

---

## 第 10 章：一页 SQL 速查表（长期复习）

### 10.1 单表查

```sql
SELECT DISTINCT product_type, product_name
FROM product
WHERE sale_price >= 500
ORDER BY sale_price DESC
LIMIT 10;
```

### 10.2 常见条件

```sql
WHERE product_type = '衣服'
WHERE product_type IN ('衣服', '办公用品')
WHERE sale_price BETWEEN 500 AND 3000
WHERE product_name LIKE '%T恤%'
WHERE purchase_price IS NULL
WHERE sale_price > 500 AND product_type = '衣服'
```

### 10.3 分组统计

```sql
SELECT product_type, COUNT(*) AS n, AVG(sale_price) AS avg_price
FROM product
WHERE sale_price > 0
GROUP BY product_type
HAVING COUNT(*) >= 2
ORDER BY avg_price DESC;
```

### 10.4 增删改（务必检查 WHERE）

```sql
INSERT INTO product (product_id, product_name, product_type, sale_price)
VALUES ('0010', '示例商品', '衣服', 100);

UPDATE product SET sale_price = 120 WHERE product_id = '0010';

DELETE FROM product WHERE product_id = '0010';
```

> 这里是便于记忆的完整演示：首次执行会插入 `0010`，随后更新并删除；不要只复制中间一句去批量修改真实表。

### 10.5 多表关联

```sql
SELECT p.product_name, c.manager_name
FROM product AS p
LEFT JOIN category AS c ON p.product_type = c.product_type;
```

### 10.6 子查询

```sql
SELECT product_name, sale_price
FROM product
WHERE sale_price > (SELECT AVG(sale_price) FROM product);
```

### 10.7 事务

```sql
START TRANSACTION;
-- 执行需要同时成功的 DML 操作
COMMIT;             -- 成功时
-- 或 ROLLBACK;      -- 失败时
```

### 10.8 高频区别集中记忆

| 你容易混淆的两项 | 最短区分 |
|---|---|
| `WHERE` vs `HAVING` | 前者筛选**行**，后者筛选**组** |
| `COUNT(*)` vs `COUNT(col)` | 前者统计行，后者统计该列非空值 |
| `DISTINCT` vs `GROUP BY` | 前者消除结果重复行，后者用于按组组织和聚合 |
| `INNER JOIN` vs `LEFT JOIN` | 前者只保留匹配，后者保留全部左表行 |
| `ON` vs `WHERE` | 前者定义连接匹配，后者过滤结果行 |
| `IN` vs `EXISTS` | 前者查是否属于值集合，后者查是否存在匹配行 |
| `COMMIT` vs `ROLLBACK` | 前者提交，后者撤销当前未提交事务的修改 |
| `DELETE` vs `DROP TABLE` | 前者删除数据行，后者删除表本身 |
| `NULL` vs `0` | 缺失/未知不等于数字零 |

---

## 第 11 章：怎么复习，才能减少对讲解的依赖

**每学完一章，只做三件事：**

1. **自己解释**：挑一个例子，把 `FROM → WHERE → GROUP BY → HAVING → SELECT` 的含义说出来。
2. **修改条件再运行**：例如把 `>` 改成 `>=`、把 `INNER JOIN` 改为 `LEFT JOIN`，预测结果有什么变化。
3. **不看答案重写**：隔一天重做本章练习；能独立写出相似查询，比记住教程原句更有价值。

建议复习节奏：

| 时间 | 内容 | 标准 |
|---|---|---|
| 第 1 天 | 单表查询 + 条件 + 排序 | 独立完成第 2 章练习 |
| 第 2 天 | 聚合 + GROUP BY + HAVING | 分清 `WHERE` 与 `HAVING` |
| 第 3 天 | INSERT / UPDATE / DELETE | 修改前会先 SELECT，知道事务的作用 |
| 第 4 天 | INNER JOIN / LEFT JOIN | 能解释 `ON`、一对多、`NULL` 行 |
| 第 5 天 | 标量 + IN + 关联子查询 | 能拆解内层和外层 |
| 第 6 天 | 事务 + ACID | 会演示 `COMMIT` 与 `ROLLBACK` |
| 第 7 天 | 第 8 章综合题，闭卷练习 | 12 题中多数可独立完成；错误能定位原因 |

### 与后续开发的关系

- **FastAPI + SQLAlchemy**：把这里的查询与增删改写成后端接口，逐步学习 ORM、Session 与事务边界。
- **PostgreSQL**：基础 SQL 思路通用，但 `LIMIT`、自增列、部分函数、数据类型和 DDL 等存在方言差异，不要机械复制 MySQL 专有写法。
- **Agent 应用**：商品/用户/任务状态等权威业务数据通常使用数据库表；长期记忆与文档检索是另外的模块。SQL 能帮你读懂真实业务数据存取与一致性问题。

**最终目标**：给你一张简单业务需求单，你能自己判断是单表查询、分组、连接还是子查询，并能在修改数据时考虑事务和安全性。

---

## 参考来源与说明

这是一份以实用学习目标重新组织的**原创讲解和练习材料**，不是对原仓库全文的搬运。语法主题参考了 Datawhale 的 [wonderful-sql](https://github.com/datawhalechina/wonderful-sql)：

- [ch01 初识数据库](https://github.com/datawhalechina/wonderful-sql/blob/main/ch01_%E5%88%9D%E8%AF%86%E6%95%B0%E6%8D%AE%E5%BA%93.md)：建表、插入、更新、删除。
- [ch02 基础查询与排序](https://github.com/datawhalechina/wonderful-sql/blob/main/ch02_%E5%9F%BA%E7%A1%80%E6%9F%A5%E8%AF%A2%E4%B8%8E%E6%8E%92%E5%BA%8F.md)：查询、条件、聚合与分组。
- [ch03 复杂一点的查询](https://github.com/datawhalechina/wonderful-sql/blob/main/ch03_%E5%A4%8D%E6%9D%82%E4%B8%80%E7%82%B9%E7%9A%84%E6%9F%A5%E8%AF%A2.md)：子查询、关联子查询。
- [ch04 集合运算](https://github.com/datawhalechina/wonderful-sql/blob/main/ch04_%E9%9B%86%E5%90%88%E8%BF%90%E7%AE%97.md)：表连接（JOIN）。
- [MySQL 8.0 官方文档：START TRANSACTION / COMMIT / ROLLBACK](https://dev.mysql.com/doc/refman/8.0/en/commit.html)：事务控制与自动提交。
- [MySQL 8.0 官方文档：InnoDB 与事务](https://dev.mysql.com/doc/refman/8.0/en/innodb-autocommit-commit-rollback.html)：自动提交和 InnoDB 行为。

本文示例数据专门为复习设计，与原仓库的 `shop.product` 数据不完全相同；请不要混用两套数据来核对预期结果。
