# ReplacingMergeTree

`MergeTree`支持主键，但主键主要用来缩小查询范围，且不具备唯一性约束，可以正常写入相同主键的数据。但在一些情况下，可能需要表中没有主键重复的数据。`ReplacingMergeTree`就是在`MergeTree`的基础上加入了去重的功能，但它仅会在合并分区时，去删除重复的数据，写入相同数据时并不会引发异常。

&nbsp;

## 具体使用

`ReplacingMergeTree` 引擎创建规范为：`ENGINE = ReplacingMergeTree([ver])`，其中`ver`为选填参数，它需要指定一个`UInt8/UInt16、Date或DateTime`类型的字段，它决定了数据去重时所用的算法，如果没有设置该参数，合并时保留分组内的最后一条数据；如果指定了该参数，则保留`ver`字段取值最大的那一行。

&nbsp;

### 不指定ver参数

```sql
-- 创建未指定ver参数ReplacintMergeTree引擎的表
CREATE TABLE replac_merge_test
(
    `id` String, 
    `code` String, 
    `create_time` DateTime
)
ENGINE = ReplacingMergeTree()
PARTITION BY toYYYYMM(create_time)
PRIMARY KEY id
ORDER BY (id, code)
```

`ReplacingMergeTree` 会根据 `ORDER BY` 所声明的表达式去重。

```SQL
-- 在上述表中插入数据
insert into replac_merge_test values ('A000', 'code1', now()),('A000', 'code1', '2020-07-28 21:30:00'), ('A001', 'code1', now()), ('A001', 'code2', '2020-07-28 21:30:00'), ('A0002', 'code2', now());

-- 查询当前数据
select * from replac_merge_test;
┌─id────┬─code──┬─────────create_time─┐
│ A000  │ code1 │ 2020-07-28 21:23:48 │
│ A000  │ code1 │ 2020-07-28 21:30:00 │
│ A0002 │ code2 │ 2020-07-28 21:23:48 │
│ A001  │ code1 │ 2020-07-28 21:23:48 │
│ A001  │ code2 │ 2020-07-28 21:30:00 │
└───────┴───────┴─────────────────────┘

-- 强制进行分区合并
optimize table replac_merge_test FINAL;

-- 再次查询数据
select * from replac_merge_test;
┌─id────┬─code──┬─────────create_time─┐
│ A000  │ code1 │ 2020-07-28 21:30:00 │
│ A0002 │ code2 │ 2020-07-28 21:23:48 │
│ A001  │ code1 │ 2020-07-28 21:23:48 │
│ A001  │ code2 │ 2020-07-28 21:30:00 │
└───────┴───────┴─────────────────────┘
```

由于创建表时没有设置`ver`参数，故保留分组内最后一条数据(`create_time`字段)。

```SQL
-- 再次使用insert插入一条数据
insert into replac_merge_test values ('A001', 'code1', '2020-07-28 21:30:00');

-- 查询表中数据
select * from replac_merge_test;
┌─id────┬─code──┬─────────create_time─┐
│ A000  │ code1 │ 2020-07-28 21:30:00 │
│ A0002 │ code2 │ 2020-07-28 21:23:48 │
│ A001  │ code1 │ 2020-07-28 21:23:48 │
│ A001  │ code2 │ 2020-07-28 21:30:00 │
└───────┴───────┴─────────────────────┘
┌─id───┬─code──┬─────────create_time─┐
│ A001 │ code1 │ 2020-07-28 21:30:00 │
└──────┴───────┴─────────────────────┘
```

可以看到，再次插入重复数据时，查询仍然会存在重复。在ClickHouse中，默认一条`insert`插入的数据为同一个数据分区，不同`insert`插入的数据为不同的分区，所以`ReplacingMergeTree`是以分区为单位进行去重的，也就是说只有在相同的数据分区内，重复数据才可以被删除掉。只有数据合并完成后，才可以使用引擎特性进行去重。

&nbsp;

### 指定ver参数

```SQL

-- 创建指定ver参数ReplacingMergeTree引擎的表
CREATE TABLE replac_merge_ver_test
(
    `id` String, 
    `code` String, 
    `create_time` DateTime
)
ENGINE = ReplacingMergeTree(create_time)
PARTITION BY toYYYYMM(create_time)
PRIMARY KEY id
ORDER BY (id, code)

-- 插入测试数据
insert into replac_merge_ver_test values('A000', 'code1', '2020-07-10 21:35:30'),('A000', 'code1', '2020-07-15 21:35:30'),('A000', 'code1', '2020-07-05 21:35:30'),('A000', 'code1', '2020-06-05 21:35:30');

-- 查询数据
select * from replac_merge_ver_test;
┌─id───┬─code──┬─────────create_time─┐
│ A000 │ code1 │ 2020-06-05 21:35:30 │
└──────┴───────┴─────────────────────┘
┌─id───┬─code──┬─────────create_time─┐
│ A000 │ code1 │ 2020-07-10 21:35:30 │
│ A000 │ code1 │ 2020-07-15 21:35:30 │
│ A000 │ code1 │ 2020-07-05 21:35:30 │
└──────┴───────┴─────────────────────┘

-- 强制进行分区合并
optimize table replac_merge_ver_test FINAL;

-- 查询数据
select * from replac_merge_ver_test;
┌─id───┬─code──┬─────────create_time─┐
│ A000 │ code1 │ 2020-07-15 21:35:30 │
└──────┴───────┴─────────────────────┘
┌─id───┬─code──┬─────────create_time─┐
│ A000 │ code1 │ 2020-06-05 21:35:30 │
└──────┴───────┴─────────────────────┘
```

由于上述创建表是以`create_time`的年月来进行分区的，可以看出不同的数据分区，`ReplacingMergeTree`并不会进行去重，并且在相同数据分区内，指定ver参数后，会保留同一组数据内`create_time`时间最大的那一行数据。

&nbsp;

## ReplacingMergeTree 引擎特点

* 使用 `ORDER BY` 排序键，作为判断数据是否重复的唯一键。
* 只有在合并分区时，才会触发数据的去重逻辑。
* 删除重复数据，是以数据分区为单位。同一个数据分区的重复数据才会被删除，不同数据分区的重复数据仍会保留。
* 在进行数据去重时，由于已经基于ORDER BY排序，所以可以找到相邻的重复数据。
* 数据去重策略为：
    * 若指定了ver参数，则会保留重复数据中，ver字段最大的那一行。
    * 若未指定ver参数，则会保留重复数据中最末的那一行数据。
