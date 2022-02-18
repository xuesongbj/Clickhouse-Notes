# MergeTree之间的关系

如下7种MergeTree存储引擎公用一个主体，在触发Merge动作时，它们调用各自独有的合并逻辑。

![MergeTree关系](./imgs/mergetree_1.png)