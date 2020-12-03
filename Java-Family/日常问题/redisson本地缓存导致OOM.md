## 背景

凌晨3点发布了新版本，经测试和持续两个小时的监控发现一切正常，然后等第二天上午9点钟收到电话，说服务器宕机了，很多用户反馈无法正常访问。紧急起床，然后查看服务器的日志，发现大量的`Exception in thread "RMI TCP Connection(idle)" java.lang.OutOfMemoryError: GC overhead limit exceeded`，然后紧急的`jmap -dump`，然后重启服务。但是重启后过十几分钟后开始一台台的宕机。

## 解决过程

### 通过MAT分析

主界面已经很清晰的显示可能引起内容泄漏的地方，有几千个`org.redisson.PubSubMessageListener`实例，然后到支配树（`dominator_tree`）查看这个对象的GC ROOT，发现被JUC的`ConcurrentLinkedQueue`引用。

![image-20200509001201100](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200509001201100.png)

![image-20200509001631355](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200509001631355.png)

### 代码排查

发现`org.redisson.PubSubMessageListener`是在Redisson中用来做事件订阅的，但正常来说不可能这么的多。然后和项目组团队沟通了一下，本次上线了网关侧的Redisson本地缓存，其他Redisson相关的改动不多，且是常规用法。接下来看代码发现`redissonClient.getLocalCachedMap()`内部是每次new一个cacheMap，且需要订阅通知事件。

![image-20200509002000346](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200509002000346.png)

![image-20200509002040006](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200509002040006.png)

![image-20200509002014221](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200509002014221.png)