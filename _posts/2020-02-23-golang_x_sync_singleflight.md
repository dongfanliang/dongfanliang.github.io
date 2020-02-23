---
title: golang 之 golang.org/x/sync 包学习
categories: golang
date: 2020-02-23 12:38:11
layout: post
---

#### golang.org/x/sync 一共有四个工具包
* singleflight
> Package singleflight provides a duplicate function call suppression mechanism.   
* errgroup
> Package errgroup provides synchronization, error propagation, and Context cancelation for groups of goroutines working on subtasks of a common task.
* semaphore
> Package semaphore provides a weighted semaphore implementation.
* syncmap
> Package syncmap provides a concurrent map implementation.

#### singleflight: 合并请求

> 1. 10个协程同时去查数据库，且查询的数据是一样的吗, 每次查询耗时很长，singleflight包可以屏蔽9次查询，保证只有一次读db；
> 2. 缓存失效场景，在那一刻会有大量请求穿透到db，singleflight只有一次读db

##### 使用示例：
``` GO
package main

import (
	"golang.org/x/sync/singleflight"
	"log"
	"sync"
	"sync/atomic"
	"time"
)

func main() {
	var g singleflight.Group
	var calls int32

	c := make(chan string)
	fn := func() (interface{}, error) {
		atomic.AddInt32(&calls, 1)
		return <-c, nil
	}

	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func() {
			v, err, _ := g.Do("key", fn)
			log.Println(v, err)
			wg.Done()
		}()
	}

	time.Sleep(100 * time.Millisecond)
	c <- "bar"
	wg.Wait()
	got := atomic.LoadInt32(&calls)
	log.Println(got)  // 输出 1
}

```

##### 核心原理：
 ``` GO
type call struct {
	wg  sync.WaitGroup
	val interface{}
	err error
}

type Group struct {
	mu sync.Mutex       // protects m
	m  map[string]*call // 每次查询的数据生成一个key
}

func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
        // 被屏蔽掉了
		g.mu.Unlock()
		c.wg.Wait()
		return c.val, c.err
	}
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	c.val, c.err = fn()  // 只有一次真正的执行
	c.wg.Done()

	g.mu.Lock()
	delete(g.m, key)
	g.mu.Unlock()

	return c.val, c.err
}
 ```
