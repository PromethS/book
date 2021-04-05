# [Druid 连接池必配项](https://doc.qima-inc.com/pages/viewpage.action?pageId=274352806#druid-连接池必配项)

参数详解： [数据库连接池配置（案例及排查指南）](https://mp.weixin.qq.com/s/_36Ki42raKWAiCrNvOCHLw)

## 连接池配置

### 如何设置连接池大小

合适的连接池大小与请求的 qps 和 rt 相关。rt 的单位为秒。

注意：此处 qps 和 rt 为单个应用端统计。假定随连接数量增加，客户端能处理的请求数线性增加。

基本公式：需要连接数 = qps * rt + 1

统计平时的最大 qps 和此时的 rt，以此计算 minIdle，并设置 initialSize = minIdle。

统计峰值时的 qps 和此时的 rt，以此计算 maxActive。

对于分片的数据库，需要考虑分片数和数据分布情况。如果要调大 maxActive，务必联系 DBA，以免超过服务端的连接数限制。

可以通过以下方法，通过 jmx 观察 Druid 实际的连接池状况，重点关注 ActiveCount：活动连接数，PoolingCount：池子中的连接数。并根据实际情况考虑调整。

```
java -XX:+UseConcMarkSweepGC -XX:+CMSClassUnloadingEnabled  -XX:+TieredCompilation -XX:TieredStopAtLevel=1  -Xverify:none -client -jar /opt/zabbix/share/zabbix/yzscripts/cmdline-jmxclient-0.10.3.jar - 127.0.0.1:7777 'com.alibaba.druid:type=DruidDataSourceStat' DataSourceList |& grep -E 'ActiveCount|PoolingCount'
```

#### 查看实际连接数

```
# grep DB 的地址或者port 来查看连接数，有效连接数看连接状态为ESTABLISHED 的连接
netstat -na | grep $dbport
# 查看有效连接
netstat -na | grep $dbport | grep ESTABLISHED
```

### 如何设置超时时间

连接池中的超时时间主要有：

- connectTimeout 建立 TCP 连接的超时时间
- maxWait 从连接池获取连接的最长等待时间
- socketTimeout 发送请求后等待响应的超时时间

其中，connectTimeout 建议不要小于 1200ms。TCP 在建立连接时，SYN 包的**超时重传**时间为 1s。connectTimeout 设置过短，很可能造成应用发布时，初始化连接池过程中由于网络抖动，或中间网络设备需要初始化状态发生丢包触发超时，从而造成连接池初始化失败而导致发布失败。之前由于 UCloud 的问题，需要设置到 5s 已保证新发布的机器能够在网络配置初始完成之前，不会触发超时，现在已无此必要。

**maxWait** 可以根据应用期待的等待时间设置。为避免在发生网络问题，或数据库服务有问题时雪崩，这个时间设置不要过大。下面的默认值 800ms 是个保守的设置。应用可以设置一个更短的时间，如 300ms。过短的时间也会造成在连接池中连接数不足，需要新建连接时造成大量超时。建议不要低于 100ms。

**socketTimeout** 可以根据应用最长的查询返回时间设置。过长会造成生网络问题，或数据库服务有问题时雪崩；过短也会造成频繁请求超时。不要短于 300ms。TCP 的最小 RTO 为 200ms，并根据延迟动态调整。过短的超时时间会造成单个丢包就造成请求超时。生产环境数据库都配置有 SQL Killer，会自动杀死执行时间过长的请求。因此，设置过长的 socketTimeout 也是没有意义的。

### 如何设置连接保持时间

应用到 RDS 的访问路径为 App -> LVS -> RDS Proxy 。

其中，LVS 空闲连接保留时间为 90 s。RDS Proxy 为了避免访问到已被关闭的连接，自身的空闲连接保留时间为 [70, 85) s。

因此，应用程序为了避免从连接池获取到已被关闭的连接，应当设置自身保留空闲连接时间不能超过 70s。

因此有 `timeBetweenEvictionRunsMillis=10000`, `minEvictableIdleTimeMillis=44000`, `maxEvictableIdleTimeMillis=55000`。

## 必选配置项

以下默认配置可以根据实际情况调整。调整前请务必阅读本文档，并联系 RDS开发或DBA 确认。

```xml
<bean id="cartDataSource" class="com.alibaba.druid.pool.DruidDataSource"
          init-method="init" destroy-method="close">
        <property name="url" value="${cluster.jdbc.url}"/>
        <property name="username" value="${cluster.jdbc.username}"/>
        <property name="password" value="${cluster.jdbc.password}"/>
        <property name="connectionInitSqls" value="set names utf8mb4"/>
        <!-- 连接池初始连接数 -->
        <property name="initialSize" value="5" />
        <!-- 允许的最大同时使用中(在被业务线程持有，还没有归还给druid) 的连接数 -->
        <property name="maxActive" value="20" />
        <!-- 允许的最小空闲连接数，空闲连接超时踢除过程会最少保留的连接数 -->
        <property name="minIdle" value="5" />
        <!-- 从连接池获取连接的最大等待时间 800毫秒;业务方根据可以自行调整-->
        <property name="maxWait" value="800" />
        <!-- 一条物理连接的最大存活时间 120分钟-->
        <property name="phyTimeoutMillis" value="7200000"/>
        <!-- 强行关闭从连接池获取而长时间未归还给druid的连接(认为异常连接）-->
        <property name="removeAbandoned" value="true"/>
        <!-- 异常连接判断条件，超过180 秒 则认为是异常的，需要强行关闭 -->
        <property name="removeAbandonedTimeout" value="180"/>
        <!-- 从连接池获取到连接后，如果超过被空闲剔除周期，是否做一次连接有效性检查 -->
        <property name="testWhileIdle" value="true"/>
        <!-- 从连接池获取连接后，是否马上执行一次检查 -->
        <property name="testOnBorrow" value="false"/>
        <!-- 归还连接到连接池时是否马上做一次检查 -->
        <property name="testOnReturn" value="false"/>
        <!-- 连接有效性检查的SQL -->
        <property name="validationQuery" value="SELECT 1"/>
        <!-- 连接有效性检查的超时时间 1 秒 -->
        <property name="validationQueryTimeout" value="1"/>
        <!-- 周期性剔除长时间呆在池子里未被使用的空闲连接, 10秒一次-->
        <property name="timeBetweenEvictionRunsMillis" value="10000"/>
        <!-- 空闲多久可以认为是空闲太长而需要剔除 44 秒-->
        <property name="minEvictableIdleTimeMillis" value="44000"/>
        <!-- 如果空闲时间太长即使连接池所剩连接 < minIdle 也会被剔除 55 秒 -->
        <property name="maxEvictableIdleTimeMillis" value="55000"/>
        <!-- 是否设置自动提交，相当于每个语句一个事务 -->
        <property name="defaultAutoCommit" value="true"/>
        <!-- 记录被判定为异常的连接 -->
        <property name="logAbandoned" value="true"/>
        <!-- 开启连接保活，保活连接数为minIdle，如果没有突发流量场景，可以不用打开
             在1.1.20 版本中需要配置keepAliveBetweenTimeMillis
        -->
        <property name="keepAlive" value="true" />
        <property name="keepAliveBetweenTimeMillis" value="44000" />
        <!-- 使用非公平锁，建议开启，能大幅提升性能，影响就是不保证公平性 -->
        <property name="useUnfairLock" value="true" />
        <!-- 网络读取超时，网络连接超时
             socketTimeout : 对于线上业务小于5s，对于BI等执行时间较长的业务的SQL，需要设置大一点
        -->
        <property name="connectionProperties"
                  value="socketTimeout=3000;connectTimeout=1200"/>
        <property name="proxyFilters">
            <list>
                <ref bean="log-filter"/>
            </list>
        </property>
</bean>
```

1.0.28版本之后，新加入keepAlive配置，缺省关闭。使用keepAlive功能，建议使用1.1.16或者更高版本。一般业务无需打开，除非分钟请求量在个位数或者启动时间超长导致初始连接都过期，开启前请到RDS服务群咨询。

#### 使用非公平锁模式优化性能

设置useUnfairLock 为true，开始非公平锁模式，因为druid 在设置maxWait 后会将锁设置为公平模式，但测试发现公平模式锁的性能损耗十分大，改为非公平锁模式后实测tsp 整体吞吐量提升50%以上。使用jstack可以判断是否使用了公平锁，下图表示使用了公平锁：

判断是否开启了公平锁可以在arthas 中运行：（输出true 表示使用非公平锁; 1.1.22及之前版本，这个值要取反）

```
watch com.alibaba.druid.pool.DruidAbstractDataSource getConnection '{target.isUseUnfairLock()}' -n 2
```

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210327165949.(null))

下图是排查时发现公平锁时，回收连接性能非常低：

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210327165943)

#### 打开KeepAlive之后的效果

- 初始化连接池时会填充到minIdle数量。
- 连接池中的minIdle数量以内的连接，空闲时间超过minEvictableIdleTimeMillis，则会执行keepAlive操作。
- 当网络断开等原因产生的由ExceptionSorter检测出来的死连接被清除后，自动补充连接到minIdle数量。

##### keepAlive 在 druid 1.1.20 特殊配置

在1.1.16 之后的版本中新增了变量 **keepAliveBetweenTimeMillis** 默认是120000，表示在连接超过120s 后才对连接进行保活检查。因此如果使用的druid 1.1.20 版本，只开启KeepAlive，假设minIdle=10，未配置keepAliveBetweenTimeMillis 会看到一种现象是有时候连接是10个，有时候这10个都处于TIME_WAIT 状态，过一会儿又新建出10个新的连接，这是由于druid 内部的逻辑调整：即使keep alive 保活过，但是空闲超过**maxEvictableIdleTimeMillis** 的连接还是会被关闭，因此对于新版按如下调整配置性能更好（注意版本一定要大于1.1.16），调整字段 maxEvictableIdleTimeMillis：

```xml
<bean id="cartDataSource" class="com.alibaba.druid.pool.DruidDataSource"
          init-method="init" destroy-method="close">
        <!-- 其它配置略，参考上文 -->
  
        <!-- 周期性剔除长时间呆在池子里未被使用的空闲连接, 10秒一次-->
        <property name="timeBetweenEvictionRunsMillis" value="10000"/>
        <!-- 空闲多久可以认为是空闲太长而需要剔除 44 秒-->
        <property name="minEvictableIdleTimeMillis" value="44000"/>
        <!-- 如果空闲时间太长即使连接池所剩连接 < minIdle 也会被剔除 20 分钟 -->
        <property name="maxEvictableIdleTimeMillis" value="1200000"/>
        <!-- 开启连接保活，保活连接数为minIdle，如果没有突发流量场景，可以不用打开
             在1.1.20 版本中需要配置keepAliveBetweenTimeMillis
        -->
        <property name="keepAlive" value="true" />
        <!-- 配置成小于等于 minEvictableIdleTimeMillis -->
        <property name="keepAliveBetweenTimeMillis" value="44000" />
</bean>
```

注意：**如果只开启keep alive 会造成频繁的创建、销毁连接，长久的运行后（client端）会有一定的GC负担；如果无突发流量情况（毛刺比较少），可以关闭keepAlive**