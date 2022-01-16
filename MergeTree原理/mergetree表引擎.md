# MergeTree表引擎

## 数据TTL

在MergeTree中，可以为某个列字段或整张表设置TTL。当时间达到时，会删除该列或表的数据。如果两项都满足，则谁时间先到，则先被触发。设置`TTL`的表或列的数据类型必须是 `DateTime` 或 `Date` 类型。

```SQL
sql> TTL time_col + INTERVAL 1 MONTH    // 数据存活时间为time_col时间的一个月后
sql> TTL time_col + INTERVAL 3 DAY      // 数据存活时间为time_col时间的三天之后
```

`INTERVAL` 可以设置为`SECOND`、`MINUTE`、`HOUR`、`DAY`、`WEEK`、`MONTH`、`QUARTER`和`YEAR`，用来表示数据多久后过期。

&nbsp;

### 列级别TTL

设置列级别TTL，需要在定义表字段的时候。主键字段不能被声明TTL。

```SQL
CREATE TABLE ttl_table_v1 (
    id String,
    create_time DateTime,
    code String TTL create_time + INTERVAL 10 SECOND,
    type UInt8 TTL create_time + INTERVAL 10 SECOND
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(create_time)
ORDER BY id
```

该实例为`code`和`type`字段设置了TTL，TTL过期时间是在数据被创建后(`create_time`)，10秒后(`+ INTERVAL 10 SECOND`)数据被清理。

&nbsp;

#### optimize强制执行清理

直接执行清理，无需等待TTL时间到了再清理。

```SQL
optimize TABLE ttl_table_v1 FINAL
```

&nbsp;

#### 更改列字段TTL

```SQL
; 更改为一天后数据被清理

ALTER TABLE ttl_table_v1 MODIFY COLUMN code String TTL create_time + INTERVAL 1 DAY
```

&nbsp;

### 表级TTL

表级别的TTL，TTL时间到了会清理整张表的数据。

```SQL
CREATE TABLE ttl_table_v2 (
    id String,
    create_time DateTime,
    code String TTL create_time + INTERVAL 1 MINUTE,
    type UInt8
) ENGINE = MergeTree
PARTITION BY toYYYYMM(create_time)
ORDER BY create_time
TTL create_time + INTERVAL 1 DAY
```

表级别的TTL也支持修改，修改如下：

```SQL
ALTER TABLE ttl_table_v2 MODIFY TTL create_time + INTERVAL 3 DAY
```
