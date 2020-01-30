ORA-14402：更新分区关键字列将导致分区更改（分区表注意）
建立完分区表后一定要和开发确认一点，就是是否会修改分区字段。因为update分区字段到其他分区时候，会报错。
解决办法：开启表的行转移功能

1
alter table XX enable row movement

这样在update以后，会在老分区删除数据，新分区插入数据。

一般用于分区表，将row movement设置为enable,有可能发生行的物理移动，行的rowdi会变化，某一行更新时，如果更新的是分区列，并且更新后的列值不属于原来的这个分区，如果开启了这个选项，就会把这行从这个分区中delete掉，并加到更新后所属的分区。相当于一个隐式的触发器，但不会触发Insert/delete触发器。如果没有开启这个选项，更新时就会报错。
--------------------- 
作者：bitko 
来源：CSDN 
原文：https://blog.csdn.net/gumengkai/article/details/51066136 
版权声明：本文为博主原创文章，转载请附上博文链接！


Oracle Index-organized table (IOT)概述

一、几种表类型

Type

Description

Ordinary(heap-organized) table

Data is stored as an unordered collection (heap).

Partitioned table

Data is divided into smaller, more manageable pieces.

Index-organized table (IOT)

Data (including no-key values) is sorted an stored in a B-tree index structure.

Clustered table

Related data from more than one table are stored together.

 

 

二、heap-organized table 和 Index-organized table对比

 

不同于heap-organized table，数据以无序堆形式存储，IOT中的数据按主键排序的方式存储在B-tree索引结构中。除了存储主键列值之外，IOT的B-tree的每个索引条目还存储非键列值（non-key column）。

Index-organized table具备完整的表功能。它们支持各种功能，如：约束条件、触发器、LOG和object columns，分区、并行操作、联机重组以及复制。甚至可以对索引组织的表创建索引。

索引组织表与普通表只是在屋里组织结构上有差别。逻辑上，其处理方式与普通表相同。可以像指定常规表那样，在INSERT、SELECT、DELETE和UPDATE语句中指定按索引组织的表。

Index-organized table非常适合OLTP应用程序。这类应用程序要求基于primary key实现快速访问和高可用性。在联机订单处理中，对订单表执行的查询和DML操作主要就是基于主键，而大量的DML操作会产生碎片，因而需要频繁的进行重组。由于IOT可以联机重组，并且不会使其二级索引失败，因此可以极大地缩短甚至消除不可用的时间段。

 

与堆表相比，IOT：可以基于主键更快地访问表数据

                  不会复制主键值的存储区

                  要求的存储空间更少

                  使用二级索引和逻辑ROW ID

                  可用性更高，因为表重组时不会使二级索引失效

 

IOT有一下限制：必须有一个不是DEFERRABLE的主键

                不能聚簇

                不能使用组合分区

                不能包含类型为ROWID或LONG的列

 

Index-organized table没有物理ROWID，而是使用逻辑row ID。逻辑row ID为Index-organized table提供更快的访问，通过使用一下两种方法：

l   A physical guess whose access time is equal to that of physical row IDs.（物理推测法，其访问时间相当于对物理ROWID的访问时间）

l   Access without the guess (or after an incorrect guess); this performs a primary key access of the IOT.（不推测或在推测错误后访问，这将对IOT执行主键访问）

 

推测的基础是对行所在的文件和块的了解。如果创建了索引，则块信息比较准确，但如果叶块发生了拆分，则该信息会改变。如果推测错误，且该行已不在指定的块中，则将使用逻辑ROW ID条目的剩余部分（即主键）来获取该行。

Oracle使用基于表主键的逻辑ROW ID来对Index-organized table编制二级索引。由于IOT中的行没有永久性物理地址，因此当行转移到新块中后，物理推测也会失效。此时，要获取新的推测，可以重建二级索引。

 

 

三、创建Index-organized table

 

create table indexTable

( ID varchar2(10),

   Name varchar2(20),

   constraint pk_id primary key(ID)

)

organization index

tablespace index

PCTTHRESHOLD 20

OVERFLOW TABLESPACE users

INCLUDING name;

 

1、 创建IOT时，必须设定主键，否则报错。

2、 索引组织表实际上将所有数据都放入了索引中。

3、 OVERFLOW子句（行溢出），因为所有数据都放入索引，所以当表的数据量很大时，会降低索引组织表的查询性能。此时设置溢出段将主键和溢出数据分开来存储以提高效率。溢出段的设置有两种格式：

PCTTHRESHOLD n：指定一个数据块的百分比，当行数据占用大小超出时，该行的其他列数据放入溢出段。

INCLUDING column_name：指定列之前的列都放入索引块，之后的列都放到溢出段。

如上所示，name之后的列必然被放入溢出列，而其他列根据PCTTHRESHOLD规则。

4、 compress子句（键压缩）。与普通的索引一样，索引组织表也可以使用compress子句进行键压缩以消除重复值。具体的操作是，在organization index之后加上COMPRESS n子句。

n的意义在于：指定压缩的列数。默认为无穷大。例如：对于数据（1,2,3）、（1,2,4）、（1，2,5）、（1,3,4）、（1,3,5），若使用COMPRESS，则会将重复出现的（1,2）、（1,3）进行压缩；若使用COMPRESS 1时，只对数据（1）进行压缩

5、 索引组织表的主键不能被删除、禁用或延迟。

 

四、在Index-organized table上创建Bitmap索引

Oracle supports bitmap indexes on partitioned and nonpartitioned index-organized tables. A mapping table is required for creating bitmap indexes on an index-organized table.

--可以再Index-organized table上创建bitmap索引，但是需要创建mapping table。

The mapping table is a heap-organized table that stores logical rowids of the index-organized table. Specifically, each mapping table row stores one logical rowid for the corresponding index-organized table row. Thus, the mapping table provides one-to-one mapping between logical rowids of the index-organized table rows and physical rowids of the mapping table rows.

--mapping table是heap-organized table，mapping table存储Index-organized table的逻辑ROW ID。特别的，每一个mapping table行存储一个相应IOT 行的逻辑ROW ID。这样，mapping table就提供了一对一的mapping，在IOT的逻辑ROW ID和mapping table的物理ROWID之间。

A bitmap index on an index-organized table is similar to that on a heap-organized table except that the rowids used in the bitmap index on an index-organized table are those of the mapping table as opposed to the base table. There is one mapping table for each index-organized table and it is used by all the bitmap indexes created on that index-organized table.

--IOT上的bitmap索引的ROWID是mapping table的，而不是base table的。

一个mapping table对应于一个IOT，IOT上所有的bitmap index都用这一个mapping table。

In both heap-organized and index-organized base tables, a bitmap index is accessed using a search key. If the key is found, the bitmap entry is converted to a physical rowid. In the case of heap-organized tables, this physical rowid is then used to access the base table. However, in the case of index-organized tables, the physical rowid is then used to access the mapping table. The access to the mapping table yields a logical rowid. This logical rowid is used to access the index-organized table.

--在heap-organized table和IOT上，bitmap都是通过键值访问，当一个键值被发现时，bitmap条目就转换为一个物理ROWID。在heap-organized table的情况下，这一物理ROWID被用来访问基表。然而，在IOT的情况下，物理ROWID被用来访问mapping table，通过对mapping table的访问，产生出一个逻辑ROW ID，这个逻辑ROW ID被用来访问IOT。

Though a bitmap index on an index-organized table does not store logical rowids, it is still logical in nature.

Movement of rows in an index-organized table does not leave the bitmap indexes built on that index-organized table unusable. Movement of rows in the index-organized table does invalidate the physical guess in some of the mapping table's logical rowid entries. However, the index-organized table can still be accessed using the primary key.

--IOT上的行移动不会导致其上的bitmap 索引失效。

--IOT上的行移动会使mapping table上的逻辑ROW ID条目的物理推测失效。然而，IOT依然可以通过主键来访问。