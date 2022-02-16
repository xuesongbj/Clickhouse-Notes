# CollapsingMergeTree(折叠合并树)

Clickhouse存储按列顺序进行存储，对数据按行修改和删除的操作代价非常大。相比于直接修改源文件，通常会将修改和删除操作转换成新增操作，以增代删。

CollapsingMergeTree就是一种通过以增代删的思路，支持行级数据修改和删除的表引擎。它通过定义个`sign`标记位字段，记录数据行的状态。如果`sign`标记为1，则表示这是一行有效的数据；如果`sign`标记为-1，则表示这行数据需要被删除。当`CollapsingMergeTree`分区合并时，同一数据源分区内，`sign`标记为1和-1的一组数据会被删除。

&nbsp;

## 具体使用

### 创建CollapsingMergeTree引擎表

```SQL
CREATE TABLE emp_collapsingmergetree
(
    emp_id     UInt16 COMMENT '员工id',
    name       String COMMENT '员工姓名',
    work_place String COMMENT '工作地点',
    age        UInt8 COMMENT '员工年龄',
    depart     String COMMENT '部门',
    salary     Decimal32(2) COMMENT '工资',
    sign       Int8
) ENGINE = CollapsingMergeTree(sign) ORDER BY (emp_id, name) PARTITION BY work_place;
```

&nbsp;

### 使用方式

CollapsingMergeTree同样是以`ORDER BY`排序键作为判断数据唯一性的依据。

#### 修改数据

```SQL
-- 插入新增数据,sign=1表示正常数据
INSERT INTO emp_collapsingmergetree VALUES (1,'tom','上海',25,'技术部',20000,1);

-- 更新上述的数据
-- 首先插入一条与原来相同的数据(ORDER BY字段一致),并将sign置为-1
INSERT INTO emp_collapsingmergetree VALUES (1,'tom','上海',25,'技术部',20000,-1);

-- 再插入更新之后的数据
INSERT INTO emp_collapsingmergetree VALUES (1,'tom','上海',25,'技术部',30000,1);
```

修改完之后，并不会发上生效，而是需要执行分区合并时才会触发更新操作。

```SQL
-- 查看一下结果
select * from emp_collapsingmergetree ;
┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┐
│      1 │ tom  │ 上海       │  25 │ 技术部 │ 20000.00 │   -1 │
└────────┴──────┴────────────┴─────┴────────┴──────────┴──────┘
┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┐
│      1 │ tom  │ 上海       │  25 │ 技术部 │ 20000.00 │    1 │
└────────┴──────┴────────────┴─────┴────────┴──────────┴──────┘
┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┐
│      1 │ tom  │ 上海       │  25 │ 技术部 │ 30000.00 │    1 │
└────────┴──────┴────────────┴─────┴────────┴──────────┴──────┘

-- 执行分区合并操作
optimize table emp_collapsingmergetree;
-- 再次查询，sign=1与sign=-1的数据相互抵消了，即被删除
select * from emp_collapsingmergetree ;

┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┐
│      1 │ tom  │ 上海       │  25 │ 技术部 │ 30000.00 │    1 │
└────────┴──────┴────────────┴─────┴────────┴──────────┴──────┘
```
