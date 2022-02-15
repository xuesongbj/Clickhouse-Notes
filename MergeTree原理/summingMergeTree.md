# SummingMergeTree

## SummingMergeTree 使用场景

针对只需要查询数据的汇总结果，不关心明细数据，并且数据的汇总条件是预先明确的(GROUP BY 条件明确，且不会随意改变)。

虽然可以通过明细数据，通过`group by`实现以上汇总，但这种方案可能存在两个问题:

* 额外的存储开销：用户不会查询任何明细数据，只关心汇总结果，所以不应该一直保存所有的明细数据。

* 额外的查询开销：用户只关心汇总结果，虽然MergeTree性能强大，但是每次查询都进行实时聚合计算也是一种性能消耗。

如果使用SummingMergeTree能够在合并分区的时候按照预先定义的条件聚合汇总数据，将同一分组下的多行数据汇总合并成一行，这样既可以减少数据行，又降低了后续汇总查询的开销。

&nbsp;

## SummingMergeTree使用

&nbsp;

### 建表

```SQL
create table temp.summing_table_test1 on cluster ck_cluster
(
       v1 Int32,
       v2 Int32,
       name String,
       total_date DateTime
) ENGINE = SummingMergeTree((v1,v2))
order by (name)
partition by toDate(total_date)
SETTINGS index_granularity = 8192;
```

&nbsp;

### 写入数据

```SQL
insert into temp.summing_table_test1
values (1,2,'a',now()),(2,2,'a',now()-1*60*60),(3,4,'b',now());
```

&nbsp;

### 查询数据

```SQL
SELECT * FROM temp.summing_table_test1;

Query id: 2da82c96-2a90-496a-83fe-8a6528ba336c
┌─v1─┬─v2─┬─name─┬──────────total_date─┐
│  3 │  4 │ a    │ 2021-10-13 11:41:12 │
│  3 │  4 │ b    │ 2021-10-13 11:41:12 │
└────┴────┴──────┴─────────────────────┘
```
