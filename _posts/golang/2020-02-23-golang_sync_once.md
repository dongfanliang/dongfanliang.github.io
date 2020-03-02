---
title: golang 之 sync.Once的用法
categories: golang
date: 2020-02-23 12:38:11
layout: post
tags: golang
---

### sync.Once.Do(f func()) 全局只执行一次
> 无论你是否更换once.Do(xx)这里的方法, 这个sync.Once块只会执行一次;

``` GO
package main

import (
	"fmt"
	"sync"
)

var once sync.Once

func main() {

	for i, v := range make([]string, 10) {
		once.Do(onces)  // 只会执行一次
		fmt.Println(i)
	}
}
func onces() {
	fmt.Println("onces")
}
```

### 源码如下：
``` GO
// Once is an object that will perform exactly one action.
type Once struct {
    m    Mutex
    done uint32
}

// Do calls the function f if and only if Do is being called for the
// first time for this instance of Once. In other words, given
//  var once Once
// if once.Do(f) is called multiple times, only the first call will invoke f,
// even if f has a different value in each invocation. A new instance of
// Once is required for each function to execute.
//
// Do is intended for initialization that must be run exactly once. Since f
// is niladic, it may be necessary to use a function literal to capture the
// arguments to a function to be invoked by Do:
//  config.once.Do(func() { config.init(filename) })
//
// Because no call to Do returns until the one call to f returns, if f causes
// Do to be called, it will deadlock.
//
// If f panics, Do considers it to have returned; future calls of Do return
// without calling f.
//
func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 1 {
        return
    }
    // Slow-path.
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
        defer atomic.StoreUint32(&o.done, 1)
        f()
    }
}
```