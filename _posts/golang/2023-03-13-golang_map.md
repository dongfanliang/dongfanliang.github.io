---
title: golang 之 map
date:  2023-03-13 15:45:52 +0800
category: golang
tags: golang
excerpt:
---

## 源码说明
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

## 底层原理
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

## 参考资料
- [map 的实现原理](https://golang.design/go-questions/map/principal/)




