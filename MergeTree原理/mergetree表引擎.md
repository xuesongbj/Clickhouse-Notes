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

表级别TTL目前没有取消的方法。

&nbsp;

## TTL的运行原理

如果一张 `MergeTree` 表被设置了 `TTL` 表达式，**那么在写入数据时，会以数据分区为单位，在每个分区目录内生成一个名为`ttl.txt`的文件。**

```SQL
; 设置列级别TTL
code String TTL create_time + INTERVAL 1 MINUTE

; 同时设置表级别的TTL
TTL create_time + INTERVAL 1 DAY

; 在写入数据之后，它的每个分区目录内都会生成 `ttl.txt` 文件

# pwd
/chbase/data/data/default/ttl_table_v2/201905_1_1_0/ttl.txt
```

查看 `ttl.txt` 文件内容:

```json
# cat ./ttl.txt
ttl format version: 1
{"columns": [{"name": "code", "min": 1557478860, "max": 1557651660}], "table": {"min": 1557565200, "max": 1557738000}}
```

`ttl.txt` 文件通过一个`json`字符串进行控制TTL。

* **columns**：保存列级别TTL
* **table**：保存表级别TTL
* **min和max**：保存在当前数据分区内，TTL指定日期字段的最小值、最大值分别与`INTERVAL` 表达式计算后的时间戳

&nbsp;

### 处理逻辑

* MergeTree以分区目录为单位，通过 `ttl.txt` 文件记录过期时间，并将其作为后续的判断依据
* 每当写入一批数据时，都会基于 `INTERVAL` 表达式的计算结果为这个分区生成 `ttl.txt` 文件
* 只有在MergeTree合并分区时，才会触发删除TTL过期数据的逻辑
* 在选择删除的分区时，会使用贪婪算法，尽可能找到最早过期的，同时年纪又是最老的分区(合并次数最多，MaxBlockNum更大的)
* 如果一个分区内某一列数据因为TTL到期全部被删除了，那么在合并之后生成的新分区目录中，将不会包含这列字段的数据文件(`.bin` 和 `.mrk`)。

&nbsp;

#### 处理逻辑细节

* TTL默认的合并频率由MergeTree的 `merge_with_ttl_timeout` 参数控制，默认86400秒(1天)。它维护的是一个专有的TTL任务队列。和MergeTree的常规合并任务不是同时触发，这个值设置过小，可能会带来性能损耗
* 除了被动触发TTL合并外，也可以使用 `optimize` 命令强制合并。

```SQL
; 触发一个分区合并
optimize TABLE table_name

; 触发所有分区合并
optimize TABLE table_name FINAL
```

* Clickhouse目前不支持删除TTL声明的方法，但提供了控制全局TTL合并任务的启/停方法

```SQL
SYSTEM STOP/START TTL MERGES
```

&nbsp;

## 多路径存储

Clickhouse 19.15 版本之前，MergeTree表引擎只支持单路径存储，所有数据都被写入到 `config.xml` 配置中 `path` 指定的路径下，即时服务器挂载了多块磁盘，也无法有效利用这些存储空间。为解决该问题，从 `19.15` 版本开始，MergeTree表引擎实现了自定义存储策略的功能，支持以数据分区为最小移动单位，将分区目录写入多块磁盘目录。

目前大致支持三大类存储策略。

* 默认策略：MergeTree表默认策略，根据该策略会将数据自动保存到 `config.xml` 配置中 `path` 指定的路径下。

* `JBOD`策略：`Just a Bunch of Disks` 策略，适合服务器挂载了多块磁盘，但没有做RAID的场景。它是一种轮询策略，每执行一次 `INSERT` 或者 `MERGE`，所产生的新分区会轮询写入各个磁盘，可以降低单块磁盘的负载，在一定条件下能够增加数据并行读写的性能。如果单块磁盘发生故障，则会丢掉该块磁盘数据(数据的可靠性需要利用副本机制保障)。

* `HOT/COLD` 策略：这从策略适合服务器挂载了不同类型磁盘的场景。可以将磁盘分为 `HOT` 与 `COLD` 两类分区。**`HOT`分区使用SSD这类高性能磁盘，注重存取新能；`COLD` 区域则使用HDD这类大容量磁盘。** 数据在写入 `MergeTree` 之初，首先会在 `HOT` 区域创建分区目录用于保存数据，当分区数据大小积累到阈值时，数据会自行移动到 `COLD` 区域。在每个区域的内容，也支持定义多个磁盘，所以在单个区域的写入过程中，也能应用 `JBOD` 策略。

&nbsp;

### JBOD策略

JBOD策略，每当生成一个新数据分区的时候，分区目录会根据 `volume` 中定义的 `disk` 顺序一次轮询并写入各个磁盘。

```xml
<!-- 定义多路径存储策略 -->
<storage_configuration>
    <!-- 定义磁盘 -->
    <disks>
        <disk_01>   <!-- 自定义磁盘名称，全局唯一 -->
            <path>/data/chbase/data_01/</path>  <!-- 磁盘路径 -->
        </disk_01>
        <disk_02>
            <path>/data/chbase/data_02/</path>
        </disk_02>
        <disk_03>
            <path>/data/chbase/data_03/</path>
        </disk_03>
    </disks>

    <!-- 定义策略 -->
    <policies>
        <jbod_policies> <!-- 自定义策略全局唯一 -->
            <!-- 定义volume -->
            <volumes>
                <jbod_volume>  <!-- 自定义volume名称，全局唯一 -->
                   <disk>disk_01</disk> <!-- 指定该volume 下使用的磁盘，磁盘名称和上面定义的对应 -->
                   <disk>disk_02</disk>
                </jbod_volume>
            </volumes>
        </jbod_policies>
    </policies>
</storage_configuration>
```

* `keep_free_space_bytes`：用于指定指定磁盘的预留空间。
* `max_data_part_size_bytes`：用于在这个卷的单个disk磁盘中，一个数据分区的最大磁盘存储阈值，若当前分区的数据大小超过阈值，则之后的分区会写入下一个disk磁盘。
* `move_factor`：若当前卷的可用空间大小小于factor因子，并且定义多个卷，则数据会自动移动到下一个卷。
