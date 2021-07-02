# SQL性能调优

1. **优化器**：oracle数据库内置的一个核心子系统。优化器的目的是按照一定的判断原则来得到目标sql的最佳执行计划。优化器可分为RBO（基于原则的优化器）和CBO（基于成本的优化器）

* **RBO**：优化器在分析sql语句时，所遵循的是oracle内部预定的一些规则，比如，当一个where子句中的一列有索引时去走索引

* **CBO**：看语句的代价（CPU和内存）。依据是表及索引的统计信息。

2. **执行计划**：一条查询语句在oracle中的执行过程或访问路径的描述

   **统计信息**描述数据库中表、索引的大小、规模，数据分布状况的一类信息。

## PL/SQL执行计划

**常用字段解释**

* **基数（Rows）**oracle估计当前操作返回结果的行数。
* **字节（Bytes）**执行该步骤后返回的字节数
* **耗费（Cost）**理论上越小越好
* **时间（Time）**oracle估计当前操作所需的时间

**执行计划的顺序**

* 同一级如果某个动作没有子ID就最先执行
* 同一级的动作执行时遵循最上最右先执行的原则

**ROWID的概念**

* rowid是一个伪列，由系统自动添加，对每个表都有一个rowid的伪列，但是表中并不物理存储rowid的值。

**oracle表访问的几种常见方式**

* **TABLE ACCESS FULL（全表扫描）**

  读取表中所有行，并检查每一行是否满足sql语句中的where限制条件

* **TABLE ACCESS BY ROWID（通过ROWID的表存取）**

  行的rowid指出了该行所在的数据文件、数据块以及行在该块中的位置，所以通过rowid可以快速定位到目标数据上，这也是oracle中存取单行数据最快的方法。

* **TABLE ACCESS BY INDEX SCAN（索引扫描）**

  在索引块中，既存储每个索引的键值，也存储具有该键值的行的rowid

  所以索引扫描分两步：1、扫描索引得到对应的rowid  2、通过rowid定位到具体的行读取数据

## 关于索引扫描

* **INDEX UNIQUE SCAN（索引唯一扫描）**

  * 针对唯一性索引的扫描，每次仅返回一条记录

  * 针对某字段含有UNIQUE、PRIMARY KEY约束时

* **INDEX RANGE SCAN（索引范围扫描）**

  一个索引存取多行数据

* **INDEX FULL SCAN（索引全扫描）**

  查询的目标列上建有索引，则会直接走索引全扫描来获取数据，并且获取到的数据有序排列

* **INDEX FAST FULL SCAN（索引快速扫描）**

  与索引全扫描类似，当所访问到的列包含在索引范围内时，但与索引全扫描不同的是，索引快速扫描是多块读。

* **INDEX SKIP SCAN（索引跳跃扫描）**

  对复合索引的列进行过滤时发生

## 表连接方式及相关概念

#### 相关概念

join关键字用于连接两张表，连接操作一般是串行的。

表（row source）之间的连接顺序对查询效率有很大的影响，对首先存取的表（驱动表）先应用某些限制条件（where过滤条件）以得到一个较小的row source，可以使得连接效率提高。

**驱动表（driving table）**

表连接时首先存取的表，又称为外层表（outer table）。这个概念用于nested loops（嵌套循环）与hash join（哈希连接）中；

如果驱动表返回较多的行数据，则对所有后续操作有负面影响，故一般选择小表作为驱动表。

**匹配表（probed table）**

又称为内层表（inner table），从驱动表获取一行具体数据后，会到该表中寻找符合连接条件的行。故该表一般为大表。

#### 表连接的几种方式

* **sort merge join（排序-合并连接）**

  假设有查询：

  ```sql
  select a.name,b.name from table_A a join table_B b on (a.id=b.id)
  ```

  内部连接过程：

  1.  生成row source1需要的数据，按照连接操作相关列（a.id）对这些数据进行排序。
  2.  生成row source2需要的数据，按照与1中对应的连接操作关联列（b.id）对数据进行排序。
  3.  两边已排序的行放在一起执行合并操作（对两边的数据集进行扫描并判断是否连接。

  延伸：

  * 若连接操作关联列（a.id，b.id）是已经排好序的，连接速度便可大大提高，因为排序是很浪费时间和资源的操作。
  * 排序-合并连接的表无驱动顺序，谁在前面都可以。
  * 排序-合并连接适用的连接条件有：< 、<=、 =、 =>、 >，不适用的连接条件：<、>、like 

* **nested loops（嵌套循环）**

  假设有查询：

  ```sql
  select a.name,b.name from table_A a join table_B b on (a.id=b.id)
  ```

  内部连接过程：

  1. 取出row-source1的row1（第一行数据），遍历row source2的所有行并检查是否有匹配的，取出匹配的行放入结果集中。

  2. 取出row-source1的row2（第二行数据），遍历row source2的所有行并检查是否有匹配的，取出匹配的行放入结果集中。

     ……

     若row source1（即驱动表）中返回了N行数据，则row source2也相应的会被全表遍历N次。

     因为row source1的每一行都会去匹配row source2的所有行，所以row source1返回的行数尽可能少，并且能高效访问row source2（如建立适当索引）时，效率较高。

* **hash join（哈希连接）**

  哈希连接只适用于等值连接。

  假设有查询：

  ```sql
  select a.name,b.name from table_A a join table_B b on (a.id=b.id)
  ```

  内部连接过程：

  1. 取出row source1（驱动表，在hash join中又被称为build table）的数据集，然后将其构建成内存中的一个hash table。创建hash位图（bitmap）
  2. 取出row source2（匹配表）的数据集，对其中的每一条数据的连接操作关联列使用相同的hash函数并找到对应的1.里的数据在hash table中的位置，在该位置上检查能否找到匹配的数据。

* **cartesian product（笛卡尔积）**

