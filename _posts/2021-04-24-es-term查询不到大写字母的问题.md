---

layout:     post
title:      "es-term查询不到大写字母问题"
subtitle:   ""
date:       2022-02-10
author:     "Chenxs"
header-img: "img/post-bg-js-version.jpg"
tags:
    - 问题处理
    - es
---


### 背景
- 通过身份证查询用户
### 现象
- 如身份证123123123123X的用户查询不到
### 原因
- mapping
	```json
	  "zhengjianhm": {
		"type": "text",
		"fields": {
		  "keyword": {
			"type": "keyword",
			"ignore_above": 256
		  }
		}
	  },
	```
- `keyword` 写入不分词，默认索引是小写写入
- `term` 查询，读不分词，查不到小写的索引

##### 为什么match可以查到
- 因为match因为读分词，对关键词进行处理统一小写

### 方案
- 建立mapping的时候，指定某个filed的标准化配置normalizer
	```json
	PUT index
	{
	  "settings": {
	    "analysis": {
	      "normalizer": {
	        "my_normalizer": {
	          "type": "custom",
	          "char_filter": [],
	          "filter": ["lowercase", "asciifolding"]
	        }
	      }
	    }
	  },
	  "mappings": {
	    "type": {
	      "properties": {
	        "foo": {
	          "type": "keyword",
	          "normalizer": "my_normalizer"
	        }
	      }
	    }
	  }
	}
	```
- 查询前，对入参进行`string.toLowerCase`