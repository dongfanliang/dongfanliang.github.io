---
title: flink 之 watermark
categories: flink
date: 2020-02-27 20:30:00
layout: post
---

#### Flink有windown机制，何时触发窗口开始计算呢？    
先说下结论(必须同时满足)
> 1. 窗口有数据
> 2. watermark >= 窗口结束时间

### 一. 串行情况下的watermark
##### 测试1:
- setParallelism(1);      
- window=10s；flink会自动创建[0, 10] ... [50, 60]这些窗口，数据会自动进入对应的窗口   
- 脚本每10s push一条数据     

eventtime | currentMaxTimestamp |  watermark  |  备注
-|-|-
11:36:42 | 11:36:42 | 11:36:32 | 
11:36:52 | 11:36:52 | 11:36:42 | 
11:37:02 | 11:37:02 | 11:36:52 | 触发窗口(11:36:42)
11:37:12 | 11:37:12 | 11:37:02 | 触发窗口(11:36:52)
11:37:22 | 11:37:22 | 11:37:12 | 触发窗口(11:37:02)
11:37:32 | 11:37:32 | 11:37:22 | 触发窗口(11:37:12)


### 二. 并行情况下的watermark
![并行视图的watermark](/assets/img/flink/watermark-in-paralle-stream.png)

如上图各个标志的说明：
- 算子实例右上角的黄色框数字表示算子实例的 low watermark
- 数据管道末端的黄色框数字表示该数据管道的 low watermark    
- 数据管道中的白色框表示 "flag\|eventtime" 形式的数据元素
- 数据管道中的虚线段表示watermark元素
- 在map算子后面接着一个keyBy操作，因此下游的 window 算子的实例会接受上游多个输入数据流。     

从图上看一共有两个source在同时写入数据，Source(1)的最新的watermark为33，Source(2)的最新的watermark为17；map(1)为29，map(2)为17；说明source(1)的数据来的快；此时map(1)map(2)会向下有算子传递自己的watermark，分别是29和17；此时的window都为14，下一时刻window会都设置为17；      

有几个点需要明确：
- 所有的window的watermark都一样
- 向下游传递时会取最小的时间戳为window的watermark, 所以map(2)的17最终会传给window，作为所有window的watermark
- 极端情况下，source2一直都没有新数据，window的watermark一直都不会更新，一直都是17，此时就不会有窗口触发，内存累加，缓存数据，最后OOM；要保证每个source都有新数据进来；
- 同一个key会落入下游同一个slot里

##### 测试2：
- setParallelism(2);    
- window=10s；   
- 脚本每10s push一条数据     

ThreadId | eventtime | currentMaxTimestamp | source-watermark | window-watermark | 备注
-|-|-|-
69 | 12:02:25 | 12:02:25 | 12:02:15 | 12:02:15 | 
70 | 12:02:35 | 12:02:35 | 12:02:25 | 12:02:15 | 
69 | 12:02:45 | 12:02:45 | 12:02:35 | 12:02:25 |
69 | 12:02:55 | 12:02:55 | 12:02:45 | 12:02:25 |
70 | 12:03:05 | 12:03:05 | 12:02:55 | 12:02:45 | 触发窗口(12:02:25, 12:02:35)
69 | 12:03:15 | 12:03:15 | 12:03:05 | 12:02:55 | 触发窗口(12:02:45)
70 | 12:03:25 | 12:03:25 | 12:03:15 | 12:03:05 | 触发窗口(12:02:55）
70 | 12:03:35 | 12:03:35 | 12:03:25 | 12:03:05 | 
70 | 12:03:45 | 12:03:45 | 12:03:35 | 12:03:05 |
69 | 12:03:55 | 12:03:55 | 12:03:45 | 12:03:35 | 触发窗口(12:03:05, 12:03:15, 12:03:25); 

注： 多窗口触发时顺序是按照时间顺序触发的

#### 测试代码如下：
``` java
// 指定watermark 为延后10s
class BoundedOutOfOrdernessGenerator extends AssignerWithPeriodicWatermarks[FlinkItem] {
  val maxOutOfOrderness = 10000L

  var currentMaxTimestamp: Long = 0
  import java.text.SimpleDateFormat
  val sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS")

  override def extractTimestamp(record: FlinkItem, previousElementTimestamp: Long): Long = {
    val currentTimestamp = record.timestamp * 1000
    currentMaxTimestamp = Math.max(currentTimestamp, currentMaxTimestamp)
	val id = Thread.currentThread.getId

	println("extractTimestamp=======>" + ",currentThreadId:" + id + ",key:" + record.counter + ",eventtime:[" + record.timestamp + "|" + sdf.format(record.timestamp*1000) + "]," +
		    "currentMaxTimestamp:[" + currentMaxTimestamp + "|" +
			sdf.format(currentMaxTimestamp) + "],watermark:[" + getCurrentWatermark().getTimestamp() + "|" + sdf.format(getCurrentWatermark().getTimestamp()) + "]")
    currentTimestamp
  }

  override def getCurrentWatermark = new Watermark(currentMaxTimestamp - maxOutOfOrderness)
}
```

参考文章：
[Flink Event Time 倾斜](http://www.whitewood.me/2018/10/28/Flink-Event-Time-%E5%80%BE%E6%96%9C/)      
[Flink中的Watermark](http://www.liaojiayi.com/flink-watermark/)      
[WaterMark分布式执行理解](https://blog.csdn.net/u013560925/article/details/82499612)     


<style>
table th:first-of-type {
    width: 11%;
}
table th:nth-of-type(2) {
    width: 15%;
}
table th:nth-of-type(3) {
    width: 15%;
}
table th:nth-of-type(4) {
    width: 15%;
}
table th:nth-of-type(5) {
    width: 15%;
}
</style>


