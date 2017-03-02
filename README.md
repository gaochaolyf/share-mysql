## MySQL慢查询优化

@(索引)[MySQL EXPLAIN|精确查询]

**MySQL**系统本身，我们可以使用工具来优化数据库的性能，通常有三种：
 
- **使用索引** ：索引的实现通常使用B树及其变种B+树；
- **使用EXPLAIN分析查询** ：EXPLAIN是MySQL的一个关键字，当在SELECT语句前加上该关键字，即可知道：MySQL是如何执行SQL查询的，如：查询时是否正确使用索引，MySQL是否采用最优的表连接；
- **调整MySQL的内部配置** ：不太懂。

-------------------

[TOC]

### 查询的逻辑执行顺序
``` 
FROM < left_table>
 < join_type>  JOIN < right_table>   ON < join_condition>
WHERE < where_condition>
GROUP BY < group_by_list>
WITH {cube | rollup}
HAVING < having_condition>
SELECT   DISTINCT < top_specification>  < select_list>
ORDER BY < order_by_list>  
```
>标准的SQL 的解析顺序为:
- FROM 子句 组装来自不同数据源的数据
- WHERE 子句 基于指定的条件对记录进行筛选
- GROUP BY 子句 将数据划分为多个分组
- 使用聚合函数进行计算
- 使用HAVING子句筛选分组
- 计算所有的表达式
- 使用ORDER BY对结果集进行排序

### 检查索引
>索引能增加查询效率 同样会影响数据在插入和更新操作时的效率 所以合理的建立索引非常有必要 在SQL语句的WHERE和JOIN部分中用到的所有字段上，都应该加上索引。

合理的建立索引的建议：

(1)  越小的数据类型通常更好：越小的数据类型通常在磁盘、内存和CPU缓存中都需要更少的空间，处理起来更快。 

(2)  简单的数据类型更好：整型数据比起字符，处理开销更小，因为字符串的比较更复杂。在MySQL中，应该用内置的日期和时间数据类型，而不是用字符串来存储时间；以及用整型数据类型存储IP地址。

(3)  尽量避免NULL：应该指定列为NOT NULL，除非你想存储NULL。在MySQL中，含有空值的列很难进行查询优化，因为它们使得索引、索引的统计信息以及比较运算更加复杂。你应该用0、一个特殊的值或者一个空串代替空值

### 只选择需要的字段
> 不用或少用 SELECT *
> 额外的字段通常会增加返回数据的纹理，从而导致更多的数据被返回。

- 在查询中包含列越少，IO开销就越小
- sql总耗时 分查询时间和抓取数据时间 查询时间不变的情况下 抓取的数据量越多越大也会增加sql总耗时

### MySQL EXPLAIN 命令详解
>MySQL的EXPLAIN命令用于SQL语句的查询执行计划(QEP)。这条命令的输出结果能够让我们了解MySQL 优化器是如何执行SQL 语句的。这条命令并没有提供任何调整建议，但它能够提供重要的信息帮助你做出调优决策。

#### EXPLAIN 输出列信息
| 字段      |    描述 |
| :-------- | :--------|
|id	|SQL语句执行的顺序|
|select_type|	SELECT的类型|
|table|	查询出的行所用的表|
|type|	连接类型|
|possible_keys|	可能使用的索引|
|key|	实际使用的索引|
|key_len	|使用的索引的长度|
|ref	|使用哪一列或常数与索引进行比较|
|rows	|估计需要查找的条数|
|extra	|My解决查询的详细信息|

一般情况下看：
- type 如果是**ALL**效率最差，则需要优化代表全表扫描，index的话则是运用到了索引，range 这个值表示所有符合一个给定范围值的索引行都被用到，还有fulltext 、ref_or_null 、index_merge 、unique_subquery、index_subquery 以及index。
- rows 这个数表示MySQL要遍历多少数据才能找到，在innodb上是不准确的。
- key 用到哪个索引，没用到索引为NULL
- Extra
	- 如果是Using index，这意味着信息只用索引树中的信息检索出的，这比扫描整个表要快。
	- 如果是where used，就是使用上了where限制。
	- 如果是impossible where 表示用不着where，一般就是没查出来啥。
	- 如果此信息显示**Using filesort**或者**Using temporary**的话会很吃力，WHERE和ORDER BY的索引经常无法兼顾，如果按照WHERE来确定索引，那么在ORDER BY时，就必然会引起Using filesort，这就要看是先过滤再排序划算，还是先排序再过滤划算。

#### 调优实践
http://slowlogmanage.qccr.com/mysql/slow_log/
发现 insurance 库经常报这个慢查询

|Server IP| DB|user@host|查询行数|返回行数|锁表时间(ms)|耗时(ms)|发生时间|慢日志语句|
| :--- | :-----| :--- | :-----|:--- | :-----|:--- | :-----|:---- |
|192.168.69.150|insurance|qccr@192.168.69.36| 481986|1|0.086| 	1141.749| 	2017-02-24 11:02:11|SELECT COUNT(*) FROM (SELECT * FROM `ins_log_offer` WHERE  1  GROUP BY `user_token`) `c`;| 

``` 
EXPLAIN SELECT
	COUNT(*)
FROM
	(
		SELECT
			*
		FROM
			`ins_log_offer`
		WHERE
			1
		GROUP BY
			`user_token`
	) `c`;
```
在192.168.5.110上 EXPLAIN 该语句结果：

| id | select_type| table| type| possible_keys| key | key_len| ref|rows| Extra |
| :--- | :---| :---|:---|:---|:---|:---|:---|:---|:---|                   
|  1 | PRIMARY     | `<`derived2`>`| ALL  | NULL          | NULL | NULL    | NULL | 8997 | NULL    |
|  2 | DERIVED     | ins_log_offer | ALL  | NULL   | NULL | NULL    | NULL | 8997 | Using temporary; Using filesort |

可以发现结果出现：
1. type 字段为ALL效率最差
2. Extra 字段为Using temporary; Using filesort
	- Using temporary 为了解析查询，MySQL需要创建一个临时表来保存记录。这种情况经常发生在group by 和 order by作用在不同列上。
	- Using filesort 通常发生排序的列没有用上索引，导致全表检索。
3. SELECT * 对user_token进行分组，然后计算行数，没必要取全部数据影响性能，Where条件要不要都可以

所以该语句可以优化：
针对GROUP BY字段 user_token建立普通索引 然后在GROUP BY之前，应用该索引，SELECT * 改为 SELECT `user_token`
``` 
# ins_log_offer 在user_token字段上添加名为user_token_index的索引
ALTER TABLE ins_log_offer ADD INDEX user_token_index(user_token);

EXPLAIN SELECT
	COUNT(*)
FROM
	(
		SELECT
			`user_token`
		FROM
			`ins_log_offer`
		WHERE
			1
		GROUP BY
			`user_token`
	) `c`;
```
| id | select_type| table| type| possible_keys| key | key_len| ref|rows| Extra |
| :--- | :---| :---|:---|:---|:---|:---|:---|:---|:---|                   
|  1 | PRIMARY     | `<`derived2`>`| ALL  | NULL | NULL | NULL    | NULL | 8997 | NULL  |
|  2 | DERIVED     | ins_log_offer | index  | user_token_index   | user_token_index | 767    | NULL | 8997 | Using index |

在第一条SELECT COUNT(*) 依旧是ALL，所以必然还可以优化。

```
EXPLAIN SELECT
	COUNT(DISTINCT user_token)
FROM
	`ins_log_offer`
```
| id | select_type| table| type| possible_keys| key | key_len| ref|rows| Extra |
| :--- | :---| :---|:---|:---|:---|:---|:---|:---|:---|                   
|  1 | SIMPLE| ins_log_offer| range  | user_token_index| user_token_index | 767 | NULL | 8997 | Using index for group-by (scanning)|

在执行计划的Extra 信息中有信息显示“Using index for group-by”，实际上这就是告诉我们，MySQL Query Optimizer 通过使用松散索引扫描来实现了我们所需要的GROUP BY 操作。

**结论**: 三句结果集均为 3783条，说明结果也没影响，由于数据量不同，没法得到线上修改后的查询时间，但从EXPLAIN给出的信息可以得知对user_token创建索引，然后修改SQL应该能优化掉这句慢查询。在同样不加索引的情况下，第三句查询耗时0.02s，第一句耗时0.32s。

### 其他注意点
**这部分是关于索引和写SQL语句时应当注意的一些琐碎建议和注意点**

1. 当结果集只有一行数据时使用LIMIT 1
2. 避免SELECT *，始终指定你需要的列从表中读取越多的数据，查询会变得更慢。他增加了磁盘需要操作的时间，还是在数据库服务器与WEB服务器是独立分开的情况下。你将会经历非常漫长的网络延迟，仅仅是因为数据不必要的在服务器之间传输。
3. 使用连接（JOIN）来代替子查询(Sub-Queries)
连接（JOIN）.. 之所以更有效率一些，是因为MySQL不需要在内存中创建临时表来完成这个逻辑上的需要两个步骤的查询工作。
4. 使用ENUM、CHAR 而不是VARCHAR，使用合理的字段属性长度，空间换时间概念，定长可直接偏移量取值计算
5. 尽可能的使用NOT NULL
6. 固定长度的表会更快
7. 拆分大的DELETE 或INSERT 语句
8. 查询的列越小越快

**Where条件**
>在查询中，WHERE条件也是一个比较重要的因素，尽量少并且是合理的where条件是很重要的，尽量在多个条件的时候，把会提取尽量少数据量的条件放在前面，减少后一个where条件的查询时间。

有些where条件会导致索引无效：
- where子句的查询条件里有！=，MySQL将无法使用索引。
- where子句使用了Mysql函数的时候，索引将无效，比如：select * from tb where left(name, 4) = ‘xxx’。
- 使用LIKE进行搜索匹配的时候，这样索引是有效的：select * from tbl1 where name like ‘xxx%’，而like ‘%xxx%’ 时索引无效。
