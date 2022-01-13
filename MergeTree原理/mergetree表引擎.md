# MergeTree表引擎

## 数据TTL

在MergeTree中，可以为某个列字段或整张表设置TTL。当时间达到时，会删除该列或表的数据。如果两项都满足，则谁时间先到，则先被触发。设置`TTL`的表或列的数据类型必须是 `DateTime` 或 `Date` 类型。

```SQL
sql> TTL time_col + INTERVAL 1 MONTH    // 数据存活时间为time_col时间的一个月后
sql> TTL time_col + INTERVAL 3 DAY      // 数据存活时间为time_col时间的三天之后
```

`INTERVAL` 可以设置为`SECOND`、`MINUTE`、`HOUR`、`DAY`、`WEEK`、`MONTH`、`QUARTER`和`YEAR`，用来表示数据多久后过期。
