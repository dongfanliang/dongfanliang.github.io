---
title: golang 之 golang.org/x/time/rate
date:  2020-03-02 22:18:01 +0800
category: golang
tags: golang
excerpt:
---

##### 使用示例：
``` GO
package main

import (
	"golang.org/x/time/rate"
)

func main() {
	limit := rate.NewLimiter(2, 10)  // 每秒生成2个token， 桶大小为10
	for i := 0; i < 20; i++ {
		println(limit.Allow(), i)
	}
}

// 输出：
true 0
true 1
true 2
true 3
true 4
true 5
true 6
true 7
true 8
true 9
false 10
false 11
false 12
false 13
false 14
false 15
false 16
false 17
false 18
false 19
```
### 令牌桶实方式实现限速
[golang.org/x/time/rate](https://github.com/golang/time/blob/master/rate/rate.go)

#### 核心思想：如果生成token速率一定，那么**token数** 和 **时间段**可以相互转换；      


惰性生成的，只有有请求的时候，才会触发计算token数；    
#### 核心代码
``` GO
// 这里只关注Allow()方法实现，其他方法也类似，可以看文档或者源码

type Limit float64

type Limiter struct {
	limit Limit     // 每秒产生多少个token            
	burst int       // 桶大小

	mu     sync.Mutex
	tokens float64   // 当前token数
	last time.Time
	lastEvent time.Time
}

// 客户端获取token，桶里有返回true，否则返回false
func (lim *Limiter) Allow() bool {
	return lim.AllowN(time.Now(), 1)
}

// 也可以同时获取多个token
func (lim *Limiter) AllowN(now time.Time, n int) bool {
	return lim.reserveN(now, n, 0).ok
}

func (lim *Limiter) reserveN(now time.Time, n int, maxFutureReserve time.Duration) Reservation {
    // tokens 为现在桶的token数
	now, last, tokens := lim.advance(now)

	// n为需要消费的tokens，这个结果可能负数
	tokens -= float64(n)

	var waitDuration time.Duration
	if tokens < 0 {
        // 结果小于0 意味着桶内没有token需要等待一段时间生成token
        // 计算需要等待时长
		waitDuration = lim.limit.durationFromTokens(-tokens)
	}

    // AllowN调用时 maxFutureReserve=0
	// 如果请求数N >桶大小 或者 waitDuration 等待时长 > 0；结果为false
	ok := n <= lim.burst && waitDuration <= maxFutureReserve  

	// 统一的结果struct，包括等待时长等等，这里只关注ok
	r := Reservation{
		ok:    ok,  // 是否能够获取到n个token
		lim:   lim,
		limit: lim.limit,
	}
	if ok {
        r.tokens = n
        // 如果等待时间 > 0  timeToAct表示未来的一个时刻，桶内会有n个token
		r.timeToAct = now.Add(waitDuration)  
	}

    // 更新各种状态
	if ok {
		lim.last = now  // 更新时间
		lim.tokens = tokens  // 更新桶内token数
		lim.lastEvent = r.timeToAct
	} else {
		lim.last = last
	}

	lim.mu.Unlock()
	return r
}

func (lim *Limiter) advance(now time.Time) (newNow time.Time, newLast time.Time, newTokens float64) {
	last := lim.last
	if now.Before(last) {
		last = now
	}

    // 根据token数计算填满桶需要的时长
    maxElapsed := lim.limit.durationFromTokens(float64(lim.burst) - lim.tokens)
    // 当前时间 - 上次更新时间，如果elapsed特别大，也就是last很久没更新，相当于桶为空
	elapsed := now.Sub(last)
	if elapsed > maxElapsed {
		elapsed = maxElapsed
	}

	// 根据时间段计算token数
    delta := lim.limit.tokensFromDuration(elapsed)
    // 当前桶里的token数 = 桶内现有token数 + 这段时间生成的token数
	tokens := lim.tokens + delta  
	if burst := float64(lim.burst); tokens > burst {
		tokens = burst
	}

	return now, last, tokens
}
```

#### 相关开源项目：
```
// 令牌桶方式， 以golang.org/x/time/rate为基础，提供gin的中间件
https://github.com/didip/tollbooth

// 经过优化的漏斗方式， 提供最大松弛量等方案
https://github.com/uber-go/ratelimit/
```



