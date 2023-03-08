# 热key统计算法-滑动窗口

> 官方说是用的滑动窗口，看完实现后发现其实是用的滑动窗口的思想，不是LeeCode上的题目那样的典型的滑动窗口实现(双指针)。

统计热key的方法就是统计“最后一次访问（或者当前的访问）”key时截至，往前一段时间内key访问的次数。

jd-hotkey将这段时间定义为duration，划分了几个（windowSize）分片，每个分片记录 duration/widowSize 时间段中访问key的次数。然后判断key是否是热key就是找到当前时间分片再加上之前的分片（总共windowSize个分片，duration时间段）统计总的访问次数是否达到设置的阈值（threshold）。

duration <= 5s : windowSize = 5; duration > 5s (超过600s按600s算，即统计窗口最大统计10分钟内的访问数据)：windowSize = 10;

存储时间片的 AtomicLong[] timeSlices 数组当作循环队列使用。

当前时间对应的时间片索引:  `idx = (currentTimestamp - beginTimestamp) / timeMillisPerSlice % timeSliceSize`, 理解了这行代码就理解jd-hotkey的滑动窗口是怎么实现的了，其实很简单。

数据结构：

```java
// 滑动窗口共有多少个时间片
private final int windowSize;
// 滑动窗口每个时间片的时长，以毫秒为单位, 滑动窗口总时长 duration = windowSize * timeMillisPerSlice
private final int timeMillisPerSlice;
// 在一个完整窗口期内判定热key的最大阈值
private final int threshold;
// 该滑窗的起始创建时间，也就是第一次访问时的时间戳
private long beginTimestamp;
// 最后一次访问（上一次访问）的时间戳
private long lastAddTimestamp;
// 循环队列，装多个窗口时间片用，该数量是windowSize的2倍
// 这里不明白为何要设置windowSize的两倍，按理说windowSize个就够用了，预留拓展的么？
private AtomicLong[] timeSlices;
// 队列的总长度
private final int timeSliceSize;
```

