---

layout:     post
title:      "索引失效的几种情况"
subtitle:   ""
date:       2018-01-15
author:     "Chenxs"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Mysql
    - 索引
---
![](/media/15160210700337.jpg)

1.`索引字段参与函数计算`
``` sql
SELECT * FROM t_order WHERE concat('UN',Status)='UNPUSH';
SELECT * FROM t_order WHERE 2 =type+1;(索引失效)
SELECT * FROM t_order WHERE type=2-1;(索引有效)
```
<!-- more -->
2.`类型隐式转换`
``` sql
(status char(1))
SELECT * FROM t_order WHERE status=1;（索引无效）
SELECT * FROM t_order WHERE status='1'(索引有效);
```
3.`like 前置%`
``` sql
SELECT * FROM t_order WHERE proName like '%a';(索引无效)
SELECT * FROM t_order WHERE proName like 'a%';（索引有效）
```
4. or 两边字段有非索引
5.当变量采用的是times变量，而表的字段采用的是date变量时.或相反情况。 
6.使用not in ,not exist等语句时。



