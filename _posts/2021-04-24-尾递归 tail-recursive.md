---

layout:     post
title:      "尾递归 tail-recursive"
subtitle:   ""
date:       2022-02-10
author:     "Chenxs"
header-img: "img/post-bg-js-version.jpg"
tags:
    - SICP
---
# 尾递归 tail-recursive
- 在计算机程序调用中，需要不断的入栈出栈，当后面的调用依赖前一次的结果，就会一直入栈，最坏的情况可能栈溢出
- **尾递归**是**尾调用(tail -call)** 的一种特殊且重要的场景
- 定义：
	- 当递归调用函数在函数的末尾
		- 如何处理：把前一次的结果，作为中间变量作为入参传入
	- **编译器优化**
		- 由于上个栈是函数末尾，不再使用了
		- 直接覆盖栈，节约出入栈的时间，且没有栈溢出风险。
	- 例子
		- 线性递归调用,保存n个调用记录
			```javascript
			function factorial(n) {
			  if (n === 1) return 1;
			  return n * factorial(n - 1);
			}
			
			factorial(5) // 120
			```
		- 改写成尾递归，`total`保存中间值
			```javascript
			function factorial(n, total) {
			  if (n === 1) return total;
			  return factorial(n - 1, n * total);
			}
			
			factorial(5, 1) // 120
			```
		- 再改进，不暴露第二个参数
			```javascript
			function tailFactorial(n, total) {
			  if (n === 1) return total;
			  return tailFactorial(n - 1, n * total);
			}
			
			function factorial(n) {
			  return tailFactorial(n, 1);
			}
			
			factorial(5) // 120
			```