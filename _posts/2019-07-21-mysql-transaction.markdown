---

layout:     post
title:      "Mysql隔离级别与MVCC"
subtitle:   ""
date:       2019-07-21
author:     "Chenxs"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Mysql
    - MVCC
    - 隔离级别
---

# mysql隔离级别与MVCC

#### 脏读，不可重复读，幻读 

> 1. `脏读` ：脏读就是指当一个事务正在访问数据，并且对数据进行了修改，而这种修改还没有提交到数据库中，这时，另外一个事务也访问 这个数据，然后使用了这个数据。 
> 2. `不可重复读` ：是指在一个事务内，多次读同一数据。在这个事务还没有结束时，另外一个事务也访问该同一数据。那么，在第一个事务中的两 次读数据之间，由于第二个事务的修改，那么第一个事务两次读到的的数据可能是不一样的。这样就发生了在一个事务内两次读到的数据是不一样的，因此称为是不 可重复读。例如，一个编辑人员两次读取同一文档，但在两次读取之间，作者重写了该文档。当编辑人员第二次读取文档时，文档已更改。原始读取不可重复。如果 只有在作者全部完成编写后编辑人员才可以读取文档，则可以避免该问题。 
> 3. `幻读` : 是指当事务不是独立执行时发生的一种现象，例如第一个事务对一个表中的数据进行了修改，这种修改涉及到表中的全部数据行。 同时，第二个事务也修改这个表中的数据，这种修改是向表中插入一行新数据。那么，以后就会发生操作第一个事务的用户发现表中还有没有修改的数据行，就好象 发生了幻觉一样。例如，一个编辑人员更改作者提交的文档，但当生产部门将其更改内容合并到该文档的主复本时，发现作者已将未编辑的新材料添加到该文档中。 如果在编辑人员和生产部门完成对原始文档的处理之前，任何人都不能将新材料添加到文档中，则可以避免该问题。

 

> 不可重复读的重点是**<u>`修改`</u>** : 
>
> 同样的条件, 你读取过的数据,再次读取出来发现值不一样了 
>
> 幻读的重点在于**<u>`新增`</u>**或者**<u>`删除`</u>** 
>
> 同样的条件, 第 1 次和第 2 次读出来的记录数不一样 

| 隔离等级         | 效果                                                      |
| ---------------- | --------------------------------------------------------- |
| READ_UNCOMMITTED | 会出现脏读、不可重复读、幻读 ( 隔离级别最低，并发性能高 ) |
| READ_COMMITTED   | 会出现不可重复读、幻读问题（锁定正在读取的行）            |
| REPEATABLE_READ  | 会出幻读（锁定所读取的所有行）                            |
| SERIALIZABLE     | 保证所有的情况不会发生（锁表）                            |



## MVCC

多版本并发控制

因为一般环境中读操作好几倍于写操作，MVCC实现了**`读写不冲突`**，高效的并发读。

> 读-读，不存在任何问题
> 读-写，有隔离性问题，可能遇到脏读（会读到未提交的数据） ，幻影读等。
> 写-写，可能丢失更新

*图片来自网络*

![image-20190711234000993](../_site/img/image-20190711234000993.png)



#### 版本链

对于InnoDB存储引擎，聚簇索引记录中都包含两个必要的隐藏列（row_id并不是必要的，我们创建的表中有主键或者非NULL唯一键时都不会包含row_id列）：

- **trx_id**：每次对某条记录改动时，都会把对应的事务id赋值给trx_id隐藏列
- **roll_pointer**：每次对某条记录进行改动时，这个隐藏列会存一个指针，可以通过这个指针找到该记录修改前的信息



####  ReadView

1. m_ids[1,2,3]: 保存活跃的事务（没提交的事务）Id, select时候排除其中的事务Id
2. min_trx_id： 活跃读写事务中最小的事务id,也是m_ids中最小值
3. max_trx_id: 应该分配给下一个事务的id
4. Creator_trx_id: 表示生成改ReadView的事务id

<!--RC -->

事务A(300)==1 ,事务B（200）==2

第一次 A select 

**readview**:m_ids:[1,2,3,4, 200,300] ； - - 1

b commit;

select

**readview:** m_ids:[1,2,3,4,300]  - - 2



<!--RR -->

事务A(300)==1 ,事务B（200）==2

第一次 A select 

**readview**:m_ids:[1,2,3,4, 200,300] ； - - 1

b commit;

select

**readview:** m_ids:[1,2,3,4, 200,300]  （依然还是第一次select的readView）  - - 1



**MVCC 存在于RC 和 RR** 

> 何时创建read view 
>
> 其实，我们从上面的解释已经明白了，在RC隔离级别下，是`每个`SELECT都会获取最新的read view；而在RR隔离级别下，则是当事务中的`第一个`SELECT请求才创建read view。

 

`RR 为什么能可重复读`： 

在事务开始时，获取最新快照，后续select 都是使用同一个快照。 

`如何防止幻读`： 

RR有Next-key锁，同时锁住记录(数据)，并且锁住记录前面的Gap ，保证在某个事务中能实现一致性。 



