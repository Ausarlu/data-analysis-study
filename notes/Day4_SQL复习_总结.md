# Day 4 学习总结：SQL 复习① —— SELECT/WHERE/GROUP BY/ORDER BY

## 一、数据库基础概念（对照pandas理解）

| pandas 世界 | 数据库世界 |
|---|---|
| DataFrame（内存中的表格） | Table 表（硬盘上持久化的表格） |
| 用pandas方法操作（`.loc`、筛选等） | 用SQL语句操作（SELECT、WHERE等） |
| `pd.read_csv()` 每次读入内存 | 数据本来就存在数据库里，不用每次"读入" |

- 数据库存在的核心价值：数据量极大时（几百万/千万行），在数据源头直接筛选聚合，只把需要的结果传出来，不用把全部数据读进内存
- Navicat 只是"客户端工具"，不是数据库本身；准备用 SQLite（不需要装服务、就是一个文件），后续 Day 13 会用它把数据存进去练习

---

## 二、SELECT / FROM / WHERE 基础结构

```sql
SELECT ticker, close
FROM stocks
WHERE close > 200;
```

- `SELECT`：选择要看的列，多列用逗号分隔；`SELECT *` 表示所有列
- `FROM`：指定从哪张表取数据（pandas里这一步是隐含在df本身里的，SQL必须显式写出）
- `WHERE`：筛选行，筛选的是**原始表里本来就有的列**
- 书写顺序固定：SELECT → FROM → WHERE，不能打乱

### 与 pandas 的对照
| pandas | SQL |
|---|---|
| `df[['ticker','close']]` | `SELECT ticker, close FROM stocks` |
| `df[df['close']>200]` | `WHERE close > 200` |
| `&`（且）/ `\|`（或） | `AND` / `OR`（直接用英文单词，不需要广播符号） |
| `==` 判断相等 | `=` 判断相等（注意SQL里单个`=`就是判断相等，跟Python的赋值`=`不是一回事） |
| 字符串用单/双引号都行 | 字符串标准写法用单引号 `'MSFT'` |

### 不等于：`<>` vs `!=`
- `<>` 是ANSI SQL标准写法，所有数据库都保证支持，**推荐优先使用**
- `!=` 是多数现代数据库（含MySQL/SQLite）也支持的扩展写法，但不是标准的一部分

---

## 三、ORDER BY：排序

```sql
ORDER BY close;          -- 默认升序(ASC，可省略)
ORDER BY close DESC;     -- 降序需显式写DESC
ORDER BY ticker, close DESC;   -- 多列排序：先按ticker，ticker相同再按close降序
```
对照pandas：`df.sort_values('close', ascending=False)`

---

## 四、GROUP BY：分组聚合

```sql
SELECT ticker, AVG(close)
FROM stocks
GROUP BY ticker;
```

- 先把数据按某列的值**分组**（如按ticker分成5堆），再对每组分别用**聚合函数**计算
- 常用聚合函数：`AVG`（平均）、`SUM`（求和）、`COUNT`（计数）、`MAX`（最大）、`MIN`（最小）
- **规则**：SELECT里只能出现"参与分组的字段"或"包在聚合函数里的列"，不能出现既没分组又没聚合的裸列（逻辑上这类列在组内有多个值，无法确定该显示哪一个）
- 分组字段（如ticker）不是"语法必须"出现在SELECT里，但不写的话结果会丢失"每个数值对应哪一组"这个信息，实际使用中几乎总要带上
- 对照pandas：`df.groupby('ticker')['close'].mean()`

---

## 五、HAVING：对分组聚合后的结果再筛选

### 关键问题：为什么不能直接用WHERE筛"COUNT>5"这种条件？

**SQL 实际执行顺序**（与书写顺序不同，需牢记）：
```
FROM → WHERE → GROUP BY → 聚合函数计算 → HAVING → SELECT → ORDER BY → LIMIT
```
- `WHERE` 在分组聚合**之前**执行，此时聚合结果（如COUNT、AVG）还不存在，所以WHERE管不了聚合结果
- `HAVING` 专门用在GROUP BY**之后**，对聚合算出来的结果做筛选

### 记忆方法
- 条件涉及**表里原始的列** → 用 `WHERE`
- 条件涉及**聚合函数算出来的结果** → 用 `HAVING`

```sql
SELECT ticker, COUNT(ticker)
FROM stocks
WHERE close > 300
GROUP BY ticker
HAVING COUNT(ticker) > 5
ORDER BY COUNT(ticker) DESC;
```

---

## 六、LIMIT：限制返回行数

```sql
ORDER BY SUM(volume) DESC
LIMIT 3;
```
- 放在语句最后（ORDER BY之后），表示只取结果的前几行
- 几乎总是配合ORDER BY使用——先排好序，"前几名"才有意义
- 对照pandas：`.sort_values().head(3)`

---

## 七、LIKE + %：字符串模糊匹配

```sql
WHERE date LIKE '2026-01%'    -- 以2026-01开头
WHERE date LIKE '%01-02'       -- 以01-02结尾
WHERE date LIKE '%2026%'       -- 包含2026即可，位置不限
```
- `LIKE`：模糊匹配关键字，区别于`=`的精确匹配
- `%`：通配符，代表"任意数量的任意字符"（含0个）
- 注意与Python字符串格式化里的`%`（如`'%s' % name`）不是一回事，SQL的`%`专指模糊匹配通配符
- 对照pandas：`.str.contains()`（包含）、`.str.startswith()`（开头），这块Python侧的写法计划里暂未展开，后续涉及爬虫/文本处理场景时会细讲

---

## 八、SQL 完整语句的书写顺序与执行顺序（两者不同，需分清）

**书写顺序**：
```sql
SELECT ...
FROM ...
WHERE ...
GROUP BY ...
HAVING ...
ORDER BY ...
LIMIT ...;
```

**实际执行顺序**：`FROM → WHERE → GROUP BY → 聚合 → HAVING → SELECT → ORDER BY → LIMIT`

## 九、SQL 缩进规范
- 与Python字典类似：换行/缩进纯粹是排版给人看的，不是语法要求，压缩成一行效果完全相同
- 约定俗成：每个主要关键字独占一行；关键字习惯大写（SELECT/FROM/WHERE），列名/表名习惯小写
- 与Python的关键区别：SQL**没有**"语法性缩进"这个概念，缩进/换行错了不会导致报错（Python的if/for缩进错了会报错，SQL不会）

---

## 待续：Day 5 预告
JOIN（多表关联）、子查询、窗口函数；`AS`语法（给列/表起别名，在JOIN多表时尤为常用）；并对照pandas的等价写法（merge ↔ JOIN 等）
