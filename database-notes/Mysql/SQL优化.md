# 慢查询

默认情况下，MySQL数据库并不启动慢查询日志，需要用户手动开启。不需要调优时不建议开启该功能，会影响MySQL数据库的性能。

```
mysql> show variables like 'slow_query_log%';
+---------------------+-----------------------------------------------------+
| Variable_name       | Value                                               |
+---------------------+-----------------------------------------------------+
| slow_query_log      | OFF                                                 |
| slow_query_log_file | D:\MySQL\mysql-5.7.30\data\DESKTOP-LE1O5PF-slow.log |
+---------------------+-----------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)

#开启命令
set global slow_query_log = 1;
```

通过long_query_time参数设置超过多少秒的查询视为慢查询，数据库会将这些查询语句记录在慢查询日志中。

```
mysql> SHOW VARIABLES LIKE 'long_query_time';
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+
1 row in set, 1 warning (0.00 sec)

#修改值
set global long_query_time = 4;
```

可以通过`log_output`参数选择将日志存储到文件中或者MySQL的slow_log表中，默认是存储在文件中（存储在表中会耗费更多的系统资源）。

```
mysql> show variables like '%log_output%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_output    | FILE  |
+---------------+-------+
1 row in set, 1 warning (0.00 sec)
```

# explain

**explain各个字段代表的意思:**

- id ：select查询的序列号，包含一组数字，表示查询中执行select子句或操作表的顺序
- select_type ：查询类型 或者是 其他操作类型
- table ：正在访问哪个表
- partitions ：匹配的分区
- type ：访问的类型
- possible_keys ：显示可能应用在这张表中的索引，一个或多个，但不一定实际使用到
- key ：实际使用到的索引，如果为NULL，则没有使用索引
- key_len ：表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度
- ref ：显示索引的哪一列被使用了，如果可能的话，是一个常数，哪些列或常量被用于查找索引列上的值
- rows ：根据表统计信息及索引选用情况，大致估算出找到所需的记录所需读取的行数
- filtered ：查询的表行占表的百分比
- Extra ：包含不适合在其它列中显示但十分重要的额外信息

## id+table

```
EXPLAIN
SELECT *
FROM course AS C
WHERE C.`id` = (
  SELECT SC.`cid`
  FROM stu_course AS SC
  WHERE SC.`sid` =
  (
    SELECT
    S.`id`
    FROM student AS S
    WHERE  S.`name` = "安其拉"
  ) ORDER BY SC.`cid` LIMIT 1
)
```

```
    id  select_type  table   partitions  type    possible_keys  key      key_len  ref
------  -----------  ------  ----------  ------  -------------  -------  -------  ------
     1  PRIMARY      C       (NULL)      const   PRIMARY        PRIMARY  4        const
     2  SUBQUERY     SC      (NULL)      ref     PRIMARY        PRIMARY  4        const
     3  SUBQUERY     S       (NULL)      ref     name,name_age  name     63       const
```

表的读取顺序：按照id升序读取table，当id相同时，行号小的表先被读取。

## select_type

- SIMPLE 简单查询，不包括子查询和union查询

- PRIMARY 当存在子查询时，最外面的查询被标记为主查询
- SUBQUERY 子查询

## type

首先说一下这个字段，要记住以下10个状态，（从左往右，越靠左边的越优秀）

```
NULL > system > const > eq_ref > ref > ref_or_null > index_merge
> range > index > ALL
```

NULL：MySQL能够优化阶段分解查询语句，在执行阶段用不着再访问表或索引。例如：

```
EXPLAIN SELECT 5*7

    id  select_type  table   partitions  type    possible_keys  key
------  -----------  ------  ----------  ------  -------------  ------
     1  SIMPLE       (NULL)  (NULL)      (NULL)  (NULL)         (NULL)
```

SYSTEM：表只有一行记录（等于系统表），这是const类型的特列，平时不大会出现，可以忽略。

const：表示通过索引一次就找到了，const用于比较primary key或uique索引，因为只匹配一行数据，所以很快，如主键置于where列表中，MySQL就能将该查询转换为一个常量。简单来说，const是直接按主键或唯一键读取。

```
EXPLAIN SELECT * FROM student AS S WHERE id = 10

    id  select_type  table   partitions  type    possible_keys  key
------  -----------  ------  ----------  ------  -------------  -------
     1  SIMPLE       S       (NULL)      const   PRIMARY        PRIMARY
```

eq_ref：用于联表查询的情况，按联表的主键或唯一键联合查询。多表join时，对于来自前面表的每一行，在**当前表中只能找到一行**。这可能是除了system和const之外最好的类型。当主键或唯一非NULL索引的所有字段都被用作join联接时会使用此类型。

```
EXPLAIN SELECT * FROM student AS S JOIN stu_course AS SC ON  S.`id` = SC.`cid`

    id  select_type  table   partitions  type    possible_keys  key
------  -----------  ------  ----------  ------  -------------  -------
     1  SIMPLE       SC      (NULL)      index   (NULL)         PRIMARY
     1  SIMPLE       S       (NULL)      eq_ref  PRIMARY        PRIMARY
```

以上面查询为例，我们观察id和table会知道，先是从SC表中取出一行数据，然后再S表查找匹配的数据，我们观察，SC中取出cid和S表中的id比较，毫无疑问因为id是S表中的主键（不重复），所以只能出现一个id与cid的值相同。所以满足条件 S 表的 type为eq_ref。

ref：可以用于单表扫描或者连接。如果是连接的话，驱动表的一条记录能够在被驱动表中通过非唯一（主键）属性所在索引中匹配多行数据，或者是在单表查询的时候通过非唯一（主键）属性所在索引中查到一行数据。
