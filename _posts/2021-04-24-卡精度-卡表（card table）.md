- 精确到一块内存区域
- 是rememberSet的一种实现
- 最简单的形式
	- 字节数组
		```
		CARD_TABLE [this address >> 9] = 0;
		```
		- 每个元素对应一个内存块---`卡页（Card Page）`
		- ![](http://static.chenxs.com/img/fix-dir/2022/03/01/19-26-37-69422758dc56176505652d18c94eee80-20220301192635-7614ae.png)
	- 哈希表（G1）
		- [[GC-G1细节#RSet Remember Set |G1 R-Set]]
- 如何维护？
	- 写屏障
		- 用来维护卡表的，并非内存屏障
		- 类似于-`引用类型字段赋值`动作的AOP
			- 写前屏障（Pre-Write Barrier）
				- G1用到了写前和写后，队列操作再异步，操作复杂消耗更多CPU
			- 写后屏障（Post-Write Barrier）
				- CMS等只用到写后，同步
		- [[Cache Line伪共享]]情况下，反复更新的问题
			- 在JDK 7之后，HotSpot虚拟机增加了一个新的参数`-XX：+UseCondCardMark` 决定是否开启多一次判断，两者性能各有损耗
				- 增加判断，当未被标记`dirty`，才标记
					```java
					if (CARD_TABLE [this address >> 9] != 0) 
						CARD_TABLE [this address >> 9] = 0;
					```