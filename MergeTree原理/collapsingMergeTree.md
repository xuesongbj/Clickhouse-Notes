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

&nbsp;

### 删除数据

```SQL
-- 修改前的源数据，它需要被删除
INSERT INTO emp_collapsingmergetree VALUES (1,'tom','上海',25,'技术部',20000,1);

-- 镜像数据，ORDER BY字段与源数据相同，sign去反为-1，它会和源数据折叠
INSERT INTO emp_collapsingmergetree VALUES (1,'tom','上海',25,'技术部',20000,-1);
```

&nbsp;

### 写入顺序问题

`CollapsingMergeTree`对于写入数据的顺序有着严格要求，否则导致无法正常折叠。

```SQL
-- 建表
CREATE TABLE emp_collapsingmergetree_order
(
    emp_id     UInt16 COMMENT '员工id',
    name       String COMMENT '员工姓名',
    work_place String COMMENT '工作地点',
    age        UInt8 COMMENT '员工年龄',
    depart     String COMMENT '部门',
    salary     Decimal32(2) COMMENT '工资',
    sign       Int8
) ENGINE = CollapsingMergeTree(sign) ORDER BY (emp_id, name) PARTITION BY work_place;

-- 先插入需要被删除的数据，即sign=-1的数据
INSERT INTO emp_collapsingmergetree_order VALUES (1,'tom','上海',25,'技术部',20000,-1);

-- 再插入sign=1的数据
INSERT INTO emp_collapsingmergetree_order VALUES (1,'tom','上海',25,'技术部',20000,1);

-- 查询表
SELECT * FROM emp_collapsingmergetree_order;
┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┐
│      1 │ tom  │ 上海       │  25 │ 技术部 │ 20000.00 │   -1 │
└────────┴──────┴────────────┴─────┴────────┴──────────┴──────┘
┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┐
│      1 │ tom  │ 上海       │  25 │ 技术部 │ 20000.00 │    1 │
└────────┴──────┴────────────┴─────┴────────┴──────────┴──────

-- 执行合并操作
optimize table emp_collapsingmergetree_order;

-- 再次查询表
-- 旧数据依然存在
SELECT * FROM emp_collapsingmergetree_order;
┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┐
│      1 │ tom  │ 上海        │  25 │ 技术部 │ 20000.00 │   -1 │
│      1 │ tom  │ 上海        │  25 │ 技术部 │ 20000.00 │    1 │
└────────┴──────┴────────────-┴─────┴────────┴──────────┴──────
```

如果数据的写入顺序是**单线程执行**的，则能够较好地控制**写入顺序**；如果需要处理的数据量很大，数据的写入顺序通常是多线程执行的，那么此时就不能保障数据的写入顺序了。在这种情况下，**CollapsingMergeTree**的工作机制就会出现问题。但可以通过**VersionedCollapsingMergeTree**表引擎得到解决。

&nbsp;

## VersionedCollapsingMergeTree

VersionedCollapsingMergeTree表引擎的作用与CollapsingMergeTree相同，他们的不同在于VersionedCollapsingMergeTree对数据的写入顺序没有要求，在同一个分区内，任意顺序的数据都能够完成折叠操作。该引擎是在建表的时候除了指定`sign`标记字段外，还需要指定一个Uint8类型的`ver`版本号字段。

&nbsp;

### 建表语法

```SQL
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
    ...
) ENGINE = VersionedCollapsingMergeTree(sign, version)
[PARTITION BY expr]
[ORDER BY expr]
[SAMPLE BY expr]
[SETTINGS name=value, ...]
```

&nbsp;

### 使用方式

```SQL
CREATE TABLE emp_versioned
(
    emp_id     UInt16 COMMENT '员工id',
    name       String COMMENT '员工姓名',
    work_place String COMMENT '工作地点',
    age        UInt8 COMMENT '员工年龄',
    depart     String COMMENT '部门',
    salary     Decimal32(2) COMMENT '工资',
    sign       Int8,
    version    Int8
) ENGINE = VersionedCollapsingMergeTree(sign, version) ORDER BY (emp_id, name) PARTITION BY work_place;
  
-- 先插入需要被删除的数据，即sign=-1的数据
INSERT INTO emp_versioned VALUES (1,'tom','上海',25,'技术部',20000,-1,1);

-- 再插入sign=1的数据
INSERT INTO emp_versioned VALUES (1,'tom','上海',25,'技术部',20000,1,1);

-- 在插入一个新版本数据
INSERT INTO emp_versioned VALUES (1,'tom','上海',25,'技术部',30000,1,2);

-- 先不执行合并，查看表数据
select * from emp_versioned;
┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┬─version─┐
│      1 │ tom  │ 上海       │  25 │ 技术部 │ 20000.00 │    1 │       1 │
└────────┴──────┴────────────┴─────┴────────┴──────────┴──────┴─────────┘
┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┬─version─┐
│      1 │ tom  │ 上海       │  25 │ 技术部 │ 20000.00 │   -1 │       1 │
└────────┴──────┴────────────┴─────┴────────┴──────────┴──────┴─────────┘
┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┬─version─┐
│      1 │ tom  │ 上海       │  25 │ 技术部 │ 30000.00 │    1 │       2 │
└────────┴──────┴────────────┴─────┴────────┴──────────┴──────┴─────────┘

-- 获取正确查询结果
SELECT emp_id,name,sum(salary * sign) FROM emp_versioned GROUP BY emp_id,name HAVING sum(sign) > 0;
┌─emp_id─┬─name─┬─sum(multiply(salary, sign))─┐
│      1 │ tom  │                    30000.00 │
└────────┴──────┴─────────────────────────────┘

-- 手动合并
optimize table emp_versioned;

-- 再次查询
select * from emp_versioned;
┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┬─version─┐
│      1 │ tom  │ 上海       │  25 │ 技术部 │ 30000.00 │    1 │       2 │
└────────┴──────┴────────────┴─────┴────────┴──────────┴──────
```

使用`VersionedCollapsingMergeTree`会自动将`Vesion`作为排序条件并添加到`ORDER BY`的末端，最终的排序字段为`ORDER BY emp_id,name, version desc`。
