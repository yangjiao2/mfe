## MySQL 常见的两种存储引擎：MyISAM 与 InnoDB 的理解

关于二者的对比与总结:

count 运算上的区别：因为 MyISAM 缓存有表 meta-data（行数等），因此在做 COUNT(\*)时对于一个结构很好的查询是不需要消耗多少资源的。而对于 InnoDB 来说，则没有这种缓存。
是否支持事务和崩溃后的安全恢复： MyISAM 强调的是性能，每次查询具有原子性,其执行数度比 InnoDB 类型更快，但是不提供事务支持。但是 InnoDB 提供事务支持事务，外部键等高级数据库功能。 具有事务(commit)、回滚(rollback)和崩溃修复能力(crash recovery capabilities)的事务安全(transaction-safe (ACID compliant))型表。
是否支持外键： MyISAM 不支持，而 InnoDB 支持。

MyISAM 更适合读密集的表，而 InnoDB 更适合写密集的的表。 在数据库做主从分离的情况下，经常选择 MyISAM 作为主库的存储引擎。
一般来说，如果需要事务支持，并且有较高的并发读取频率(MyISAM 的表锁的粒度太大，所以当该表写并发量较高时，要等待的查询就会很多了)，InnoDB 是不错的选择。如果你的数据量很大（MyISAM 支持压缩特性可以减少磁盘的空间占用），而且不需要支持事务时，MyISAM 是最好的选择。

## Mysql 索引使用的数据结构主要有 BTree 索引 和 哈希索引

对于哈希索引来说，底层的数据结构就是哈希表，因此在绝大多数需求为单条记录查询的时候，可以选择哈希索引，查询性能最快；其余大部分场景，建议选择 BTree 索引。
Mysql 的 BTree 索引使用的是 B 数中的 B+Tree，但对于主要的两种存储引擎的实现方式是不同的。

MyISAM:

- B+Tree 叶节点的 data 域存放的是数据记录的地址。在索引检索的时候，首先按照 B+Tree 搜索算法搜索索引，如果指定的 Key 存在，则取出其 data 域的值，然后以 data 域的值为地址读取相应的数据记录。这被称为“非聚簇索引”。

InnoDB:

- 其数据文件本身就是索引文件。相比 MyISAM，索引文件和数据文件是分离的，其表数据文件本身就是按 B+Tree 组织的一个索引结构，树的叶节点 data 域保存了完整的数据记录。这个索引的 key 是数据表的主键，因此 InnoDB 表数据文件本身就是主索引。这被称为“聚簇索引（或聚集索引）”。而其余的索引都作为辅助索引（非聚集索引），辅助索引的 data 域存储相应记录主键的值而不是地址，这也是和 MyISAM 不同的地方。在根据主索引搜索时，直接找到 key 所在的节点即可取出数据；在根据辅助索引查找时，则需要先取出主键的值，在走一遍主索引。 因此，在设计表的时候，不建议使用过长的字段作为主键，也不建议使用非单调的字段作为主键，这样会造成主索引频繁分裂

## 常见优化手段

1. 读/写分离： 经典的数据库拆分方案，主库负责写，从库负责读；

2. 缓存： 使用 MySQL 的缓存，另外对重量级、更新少的数据可以考虑使用应用级别的缓存；

3. 垂直分区：
   根据数据库里面数据表的相关性进行拆分。 例如，用户表中既有用户的登录信息又有用户的基本信息，可以将用户表拆分成两个单独的表，甚至放到单独的库做分库。
   简单来说垂直拆分是指数据表列的拆分，把一张列比较多的表拆分为多张表。 如下图所示，这样来说大家应该就更容易理解了。

4. 垂直拆分的优点： 可以使得行数据变小，在查询时减少读取的 Block 数，减少 I/O 次数。此外，垂直分区可以简化表的结构，易于维护。
   垂直拆分的缺点： 主键会出现冗余，需要管理冗余列，并会引起 Join 操作，可以通过在应用层进行 Join 来解决。此外，垂直分区会让事务变得更加复杂；

5. 水平分区：
   保持数据表结构不变，通过某种策略存储数据分片。这样每一片数据分散到不同的表或者库中，达到了分布式的目的。 水平拆分可以支撑非常大的数据量。
   水平拆分是指数据表行的拆分，表的行数超过 200 万行时，就会变慢，这时可以把一张的表的数据拆成多张表来存放。

6. 限定数据的范围： 务必禁止不带任何限制数据范围条件的查询语句。比如：我们当用户在查询订单历史的时候，我们可以控制在一个月的范围内。；

其他:
通常会在 WHERE、JOIN ON 和 ORDER BY 使用到字段上加上索引。
避免查询时判断 NULL，否则可能会导致全表扫描。
避免使用 OR 来连接查询条件，否则可能导致全表扫描，可以改用 UNION 或 UNION ALL。
避免 LIKE 查询，否则可能导致全表扫描。
不使用 SELECT \*，只查询必须的字段，避免加载无用数据。
能用 UNION ALL 的时候就不用 UNION，UNION 过滤重复数据要耗费更多的 cpu 资源。

## b+tree 的优点：

B+tree 的磁盘读写代价更低
内部结点并没有指向关键字具体信息的指针。因此其内部结点相对 B 树更小。
如果把所有同一内部结点的关键字存放在同一盘块中，那么盘块所能容纳的关键字数量也越多。一次性读入内存中的需要查找的关键字也就越多。相对来说 IO 读写次数也就降低了。
举个例子，假设磁盘中的一个盘块容纳 16bytes，而一个关键字 2bytes，一个关键字具体信息指针 2bytes。一棵 9 阶 B-tree(一个结点最多 8 个关键字)的内部结点需要 2 个盘快。

B+-tree 的查询效率更加稳定
由于非终结点并不是最终指向文件内容的结点，而只是叶子结点中关键字的索引。
所以任何关键字的查找必须走一条从根结点到叶子结点的路。所有关键字查询的路径长度相同，导致每一个数据的查询效率相当。

## 数据结构

MyISAM 索引: “非聚集”
MyISAM 的索引文件仅仅保存数据记录的地址
主索引和辅助索引（Secondary key）在结构上没有任何区别

InnoDB 索引: "聚集”, "回表"
表数据文件本身就是按 B+Tree 组织的一个索引结构，这棵树的叶节点 data 域保存了完整的数据记录
表数据文件本身就是主索引,主键必须存在 / 或者生成一个隐含字段作为主键
InnoDB 的辅助索引 data 域存储相应记录主键的值而不是地址

## 聚簇索引与非聚簇索引的区别？

聚簇索引：聚簇索引的顺序就是数据的物理存储顺序，并且索引与数据放在一块，通过索引可以直接获取数据，一个数据表中仅有一个聚簇索引。

非聚簇索引：索引顺序与数据物理排列顺序无关，索引文件与数据是分开存放。

## 回表

回表是对 Innodb 存储引擎而言的，在 InnoDB 存储引擎中，主键索引的叶子节点存储的记录的数据，而普通索引的叶子节点存储的主键索引的地点。
当我们通过主键查询时，只需要搜索主键索引的搜索树，直接可以得到记录的数据。
当我们通过普通索引进行查询时，通过搜索普通索引的搜索树得到主键的地址之后，还要再使用该主键对主键搜索树进行搜索，这个过程称为回表。

## MVCC

RR 解决脏读、不可重复读、幻读等问题，使用的是 MVCC：MVCC 全称 Multi-Version Concurrency Control，即多版本的并发控制协议。下面的例子很好的体现了 MVCC 的特点：在同一时刻，不同的事务读取到的数据可能是不同的(即多版本)——在 T5 时刻，事务 A 和事务 C 可以读取到不同版本的数据。

其中数据的隐藏列包括了该行数据的版本号、删除时间、指向 undo log 的指针等等；当读取数据时，MySQL 可以通过隐藏列判断是否需要回滚并找到回滚需要的 undo log，从而实现 MVCC；其中数据的隐藏列包括了该行数据的版本号、删除时间、指向 undo log 的指针等等；当读取数据时，MySQL 可以通过隐藏列判断是否需要回滚并找到回滚需要的 undo log，从而实现 MVCC；
