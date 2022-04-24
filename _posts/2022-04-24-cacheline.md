---

layout:     post
title:      "Cache Line伪共享"
subtitle:   ""
date:       2022-02-21
author:     "Chenxs"
header-img: "img/post-bg-js-version.jpg"
tags:
    - cacheline
    - 伪共享
    - disruptor
---
# Cache Line伪共享
![](http://static.chenxs.com/img/fix-dir/2022/02/02/11-20-31-632a4abbeb3f1b39d8d72354635f10f0-20220202112031-815d8c.png)

![](http://static.chenxs.com/img/fix-dir/2022/02/02/11-55-57-4ab5f3f0e7450188c89ef178297ef283-20220202115557-8177bb.png)
- cpu结构：
	- L1 ,L2 是每个core独有的
	- L3是单个cpu插槽上的所有cpu共享的


- 伪共享
	- 缓存系统以缓存行（cacheline）为单位存储的，当多线程修改相互独立的变量时，如果这些变量共享一个缓存行，就会无意中影响彼此的性能，就是伪共享（false sharing）
- 如何避免
	- 让不同线程操作的对象处于不同缓存行
		- 手动添加padding属性
			- 对象内不同的属性分开：一个long是8字节，一个缓存行64字节。8个字节可以分开。
			- 一般可以把变化和不变化的分开
			- 比如在一个long属性的先7后7都设置7个long，则这个属性一定是个单独缓存行
				```java
				public long p1, p2, p3, p4, p5, p6, p7; // cache line padding
				private volatile long cursor = INITIAL_CURSOR_VALUE;
				public long p8, p9, p10, p11, p12, p13, p14; // cache line padding
				```

				
				```java
			  //com.lmax.disruptor.LhsPadding
				  
				class LhsPadding  
				{  
				 protected long p1, p2, p3, p4, p5, p6, p7;  
				}  
				  
				class Value extends LhsPadding  
				{  
				 protected volatile long value;  
				}  
				  
				class RhsPadding extends Value  
				{  
				 protected long p9, p10, p11, p12, p13, p14, p15;  
				}
							
				```


		  - 先创建一个2x8的数组，再在指定位置放入值

			```java
			public class ContendedAtomicLong extends Contended {
			
				//	一个缓存行需要多少个Long元素的填充：8
			    private static final int CACHE_LINE_LONGS = CACHE_LINE / Long.BYTES;
			
			    private final AtomicLongArray contendedArray;
			
			    //	77
			    ContendedAtomicLong(final long init)
			    {
			        contendedArray = new AtomicLongArray(2 * CACHE_LINE_LONGS);
			        set(init);
			    }
			
			    void set(final long l) {
			        contendedArray.set(CACHE_LINE_LONGS, l);
			    }
			
			    long get() {
			        return contendedArray.get(CACHE_LINE_LONGS);
			    }
			}				
			```

			- 把不同对象分开：Java 程序的对象头固定占 8 字节(32位系统)或 12 字节( 64 位系统默认开启压缩, 不开压缩为 16 字节)，所以我们只需要填 6 个无用的长整型补上6*8=48字节，超过64也是可以的
			- 某些编译器会优化不用的属性，我们可以加个方法使用这些padding属性。防止优化
		- `@Contended` 注解 && 虚拟机参数`-XX:-RestrictContended`
			- java8
			- 可以避免编译优化的问题
			- 可以加载该字段或者类上
	- 缺点：
		- cache寸土寸金，填充是否合算需要考虑

	- 应用场景
		- LongAddr
			- 使用cells数组，将cas失败的放到另一个cell执行（**填充缓存行，避免伪共享**）在获取时合并结果
			- 
				```java
				@sun.misc.Contended static final class Cell {
					volatile long value;
					Cell(long x) { value = x; }
					final boolean cas(long cmp, long val) {
						return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
					}
				//...
				}
				```

