---
title: golang 之 map 和 sync.Map
date:  2023-03-13 15:45:52 +0800
category: golang
tags: golang
excerpt:
---

## map
### 源码分析
``` GO
// golang 1.19 src/runtime/map.go
type hmap struct {
    count     int  // map元素数量
    flags     uint8  // 当前map状态（同时读写会panic） 
    B         uint8  // map中桶的数量2^B个
    noverflow uint16 // 溢出桶数量，太多的话就会等内存扩容
    hash0     uint32 // hash seed

    buckets    unsafe.Pointer // 对应桶的指针，array of 2^B Buckets. may be nil if count==0.
    oldbuckets unsafe.Pointer // map扩容存储旧桶，所有旧桶迁移完成后位nil
    nevacuate  uintptr        // 扩容时，用于标记当前旧桶中小于nevacuate的数据都迁移到了新桶

    extra *mapextra // 溢出桶
}

// 桶的结构体
type bmap struct {
    tophash [bucketCnt]uint8  // bucketCnt=8

    // 编译时会添加以下字段，存储key和value
    keys     [bucketCnt]keytype  
    values   [bucketCnt]valuetype
    pad      uintptr
    overflow uintptr  // 溢出桶链表
}
```
### 查找
![map-1](/assets/img/golang/map-1.png)
假定 B = 5，所以 bucket 总数就是 2^5 = 32，计算出待查找 key 的哈希值
1. 找桶：使用哈希值低5(B)位 00110，找到对应的 6 号 bucket  
2. 找key/value：使用哈希值高8位10010111对应十进制151，遍历6号桶查找哈希值为151，找到了2号槽位，通过offest找到出key，value       

如果在 bucket 中没找到，并且 overflow 不为空，还要继续去 overflow bucket 中寻找，直到找到或是所有的 key 槽位都找遍了，包括所有的 overflow bucket
> 注：如果key未在map中，则返回value的0值
``` GO
func mapaccess1_faststr(t *maptype, h *hmap, ky string) unsafe.Pointer {
    ...
    ...
    return unsafe.Pointer(&zeroVal[0])
}
```

### 扩容
当发生以下两种情况之一时，map会进行重建：
#### 一. map超过了负载因子
各个桶要满了，源码中负载因子定义为6.5

> 负载因子 = 元素数量 / 桶的数量

此时新申请原来2倍(B+1)的桶容量，但不会马上迁移，新桶赋值给buckets，旧桶赋值给oldbuckets
查找算法不变，不过B+1需要多看一位
![map-2](/assets/img/golang/map-2.png)

假定B=5扩容后B=6，如上图原6号(00110)桶就分裂为6号(<span style="border-bottom: 2px dashed red;">0</span>00110)和38号(<span style="border-bottom: 2px dashed red;">1</span>00110)桶，如果原6号桶的数据哈希值第6二进制是1就需要搬迁到38号桶

> map删除、修改、新增的时候会触发数搬迁

#### 二. 溢出桶的数量过多
溢出桶的数量过多, map会退化成链表，影响性能；扩容会申请等数量的桶，然后把溢出桶数据搬迁到桶中
- 扩容前
![map-3](/assets/img/golang/map-3.png)
- 扩容后
![map-4](/assets/img/golang/map-4.png)

#### 删除
同样的查找算法，清空key和value； 将 count 值减 1，将对应位置的 tophash 值置成 Empty

## sync.Map
原生map不支持并发读写，解决方案一般有2种套路：
1. 加一把大锁(互斥锁、读写锁)，控制整个map的读写
2. 使用sync.Map, 但这种方式也不是万能的

### 源码分析
``` Go
// golang 1.19 src/sync/map.go
type Map struct {
    mu Mutex           // 保护dirty
    read atomic.Value  // 只读数据存储readOnly，是个读缓存且原子操作
    dirty map[any]*entry
    misses int         // 读dirty则加1，记录缓存穿透的次数
}

type readOnly struct {
    m  map[any]*entry  // 只读数据
    // true，表示dirty多出了一些独有key，通过该字段决定是否加锁读dirty
    amended bool       
}

type entry struct {
    p unsafe.Pointer  // 数据存储
}
```

### 读流程
``` Go
func (m *Map) Load(key any) (value any, ok bool) {
    // 1. 先读read，原子操作取出
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]

    // 2. 如果read中没有，amended为true表示dirty和read数据不一致，需要找dirty数据
    if !ok && read.amended {  // Check，常规并发检查套路CLC
        // dirty数据读写都需要上锁
        m.mu.Lock() // Lock
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {  // Check
            e, ok = m.dirty[key]
            // 记录穿透次数
            m.missLocked()
        }
        m.mu.Unlock()
    }

    // 3. read和dirty都没有，直接返回
    if !ok {
        return nil, false
    }
    return e.load()
}

func (m *Map) missLocked() {
    // 计数+1
    m.misses++
    if m.misses < len(m.dirty) {
        return
    }
    // 缓存穿透次数太多，使用dirty重建read
    m.read.Store(readOnly{m: m.dirty})  // amended初始化为false
    m.dirty = nil  // 新值入会重建，延迟重建了
    m.misses = 0
}

func (e *entry) load() (value interface{}, ok bool) {
    // 取出p指针对应的值
    p := atomic.LoadPointer(&e.p)
    if p == nil || p == expunged {
        return nil, false
    }
    return *(*interface{})(p), true
}
```

### 写流程
``` Go
/* 
    entry.p有3个状态：
        1. p == nil，被标记删除，read和dirty共同存在
        2. p == expunged, 被标记删除，只存在read中(dirty中没有)
        3. e.p == 普通指针，在read中，如果dirty!=nil，也在dirty中
*/
func (m *Map) Store(key, value any) {
    // 1. 如果read里存在，尝试更新entry.p的指针
    read, _ := m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }

    // 2. 如果read里面不存在key或者是对应的key是被标记为expunged
    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok {
        // 情况1. read存在，但是被标记为expunged，说明dirty里边没有需要写进去
        if e.unexpungeLocked() {
            m.dirty[key] = e
        }
        e.storeLocked(&value)  // 更新entry的p指针
    } else if e, ok := m.dirty[key]; ok {
        // 情况2. read不存在，dirty存在，直接更新entry的p指针
        e.storeLocked(&value)
    } else {
        // 情况3. 新值，read和dirty都不存在
        // read.amended == false，表示dirty中没有多出来的key或两者均是初始化状态
        if !read.amended {
            // 保证dirty初始化，用read重建dirty
            m.dirtyLocked()
            // 设置amended=true，dirty多出了一些key，read里边没有新增
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        // 新增dirty
        m.dirty[key] = newEntry(value)
    }
    m.mu.Unlock()
}

func (m *Map) dirtyLocked() {
    if m.dirty != nil {
        return
    }
    // 一顿操作:
    // 1. dirty里存储了全部的非expunged的entry
    // 2. read里面存储了dirty的全集，以及所有expunged的entry
    // 3. 且read中不存在e.p == nil的entry，因为已经被转成了expunged
    read, _ := m.read.Load().(readOnly)
    m.dirty = make(map[any]*entry, len(read.m))
    for k, e := range read.m {
        if !e.tryExpungeLocked() {
            m.dirty[k] = e
        }
    }
}

func (e *entry) tryStore(i *any) bool {
    for {
        p := atomic.LoadPointer(&e.p)
        if p == expunged {
            return false
        }
        if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
            return true
        }
    }
}

func (e *entry) tryExpungeLocked() (isExpunged bool) {
    p := atomic.LoadPointer(&e.p)
    // 当p==nil时，改成expunged，表示已删除
    for p == nil {
        if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
            return true
        }
        p = atomic.LoadPointer(&e.p)
    }
    return p == expunged
}
```

### 删除流程
``` Go
func (m *Map) Delete(key any) {
    m.LoadAndDelete(key)
}

func (m *Map) LoadAndDelete(key any) (value any, loaded bool) {
    // 整体流程和Load差不多
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    if !ok && read.amended {
        m.mu.Lock()
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
            e, ok = m.dirty[key]

            // dirty的key是被删除掉了
            delete(m.dirty, key)
            m.missLocked()
        }
        m.mu.Unlock()
    }

    // 找到key，执行删除
    if ok {
        return e.delete()
    }
    return nil, false
}

func (e *entry) delete() (value any, ok bool) {
    for {
        p := atomic.LoadPointer(&e.p)
        if p == nil || p == expunged {
            return nil, false
        }

        // read并没有真正删除，只是把p置成nil
        if atomic.CompareAndSwapPointer(&e.p, p, nil) {
            return *(*any)(p), true
        }
    }
}
```

### read和dirty会相互转换
1. dirty->read, read miss过多就会触发重建(全量覆盖)，此时dirty设置为nil等待新值写入重建（延迟重建）
2. read->dirty, 新值写入时如果dirty为nil，会用read中非expunged初始化dirty，此时read中的nil标记为expunged，下次重建read时这些就会被删除

### 性能分析
- 读性能会很好，因为有read做cache且是原子操作，而且miss过多触发read重建时也是原子操作(全量覆盖)
- 写性能会比较差，有一把大锁而且当dirty为nil时，还会触发read到dirty的转换，当数据量大时会影响性能

### 压测报告
压测代码：[benchmark-for-map](https://github.com/bigwhite/experiments/blob/master/go19-examples/benchmark-for-map/map_benchmark.go)

``` GO
// go版本：go1.19
// 机器配置：8C16G
~ go test -bench .
goos: linux
goarch: amd64
// 写，RWMutex > Mutex >> sync.Map
BenchmarkBuiltinMapStoreParalell-8      	10180335	       186.0 ns/op
BenchmarkBuiltinRwMapStoreParalell-8    	 9906567	       163.3 ns/op
BenchmarkSyncMapStoreParalell-8         	 4137398	       425.9 ns/op

// 读，sync.Map > RWMutex >> Mutex
BenchmarkBuiltinMapLookupParalell-8     	 7819263	       167.5 ns/op
BenchmarkBuiltinRwMapLookupParalell-8   	35695894	        37.91 ns/op
BenchmarkSyncMapLookupParalell-8        	55963688	        21.40 ns/op

// 删除，sync.Map >> Mutex > RWMutex
BenchmarkBuiltinMapDeleteParalell-8     	 7765178	       169.3 ns/op
BenchmarkBuiltinRwMapDeleteParalell-8   	 7622546	       187.3 ns/op
BenchmarkSyncMapDeleteParalell-8        	52537059	        21.68 ns/op
PASS
```

### 适用场景
读多写少场景

## 参考资料
- [map 的实现原理](https://golang.design/go-questions/map/principal/)

