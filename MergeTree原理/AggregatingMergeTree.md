# AggregatingMergeTree

AggregatingMergeTree能够在合并分区的时候，按照预先定义的条件聚合数据。同时，根据预先定义的聚合函数计算数据并通过二进制的格式存入表内。将同一分组下的多行数据聚合成一行，既减少了数据行，又降低了后续聚合查询的开销。

AggregatingMergeTree是SummingMergeTree的升级版，它们的许多设计思路是一致的，例如同时定义ORDER BY 与 PRIMARY KEY的原因和目的。

&nbsp;

## 建表

### 建表语句

```SQL
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
    ...
) ENGINE = AggregatingMergeTree()
[PARTITION BY expr]
[ORDER BY expr]
[SAMPLE BY expr]
[SETTINGS name=value, ...]
```

&nbsp;

### 建表实例

```SQL
CREATE TABLE emp_aggregatingmergeTree
(
    emp_id     UInt16 COMMENT '员工id',
    name       String COMMENT '员工姓名',
    work_place String COMMENT '工作地点',
    age        UInt8 COMMENT '员工年龄',
    depart     String COMMENT '部门',
    salary     AggregateFunction(sum, Decimal32(2)) COMMENT '工资'
) ENGINE = AggregatingMergeTree() ORDER BY (emp_id, name) PRIMARY KEY emp_id PARTITION BY work_place;
  
  
ORDER BY (emp_id,name) -- 注意排序key是两个字段
PRIMARY KEY emp_id     -- 主键是一个字段
```

对于`AggregateFunction`类型的列字段，在进行数据的写入和查询时与其他的表引擎有很大的区别，在写入数据时，需要调用`-State`函数；而在查询数据时，需要调用相应的 `-Merge` 函数。对于上面的建表语句而言，需要使用`sumState`函数进行数据插入。

```SQL
-- 插入数据，
-- 注意：需要使用INSERT…SELECT语句进行数据插入
INSERT INTO TABLE emp_aggregatingmergeTree SELECT 1,'tom','上海',25,'信息部',sumState(toDecimal32(10000,2));
INSERT INTO TABLE emp_aggregatingmergeTree SELECT 1,'tom','上海',25,'信息部',sumState(toDecimal32(20000,2));

-- 查询数据
SELECT emp_id,name,sumMerge(salary) FROM emp_aggregatingmergeTree GROUP BY emp_id,name;

-- 结果输出
┌─emp_id─┬─name─┬─sumMerge(salary)─┐
│      1 │ tom  │         30000.00 │
└────────┴──────┴──────────────────┘
```

&nbsp;

## 物化视图

AggregatingMergeTree更为常见的应用方式是结合物化视图使用，将它作为物化视图的表引擎。这里的物化视图是作为其他数据表上层的一种查询视图。

通常会使用MergeTree作为底表，用于存储全量的明细数据，并以此对外提供实时查询。物化视图使用AggregatingMergeTree表引擎，用于特定场景的数据查询，相比MergeTree，它拥有更高的性能。

```SQL
-- 创建一个MereTree引擎的明细表
-- 用于存储全量的明细数据
-- 对外提供实时查询
CREATE TABLE emp_mergetree_base
(
    emp_id     UInt16 COMMENT '员工id',
    name       String COMMENT '员工姓名',
    work_place String COMMENT '工作地点',
    age        UInt8 COMMENT '员工年龄',
    depart     String COMMENT '部门',
    salary     Decimal32(2) COMMENT '工资'
) ENGINE = MergeTree() ORDER BY (emp_id, name) PARTITION BY work_place;
  
-- 创建一张物化视图
-- 使用AggregatingMergeTree表引擎
CREATE MATERIALIZED VIEW view_emp_agg ENGINE = AggregatingMergeTree() PARTITION BY emp_id ORDER BY (emp_id, name) AS
SELECT emp_id, name, sumState(salary) AS salary
FROM emp_mergetree_base
GROUP BY emp_id, name;

-- 向基础明细表emp_mergetree_base插入数据
INSERT INTO emp_mergetree_base VALUES (1,'tom','上海',25,'技术部',20000),(1,'tom','上海',26,'人事部',10000);

-- 查询物化视图
SELECT emp_id,name,sumMerge(salary) FROM view_emp_agg GROUP BY emp_id,name;
-- 结果
┌─emp_id─┬─name─┬─sumMerge(salary)─┐
│      1 │ tom  │         50000.00 │
└────────┴──────┴──────────────────┘
```
