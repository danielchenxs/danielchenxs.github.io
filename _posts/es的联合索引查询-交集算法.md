---

layout:     post
title:      "es的联合索引查询-交集算法"
subtitle:   ""
date:       2022-02-10
author:     "Chenxs"
header-img: "img/post-bg-js-version.jpg"
tags:
    - 跳表
    - es
---

![](http://static.chenxs.com/img/fix-dir/2022/02/18/17-43-14-5a95a7f7f60157ba5c6c12d5d4938ea7-20220218174313-9bbae5.png)

- >通过不同索引查出多个结果，进行取交集得到最后结果
- > 即使不是**多条件查询**，由于Lucene是**分片**的，也经常需要频繁求交集


### PostingList的优化
- 如何压缩以节省磁盘空间 [[es 压缩算法 -Frame Of Reference]]
- 如何快速求交集


### 算法一：bitmap，适合大数据量
-   算法思想基于bitmap(布隆过滤器结构)的简化实现
-   posting list为[1,3,4,7,10]， 则bitset为[1,0,1,1,0,0,1,0,0,1]，一共十个，1表示有数，0表示数据不存在。1byte就能表示八个数据，能节省空间。
-   三个posting list的bitset可以做and操作就能挑出交集。
- 需要 65536 （2^16）个 bit，也就是 65536/8 = 8192 bytes，而用 Integer 数组，只需要 2 * 2 bytes = 4 bytes。计算一下临界值，很简单，无论文档数量多少，bitmap 都需要 8192 bytes，而 Integer 数组则和文档数量成线性相关，每个文档 ID 占 2 bytes，所以：
	- 8192 / 2 = 4096
	- 当文档数量少于 4096 时，用 Integer 数组，否则，用 bitmap。

### 算法二：skiplist（Integer 数组），适合小数据量
#### 复杂度分析
- n/2、n/4、n/8、第k级点个数n/(2^k)： 约高层的概率越低
- k=log2(n)-1
- 时间复杂度 O(logn) 空间复杂度O(n)

![](http://static.chenxs.com/img/fix-dir/2022/02/19/16-44-27-c448411c7548421d482d8086a0821cba-20220219164427-4203aa.png)

- 找交集：从最短的posting list进行遍历查找