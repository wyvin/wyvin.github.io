# 优化GroupBy语句


> 数据库有一张专门记录设备数据变化的表，目前已经有150W+条记录了，每次使用`group by`语句时用时都比较长，甚至出现`i/o timeout`执行超时，需要优化一下

表结构大概如下
```
+--------------+---------------------+------+-----+---------------------+----------------+
| Field        | Type                | Null | Key | Default             | Extra          |
+--------------+---------------------+------+-----+---------------------+----------------+
| id           | bigint(20) unsigned | NO   | PRI | NULL                | auto_increment |
| device_id    | varchar(64)         | NO   | MUL |                     |                |
| reading_type | int(11)             | NO   | MUL | 1                   |                |
| reading      | varchar(64)         | NO   |     | 0                   |                |
| action_time  | varchar(64)         | NO   | MUL | 1970-01-01 00:00:00 |                |
+--------------+---------------------+------+-----+---------------------+----------------+
```

**要求**：

* 需要找出某个device_id的每个reading_type的最新一条记录

## 优化前

原来的sql：
```sql
SELECT
	a.*
FROM
	device_amount_record a,
	( SELECT MAX( id ) AS id FROM device_amount_record WHERE device_id =? GROUP BY reading_type ) b
WHERE
	a.id = b.id
```

执行时间基本超过了`5s`，而数据库的超时配置为`5s`，所以经常报错。

使用`explain`分析下：
```
+----+-------------+----------------------+------------+--------+--------------------+---------+---------+-------+--------+----------+--------------------------------------------------------+
| id | select_type | table                | partitions | type   | possible_keys      | key     | key_len | ref   | rows   | filtered | Extra                                                  |
+----+-------------+----------------------+------------+--------+--------------------+---------+---------+-------+--------+----------+--------------------------------------------------------+
|  1 | PRIMARY     | <derived2>           | NULL       | ALL    | NULL               | NULL    | NULL    | NULL  | 474474 |   100.00 | Using where                                            |
|  1 | PRIMARY     | a                    | NULL       | eq_ref | PRIMARY            | PRIMARY | 8       | b.id  |      1 |   100.00 | NULL                                                   |
|  2 | DERIVED     | device_amount_record | NULL       | ref    | devID,reading_type | devID   | 258     | const | 474474 |   100.00 | Using index condition; Using temporary; Using filesort |
+----+-------------+----------------------+------------+--------+--------------------+---------+---------+-------+--------+----------+--------------------------------------------------------+
```

可以看到`Extra`列中的值有：`Using temporary`和`Using filesort`

**`Using temporary`**

表示使用了临时表，创建一个临时表比较耗时和耗内存。推测是语句中的b表或inner join时使用了临时表。

使用`show global status like '%tmp%';`命令查看发现临时表是建在硬盘上的，推测临时表超过了mysql设置的最大内存。。

**`Using filesort`**
表示没有使用索引的排序。

## 优化后

由于`id`使用了自增，所以可以通过查找最大的`id`来找到最新的数据。因为需要根据`action_time`做业务，这里还是根据`action_time`排序。

由于`group by`扫描的行数太多，而我们其实可以知道有哪些`reading_type`的，所以优化的思路就是提供`reading_type`过滤，减少扫描。

修改后的sql：
```sql
SELECT
	*
FROM
	(
        SELECT
            reading_type, reading, action_time
        FROM device_amount_record
        WHERE device_id = ?
            AND reading_type = ?
        ORDER BY action_time DESC LIMIT 1
    ) a
UNION
SELECT
	*
FROM
	(
        SELECT
            reading_type, reading, action_time
        FROM device_amount_record
        WHERE device_id = ?
            AND reading_type = ?
        ORDER BY action_time DESC LIMIT 1
    ) b
...
```

每一个`reading_type`写一条sql，然后用`union`连接起来。

单条的sql使用索引`device_id`,`reading_type`搜索，使用索引`action_time`排序，由于业务原`reading_type`的数量一般不多，`union`建立的临时表很小。

优化后执行时间只有`0.01s`~!


