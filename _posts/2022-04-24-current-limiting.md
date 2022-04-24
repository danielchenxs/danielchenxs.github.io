---

layout:     post
title:      "限流的几种方案"
subtitle:   ""
date:       2022-02-21
author:     "Chenxs"
header-img: "img/post-bg-js-version.jpg"
tags:
    - cacheline
    - 伪共享
    - disruptor
---
# 限流的几种方案
### 令牌桶
- 原理
	- 从令牌桶中获取令牌，拿到即为允许执行。
- 优点：
	- 可以应对瞬间的大流量，短时间拿到大量令牌
	- 可以控制填充令牌的速度
- 实现
	- 获取指定tokens_key（上次的可用数）、timestamp_key（上次的时间）的值
	- 计算当前和上次时间的差，先计算中间生成的数量，再计算可用数（min(上限，上次可用数+中间补充值)）
	- 当可用>请求数，true
	- 设置可用数和当前时间，ttl（>填充满的时间）
- 例子
  ```lua
	-- value= 可用数
	local tokens_key = KEYS[1]  
	-- value= 上次时间
	local timestamp_key = KEYS[2]  
	--redis.log(redis.LOG_WARNING, "tokens_key " .. tokens_key)  
	-- 填充速度
	local rate = tonumber(ARGV[1])  
	-- 上限
	local capacity = tonumber(ARGV[2])  
	local now = tonumber(ARGV[3])  
	-- 请求数
	local requested = tonumber(ARGV[4])  
	-- 计算填充时间  
	local fill_time = capacity/rate  
	-- ttl为2倍填充时间
	local ttl = math.floor(fill_time*2)  
	-- 上次可用数
	local last_tokens = tonumber(redis.call("get", tokens_key))  
	if last_tokens == nil then  
	  last_tokens = capacity  
	end  
	-- 上次时间  
	local last_refreshed = tonumber(redis.call("get", timestamp_key))  
	if last_refreshed == nil then  
	  last_refreshed = 0  
	end  
	-- 两次时间间隔  
	local delta = math.max(0, now-last_refreshed)  
	-- 加上补充数，计算可用数
	local filled_tokens = math.min(capacity, last_tokens+(delta*rate))  
	local allowed = filled_tokens >= requested  
	local new_tokens = filled_tokens  
	local allowed_num = 0  
	if allowed then  
	  new_tokens = filled_tokens - requested  
	  allowed_num = 1  
	end  
	-- 设置可用数  
	redis.call("setex", tokens_key, ttl, new_tokens)  
	-- 设置可用时间
	redis.call("setex", timestamp_key, ttl, now)  
	  
	return { allowed_num, new_tokens }
  ```

### 漏斗
- 原理
	- 从漏桶中加水，不溢出，则在**队列中排队等待执行**
- 实现
	```lua
	-- key
	local leaky_bucket_key = KEYS[1]  
	-- last update key  
	local last_bucket_key = KEYS[2]  
	-- capacity  
	local capacity = tonumber(ARGV[2])  
	-- the rate of leak water  
	local rate = tonumber(ARGV[1])  
	-- request count  
	local requested = tonumber(ARGV[4])  
	-- current timestamp  
	local now = tonumber(ARGV[3])  
	-- the key life time  
	local key_lifetime = math.ceil((capacity / rate) + 1)  
	  
	  
	-- the yield of water in the bucket default 0  
	local key_bucket_count = tonumber(redis.call("GET", leaky_bucket_key)) or 0  
	  
	-- the last update time default now  
	local last_time = tonumber(redis.call("GET", last_bucket_key)) or now  
	  
	-- the time difference  
	local millis_since_last_leak = now - last_time  
	  
	-- the yield of water had lasted  
	local leaks = millis_since_last_leak * rate  
	  
	if leaks > 0 then  
	 -- clean up the bucket  
	 if leaks >= key_bucket_count then  
	 key_bucket_count = 0  
	 else  
	 -- compute the yield of water in the bucket  
	 key_bucket_count = key_bucket_count - leaks  
	 end  
	 last_time = now  
	end  
	  
	-- is allowed pass default not allow  
	local is_allow = 0  
	  
	local new_bucket_count = key_bucket_count + requested  
	-- allow  
	if new_bucket_count <= capacity then  
	 is_allow = 1  
	else  
	 -- not allow  
	 return {is_allow, new_bucket_count}  
	end  
	  
	-- update the key bucket water yield  
	redis.call("SETEX", leaky_bucket_key, key_lifetime, new_bucket_count)  
	  
	-- update last update time  
	redis.call("SETEX", last_bucket_key, key_lifetime, now)  
	  
	-- return  
	return {is_allow, new_bucket_count}
	```

### 滑动窗口
- 是[[限流#滑动窗口]]的一种优化
- 流程
	- 获取上次请求次数，看是否小于上限
	- 新增到队列
	- 清除过期计数（一个窗口之前的所有）
	- 设置过期时间一个窗口期
- 实现例子
  ```lua
    local tokens_key = KEYS[1]  
	local timestamp_key = KEYS[2]  
	--redis.log(redis.LOG_WARNING, "tokens_key " .. tokens_key)  
	-- 速率	  
	local rate = tonumber(ARGV[1])  
	-- 上限
	local capacity = tonumber(ARGV[2])  
	-- 当前时间
	local now = tonumber(ARGV[3])  
	-- 上限填满时间  
	local window_size = tonumber(capacity / rate)  
	local window_time = 1  
	  
	--redis.log(redis.LOG_WARNING, "rate " .. ARGV[1])  
	--redis.log(redis.LOG_WARNING, "capacity " .. ARGV[2])  
	--redis.log(redis.LOG_WARNING, "now " .. ARGV[3])  
	--redis.log(redis.LOG_WARNING, "window_size " .. window_size)  
	
	-- 默认上次请求了0次  
	local last_requested = 0  
	-- 是否存在key
	local exists_key = redis.call('exists', tokens_key)  
	if (exists_key == 1) then  
	 -- 如果存在，获取上次请求次数
	 last_requested = redis.call('zcard', tokens_key)  
	end  
	--redis.log(redis.LOG_WARNING, "last_requested " .. last_requested)  
	-- 剩余可用次数  
	local remain_request = capacity - last_requested  
	local allowed_num = 0  
	if (last_requested < capacity) then  
	 allowed_num = 1  
	 redis.call('zadd', tokens_key, now, timestamp_key)  
	end  
	  
	--redis.log(redis.LOG_WARNING, "remain_request " .. remain_request)  
	--redis.log(redis.LOG_WARNING, "allowed_num " .. allowed_num)  
	
	-- 去除一个时间窗口之前的过期计数  
	redis.call('zremrangebyscore', tokens_key, 0, now - window_size / window_time)  
	-- key 设置过期时间
	redis.call('expire', tokens_key, window_size)  
	  
	return { allowed_num, remain_request }
  ```

### 计数器
- 每几秒请求几次，粗暴简单
- 缺点：容易在临界值，被请求2倍的量