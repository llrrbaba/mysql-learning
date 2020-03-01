#### MRR

[MySQL MRR 5.6文档链接](https://dev.mysql.com/doc/refman/5.6/en/mrr-optimization.html)

通过二级索引进行**范围查询**会导致大量的**随机磁盘访问**，当表比较大并且没有存储在存储引擎的缓存中时。通过Disk-Sweep Multi-Range Read(MRR)优化策略，MySQL尝试减少随机磁盘访问的数量通过首先查找二级索引，并且收集对应行的主键。然后将收集到的主键进行排序并按照排完序的主键进行回表查询。

Disk-Sweep MRR的目的就是减少随机磁盘访问，实现对磁盘上表的更加顺序化的访问。

MRR优化策略提供了这些好处：

- 基于 index tuples(这里的tuples应该就是二级索引和主键组成的元组吧)，MRR使得数据行被顺序访问而不是随机访问。Server获得一个满足查询条件的index tuples的set集合，将这个集合的数据按照主键id的顺序进行排序，然后使用排好序的tuples去获取每一行数据。这使得数据的访问更加高效，成本更低。
- 对于需要通过index tuples访问数据行的操作，MRR支持对键访问请求进行批处理，例如，范围索引扫描和为连接属性使用索引的等连接。MRR迭代一系列的索引范围，以获得合格的index tuples，随着这些结果的累积，将使用它们访问相应的数据行。在开始读取数据行之前，没有必要获取所有的index tuples。

下面的场景说明了什么时候MRR优化是有利的：

场景A: MRR可以用于InnoDB和MyISAM表，用于索引范围扫描和等连接操作。

1.索引元组的一部分累积在缓冲区中。

2.缓冲区中的元组按其数据行id排序。

3.数据行根据排序的索引元组序列进行访问。

使用MRR时，EXPLAIN输出中的额外列显示`Using MRR`。

如果不需要访问完整的表行来产生查询结果，InnoDB和MyISAM不使用MRR。如果可以完全根据索引元组中的信息(通过覆盖索引)生成结果，则会出现这种情况;MRR没有任何好处。

两个optimizer_switch系统变量标志提供了一个使用MRR优化的接口。mrr标志控制是否启用mrr。如果启用了mrr (on)， `mrr_cost_based`标志将控制优化器是否尝试在使用和不使用mrr (on)或尽可能使用mrr (off)之间做出基于成本的选择。默认情况下，mrr为on, mrr_cost_based为on。参见8.9.2节，[“Switchable Optimizations”](https://dev.mysql.com/doc/refman/5.6/en/switchable-optimizations.html)。

对于MRR，存储引擎使用`read_rnd_buffer_size`系统变量的值作为为其缓冲区分配多少内存的准则。该引擎最多使用`read_rnd_buffer_size`字节，并确定在一次传递中要处理的范围数。
