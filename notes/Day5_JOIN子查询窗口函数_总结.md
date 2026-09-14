# Day 5 学习总结：JOIN / 子查询 / 窗口函数 / AS

## 一、为什么需要 JOIN

- 当需要的信息分散在两张表里（比如`stocks`存价格、`company_info`存公司信息），需要通过某个共同字段（如ticker）把两张表"拼"在一起
- 语法结构（"表名.列名"用于消除歧义，因为两张表可能有同名列）：
```sql
SELECT stocks.ticker, stocks.close, company_info.company_name
FROM stocks
INNER JOIN company_info
ON stocks.ticker = company_info.ticker;
```

## 二、INNER JOIN vs LEFT JOIN

- **INNER JOIN**：只保留两张表都能匹配上的行（交集）。右表没有对应记录的左表行，会被整行剔除
- **LEFT JOIN**：以FROM后面那张表（左表）为主，左表全部行都保留；右表没匹配上的，对应列填NULL，但这一行本身不删除
- 判断依据：题目要不要求"某一边的数据必须全保留，即使没有匹配"，要则LEFT，否则INNER
- 特殊情况：LEFT JOIN之后，如果WHERE条件对副表字段做了非NULL限制（如`WHERE b.sector='Technology'`），效果会退化成等同于INNER JOIN，因为NULL永远不满足这类条件

## 三、多对多关联：行数会"相乘"，不是简单相加

- 如果关联键在两张表里都出现多次（比如某ticker在评级表里有多条记录），JOIN后该ticker的行数 = 左表该ticker行数 × 右表该ticker行数
- 容易踩的坑：`COUNT(*)`这类计数会因为行数膨胀而失真；`AVG()`因为重复的是同样的值，结果不受影响
- 实际处理思路：如果目的是统计，往往需要先想清楚"需要哪张表的哪个粒度"，可能要先对一张表聚合后再JOIN，避免被意外膨胀的行数污染结果

## 四、AS：给表/列起别名

```sql
FROM stocks AS s          -- AS可省略，直接写 FROM stocks s 效果一样
SELECT s.ticker, s.close
FROM stocks AS s
INNER JOIN company_info AS c
ON s.ticker = c.ticker;
```
- 给表起别名：JOIN多表时的标配写法，简化"表名.列名"的啰嗦
- 给列起别名：让聚合/窗口函数的结果列显示更清晰
```sql
SELECT ticker, AVG(close) AS avg_close FROM stocks GROUP BY ticker;
```

## 五、`a.*` 语法：只要某一张表的全部列
```sql
SELECT a.*, b.sector
FROM stocks AS a
LEFT JOIN company_info AS b
ON a.ticker = b.ticker;
```
- `SELECT *`：所有参与JOIN的表的所有列
- `SELECT a.*`：只要别名a代表的这张表的全部列
- 常用组合：主表用`.*`全要，副表只挑需要的字段单独列出

## 六、子查询（Subquery）：把一句查询的结果，当成另一句查询里"一个值"来用

```sql
SELECT *
FROM stocks
WHERE close > (SELECT AVG(close) FROM stocks WHERE ticker = 'MSFT');
```
- 子查询必须用 `()` 包裹，通常写在期待"一个值"的位置（如WHERE的比较对象）
- 子查询先被执行算出一个具体数值，再代入外层查询使用
- 区别于之前误用的`AS`：AS只是"起别名"，不能把结果存成临时表供下一句独立使用；子查询是直接嵌套，一步到位

## 七、窗口函数：既做聚合计算，又不压缩原始行数

### 与 GROUP BY 的核心区别
- `GROUP BY`：分组后**压缩行数**，只保留每组一行的汇总结果
- 窗口函数 `OVER (PARTITION BY ...)`：分组计算，但**保留全部原始行**，计算结果作为新列贴在每一行上

### 基本写法
```sql
SELECT ticker, date, close,
       AVG(close) OVER (PARTITION BY ticker) AS avg_close_per_ticker
FROM stocks;
```
- `OVER(...)`：窗口函数标志，紧跟在具体函数（如AVG、SUM、RANK）后面，表示"这个计算按窗口方式进行，不压缩行数"
- `PARTITION BY ticker`：定义分组范围，逻辑上类似GROUP BY的分组，但不压缩行数

### RANK() 排名
```sql
RANK() OVER (PARTITION BY ticker ORDER BY close DESC) AS price_rank
```
- `RANK()`括号为空：因为排名不是对某列数值做四则运算，而是依据"顺序"，所以排序规则放在OVER里的ORDER BY，而不是塞进函数自己的括号
- OVER内的`ORDER BY`默认升序，需要降序必须显式加`DESC`（这条规则和普通ORDER BY一致）
- 对照AVG(close)：AVG需要"对哪列数值做运算"，所以列名写在函数自己括号里；这是两者的本质区别

### OVER 与算术运算混合时的位置规则
```sql
-- 正确：OVER紧贴SUM(volume)，四则运算包裹在外面
close / (SUM(volume) OVER (PARTITION BY ticker)) * 100

-- 错误：OVER不能贴在整个算术表达式后面
close / (SUM(volume) * 100) OVER (PARTITION BY ticker)   -- ❌ 语法错误
```
- 核心规则：`OVER(...)` 只能紧跟在具体的聚合/窗口函数本身后面，不能跟在"函数参与的更大算术表达式"后面
- 可以把 `SUM(volume) OVER(...)` 看作已经算好的一个数值，剩余的加减乘除都是普通运算，包裹在外面

### 对照 pandas（预习，尚未正式学）
```python
df.groupby('ticker')['close'].transform('mean')      # 对应 AVG(...) OVER (PARTITION BY ticker)
df.groupby('ticker')['close'].rank(ascending=False)   # 对应 RANK() OVER (PARTITION BY ticker ORDER BY close DESC)
```

---

## 八、其他补充规则

### 引号使用规则
- 列名、表名、别名（数据库结构里已存在的名字）→ **不加引号**
- 字符串具体的值 → **加引号**（如 `WHERE ticker = 'MSFT'`）
- 数字值 → 不加引号（如 `WHERE close > 300`）
- 与Python的"数字不加引号、文字加引号"是同一套逻辑的延伸

### SQL 完整执行顺序（含JOIN和窗口函数）
```
FROM → JOIN → WHERE → GROUP BY → 聚合函数 → HAVING → SELECT(含窗口函数) → ORDER BY → LIMIT
```
- WHERE在窗口函数（写在SELECT里）之前执行，所以WHERE筛选会先减少行数，窗口函数是在筛选后的结果上计算的

---

## 九、学习心得记录
- 复杂SQL（多表JOIN+子查询+窗口函数混合）第一次手写很容易在细节处出错（漏逗号、OVER位置、SELECT多余的列），这是正常的熟练度问题，不代表核心逻辑没理解
- 真实业务SQL通常比练习题更复杂，实际工作中是分步骤写、分步调试验证，很少有人一次性写对复杂查询

---

## 待续：Day 6 预告
GitHub 基础：仓库创建、commit/push/pull、.gitignore、基本工作流
