
## Mysql采用B+树索引有什么原因

它是 B Tree 的变种，B Tree 能解决的问题，它都能解决。B Tree 解决的两大问题  是什么？（每个节点存储更多关键字；路数更多）

- 扫库、扫表能力更强（如果我们要对表进行全表扫描，只需要遍历叶子节点就可以 了，沿着叶子节点就可以拿到全部的数据，不需要遍历整棵 B+Tree 拿到所有的数据）
- B+Tree 的磁盘读写能力相对于 B Tree 来说更强（根节点和枝节点不保存数据区， 所以一个节点可以保存更多的关键字，一次磁盘加载的关键字更多，直接在叶子节点就可以拿到全部的数据，不需要在返回根节点枝节点）
- 排序能力更强（因为叶子节点上有下一个数据区的指针，数据形成了链表）
- 效率更加稳定（B+Tree 永远是在叶子节点拿到数据，所以 IO 次数是稳定的）
  问题：索引为什么不用红黑树？

  红黑树有一个特性确保没有一条路径会比其他路径长出俩倍。假设最长路径10，那最短是5。10次的IO对于操作磁盘来说是不能接受的。B+Tree 3层基本上就能存储千万级的数据。

## 更新一条记录过程

具体更新一条记录 `UPDATE t_user SET name = 'xiaolin' WHERE id = 1;` 的流程如下:

<pre class="vditor-reset" placeholder="" contenteditable="true" spellcheck="false"><p data-block="0"><br class="Apple-interchange-newline"/><img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/sql%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B/mysql%E6%9F%A5%E8%AF%A2%E6%B5%81%E7%A8%8B.png" alt="查询语句执行流程"/></p></pre>

1. 客户端先通过连接器建立连接，连接器自会判断用户身份；
2. 因为这是一条 update 语句，所以不需要经过查询缓存，但是表上有更新语句，是会把整个表的查询缓存清空的，所以说查询缓存很鸡肋，在 MySQL 8.0 就被移除这个功能了；
3. 解析器会通过词法分析识别出关键字 update，表名等等，构建出语法树，接着还会做语法分析，判断输入的语句是否符合 MySQL 语法；
4. 预处理器会判断表和字段是否存在；
5. 优化器确定执行计划，因为 where 条件中的 id 是主键索引，所以决定要使用 id 这个索引；
6. 执行器负责具体执行，会调用存储引擎的接口，通过主键索引树搜索获取 id = 1 这一行记录：
   * 如果 id=1 这一行所在的数据页本来就在 buffer pool 中，就直接返回给执行器更新；
   * 如果记录不在 buffer pool，将数据页从磁盘读入到 buffer pool，返回记录给执行器。
7. 执行器得到聚簇索引记录后，会看一下更新前的记录和更新后的记录是否一样：
   * 如果一样的话就不进行后续更新流程；
   * 如果不一样的话就把更新前的记录和更新后的记录都当作参数传给 InnoDB 层，让 InnoDB 真正的执行更新记录的操作；
8. 开启事务，获取表级别意向独占锁，当成功获取表级别意向独占锁后，对指定id的数据记录加Record Lock。此时如果有另一条修改事务将获取不到意向独占锁，将等待前面事务提交后才可以继续修改。
9. InnoDB 层更新记录前，首先要记录相应的 undo log，因为这是更新操作，需要把被更新的列的旧值记下来，也就是要生成一条 undo log，undo log 会写入 Buffer Pool 中的 Undo 页面，不过在内存修改该 Undo 页面后，需要记录对应的 redo log。
10. InnoDB 层开始更新记录，会先更新内存（同时标记为脏页），然后将记录写到 redo log 里面，这个时候更新就算完成了。为了减少磁盘I/O，不会立即将脏页写入磁盘，后续由后台线程选择一个合适的时机将脏页写入到磁盘。这就是  **WAL 技术** ，MySQL 的写操作并不是立刻写到磁盘上，而是先写 redo 日志，然后在合适的时间再将修改的行数据写到磁盘上。
11. 至此，一条记录更新完了。
12. 在一条更新语句执行完成后，然后开始记录该语句对应的 binlog，此时记录的 binlog 会被保存到 binlog cache，并没有刷新到硬盘上的 binlog 文件，在事务提交时才会统一将该事务运行过程中的所有 binlog 刷新到硬盘。
13. 事务提交（为了方便说明，这里不说组提交的过程，只说两阶段提交）：
    * **prepare 阶段** ：将 redo log 对应的事务状态设置为 prepare，然后将 redo log 刷新到硬盘；
    * **commit 阶段** ：将 binlog 刷新到磁盘，接着调用引擎的提交事务接口，将 redo log 状态设置为 commit（将事务设置为 commit 状态后，刷入到磁盘 redo log 文件）；
14. 至此，一条更新语句执行完成。

## Mysql有哪些锁

在 MySQL 里，根据加锁的范围，可以分为**全局锁、表级锁和行锁**三类。

![](https://cdn.xiaolincoding.com//mysql/other/1e37f6994ef44714aba03b8046b1ace2.png)

## 表空间文件的结构是怎么样的？

 **表空间由段（segment）、区（extent）、页（page）、行（row）组成** ，InnoDB存储引擎的逻辑存储结构大致如下图：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/row_format/%E8%A1%A8%E7%A9%BA%E9%97%B4%E7%BB%93%E6%9E%84.drawio.png)

下面我们从下往上一个个看看。

1、行（row）

数据库表中的记录都是按行（row）进行存放的，每行记录根据不同的行格式，有不同的存储结构。

后面我们详细介绍 InnoDB 存储引擎的行格式，也是本文重点介绍的内容。

2、页（page）

记录是按照行来存储的，但是数据库的读取并不以「行」为单位，否则一次读取（也就是一次 I/O 操作）只能处理一行数据，效率会非常低。

因此， **InnoDB 的数据是按「页」为单位来读写的** ，也就是说，当需要读一条记录的时候，并不是将这个行记录从磁盘读出来，而是以页为单位，将其整体读入内存。

 **默认每个页的大小为 16KB** ，也就是最多能保证 16KB 的连续存储空间。

页是 InnoDB 存储引擎磁盘管理的最小单元，意味着数据库每次读写都是以 16KB 为单位的，一次最少从磁盘中读取 16K 的内容到内存中，一次最少把内存中的 16K 内容刷新到磁盘中。

页的类型有很多，常见的有数据页、undo 日志页、溢出页等等。数据表中的行记录是用「数据页」来管理的，数据页的结构这里我就不讲细说了，之前文章有说过，感兴趣的可以去看这篇文章：[换一个角度看 B+ 树(opens new window)](https://xiaolincoding.com/mysql/index/page.html)

总之知道表中的记录存储在「数据页」里面就行。

3、区（extent）

我们知道 InnoDB 存储引擎是用 B+ 树来组织数据的。

B+ 树中每一层都是通过双向链表连接起来的，如果是以页为单位来分配存储空间，那么链表中相邻的两个页之间的物理位置并不是连续的，可能离得非常远，那么磁盘查询时就会有大量的随机I/O，随机 I/O 是非常慢的。

解决这个问题也很简单，就是让链表中相邻的页的物理位置也相邻，这样就可以使用顺序 I/O 了，那么在范围查询（扫描叶子节点）的时候性能就会很高。

那具体怎么解决呢？

 **在表中数据量大的时候，为某个索引分配空间的时候就不再按照页为单位分配了，而是按照区（extent）为单位分配。每个区的大小为 1MB，对于 16KB 的页来说，连续的 64 个页会被划为一个区，这样就使得链表中相邻的页的物理位置也相邻，就能使用顺序 I/O 了** 。

4、段（segment）

表空间是由各个段（segment）组成的，段是由多个区（extent）组成的。段一般分为数据段、索引段和回滚段等。

* 索引段：存放 B + 树的非叶子节点的区的集合；
* 数据段：存放 B + 树的叶子节点的区的集合；
* 回滚段：存放的是回滚数据的区的集合，之前讲[事务隔离 **(opens new window)**](https://xiaolincoding.com/mysql/transaction/mvcc.html)的时候就介绍到了 MVCC 利用了回滚段实现了多版本查询数据。

好了，终于说完表空间的结构了。接下来，就具体讲一下 InnoDB 的行格式了。

之所以要绕一大圈才讲行记录的格式，主要是想让大家知道行记录是存储在哪个文件，以及行记录在这个表空间文件中的哪个区域，有一个从上往下切入的视角，这样理解起来不会觉得很抽象。

## COMPACT 行格式长什么样？

先跟 Compact 行格式混个脸熟，它长这样：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/row_format/COMPACT.drawio.png)

可以看到，一条完整的记录分为「记录的额外信息」和「记录的真实数据」两个部分。

记录的额外信息包含 3 个部分：变长字段长度列表、NULL 值列表、记录头信息。

## MySQL 日志：undo log、redo log、binlog 有什么用？

更新语句的流程会涉及到 undo log（回滚日志）、redo log（重做日志） 、binlog （归档日志）这三种日志：

* **undo log（回滚日志）** ：是 Innodb 存储引擎层生成的日志，实现了事务中的 **原子性** ，主要 **用于事务回滚和 MVCC** 。
* **redo log（重做日志）** ：是 Innodb 存储引擎层生成的日志，实现了事务中的 **持久性** ，主要 **用于掉电等故障恢复** ；
* **binlog （归档日志）** ：是 Server 层生成的日志，主要 **用于数据备份和主从复制** ；

> redo log 和 undo log 区别在哪？

这两种日志是属于 InnoDB 存储引擎的日志，它们的区别在于：

* redo log 记录了此次事务「 **完成后** 」的数据状态，记录的是更新**之后**的值；
* undo log 记录了此次事务「 **开始前** 」的数据状态，记录的是更新**之前**的值；

所以这次就带着这个问题，看看这三种日志是怎么工作的。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E6%8F%90%E7%BA%B2.png)

## 并发事务会产生哪些问题？事务隔离级别是如何做到的？
