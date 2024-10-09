---
title: 分布式 ID
date: 2023/3/21 7:00:00
comments: true
no_word_count: true 
reward: true 
copyright: true 
categories: 
- 分布式系统
tags:
- 分布式系统
typora-root-url: ../../dongyifeng.github.io
index_img: /images/index_img/distributed_id.jpg
---



# 背景

在复杂分布式系统中，往往需要对大量的数据和消息进行唯一标识。随着数据日渐增长，对数据分库分表后也需要有一个唯一ID来标识一条数据或消息，数据库的自增 ID 显然不能满足需求；此时一个能够生成全局唯一 ID 的系统是非常必要的。



**分布式 ID 需要满足以下条件**

- 全局唯一：不能出现重复的 ID 号，既然是唯一标识，这是最基本的要求。
- 高性能：高可用低延时，ID 生成速度要快，否则反倒会成为业务瓶颈；
- 高可用：尽量保证服务的可用性，多实例化，避免因一个实例挂掉影响整个业务应用的运行。
- 容易接入：要秉着拿来即用的设计原则，在系统设计和实现上要尽可能的简单，避免增加开发人员的使用成本。
- 趋势递增： 最好趋势递增，这样方便进行数据排序、过滤，当然这个要求还需要根据具体的业务场景作出安排。
- 信息安全：如果 ID 是连续递增的，恶意用户就可以很容易的推测出订单号的规则，从而猜出下一个订单号，如果是竞争对手，就可以直接知道我们一天的订单量。所以在某些场景下，需要 ID 无规则来保证安全性。



# 方案一：UUID

```java
public class UUIDTest {
    public static void main(String[] args) {
        // 生成 UUID
        String uuid = UUID.randomUUID().toString();
        // 输出 UUID 串
        System.out.println(uuid);
    }
}
```



**优点：**

- 高性能
- 实现简单
- 本地生产 ID，不需要依赖第三方组件

**缺点：**

- 不是趋势递增，不方便排序
- 生成的 ID 只能用字符串类型存储，占用空间大；



# 方案二：基于数据库单机自增 ID

数据库自增 ID 是依赖数据库中提供的自动增量功能实现，这种生成 ID 的方案比较容易实现与使用。在这种方案中，为了存储生成的 ID 值，往往我们会单独创建立一张专用于存储生成 ID 的表，然后往表中插入数据替换旧数据，过程中 ID 会递增，我们只需要查询该递增的 ID 值，然后再与时间戳、随机值等元素进行组合处理，生成分布式 ID。

![](/images/distributed/WX20230206-215037@2x.png)

如下图：比方案一，多一次 Mysql 请求。

![](/images/distributed/WX20230206-215857@2x.png)



用于存储 ID 的表，结构如下

| id   | stub |
| ---- | ---- |
| 1    | a    |
| 2    | b    |

- id：自增生成的 ID 值
- stub：用于记录 ID 是归属的业务

```mysql
CREATE TABLE `myid` (  
  id bigint(20) unsigned NOT NULL auto_increment,    
  stub char(1) NOT NULL default '',    
  PRIMARY KEY (id))
```



每当应用程序需要一个 ID 时，就插入一条数据使其自增 ID，然后替换旧的数据，最后读取新生成的 ID 值：

```mysql
begin;
REPLACE INTO `myid`(stub) VALUES ('a');SELECT LAST_INSERT_ID();
commit;
```



等取到 ID 值后我们在让其与"时间戳"、"随机值"、"业务码"等组合，生成与业务挂钩的分布式 ID 串，一般时候我们生成的串都不会超过 64 位，以方便用 long 类型存储该串；



优点：

1. <font color=green>实现简单</font>
2. <font color=green>生成的 ID：有序递增，数值类型，便于创建索引，查询快。</font>



缺点：

1. <font color=green>性能较差，只能应用于并发量小的业务需求。</font>
2. <font color=green>存在单点问题，如果数据库不可用将导致依赖它的服务不能正常执行。</font>



# 方案三：基于数据库集群模式自增生成 ID

前面讲述了单机数据库方式通过自增方式生成 ID，这种方式由于单机部署，不论是性能还是可用性都无法得到保障。故而往往都不会直接采用该方案，而是对其进行改动，将其改为使用多主的集群模式部署，利用多个数据库来进行自增生成 ID。

使用多台数据库会导致每个数据库的 ID 都是从 1 开始递增，且递增步长为 1，在这种情况下一定会生成重复的 ID 值。解决这种 ID 重复生成的问题也很简单，只需要对每个数据库都提前配置好其<font color=red>初始值（auto_increment_increment）</font>，以数据库个数充当自增长<font color=red>步长（auto_increment_offset）</font>，这样每个库中增长的 ID 就不会重复了。

![](/images/distributed/WX20230206-222110@2x.png)

![](/images/distributed/WX20230207-113640@2x.png)

![](/images/distributed/WX20230206-224515@2x.png)

优点：

- 高可用
- 趋势递增



缺点：

- 性能一般，只能并发量小的业务需求
- 水平扩展比较麻烦，需要手动调整集群数据库中的初始值与步长。



# 方案四：基于数据库的号段模式

号段模式一般也是基于数据库自增实现分布式 ID 的一种方式，是当下分布式 ID 生成方式中比较流行的一种，其使用可以简单理解为每次从数据库中获取生成的 ID 号段范围，将范围数据获取到应用本地后，在范围内循<font color=red>递增生成一批 ID</font>，然后将这批数据存入缓存。



每次应用需要获取 ID 时，这时就候就可以从缓存中读取 ID 数据，当缓存中的 ID 消耗到一定数目时候，这时再去从数据库中读取一个号段范围，再执行生成一批 ID 操作存入缓存，这是一个重复循环的过程，这样重复操作每次都只是从数据库中获取待生成的 ID 号段范围，而不是一次次获取数据库中生成的递增 ID，这样减少对数据库的访问次数，大大提高了 ID 的生成效率。



![](/images/distributed/WX20230207-125626.png)

在使用号码模式时，我们通常会先建立一张表用于记录上述的 ID 号段范围。

```mysql
CREATE TABLE `myid` (
  id int(10) NOT NULL AUTO_INCREMENT,  
  max_id bigint(20) NOT NULL,  
  step int(20) NOT NULL,  
  biz_type int(20) NOT NULL,     
  version int(20) NOT NULL,    
  PRIMARY KEY (`id`)) 
```

- max_id：当前最大可用的 ID。
- step：号段的步长。
- bit_type：业务类型
- version：记录更新的版本号，主要作用是<font color=red>**乐观锁**</font>，每次更新时都会更新该值，以保证并发时数据的正确性。

<font color=red>分布在不同服务上的相同业务对应一条数据库记录，任务执行时去数据区去抢 ID 号段，并更新号段供下个服务使用。不同业务数据独立，可以 ID 重复。</font>

每次从数据库中获取号段 ID 的范围时，都会执行更新语句，其中计算新号段范围最大值 max_id 的公式 max_id = max_id + step 组成。

```mysql
begin;
UPDATE `myid` 
SET `max_id` = max_id + step, 
`version` = version + 1 
WHERE `version` = {执行更新的版本号} AND `biz_type` = {业务类型};
SELECT `max_id`, `step`, `version` FROM `myid` WHERE `biz_type` = {业务类型};
commit;
```



## 过程描述

例如：某个业务需要批量获取 ID，首先它往数据库 myid 中插入一条初始化值，设置 max_id = 0 和步长 step= 1000 及使用该 ID 的业务标识 biz_type = test 与版本 version = 0，如下：

```mysql
INSERT INTO `myid`(
  `max_id`,`step`,`biz_type`,`version`) 
  VALUES(0,1000,"test",0);
```

| id   | max_id                   | step | biz_type | version |
| ---- | ------------------------ | ---- | -------- | ------- |
| 1    | <font color=red>0</font> | 1000 | test     | 0       |

然后以 biz_type 作为筛选条件，从数据库 myid 中读取 max_id 与 step 的值:

- max_id：0
- step：1000

通过这两个值可以知道号段范围为 (0, 1000 ]，生成该批 ID 存入缓存中，那么这是缓存大小为：1000



每次都从缓存中取值，创建一个监听器用于监听缓存中 ID 消耗比例，设置阈值，判断如果取值超过的阈值后就进行数据库号段更新操作，跟上面第一次执行更新时候一样，也是执行下面的更新 SQL 语句。

然后执行获取分布式 ID 的方法，方法中应执行下面语句进行号段更新，方便生成新的一批号段：

```mysql
begin;
UPDATE `myid` 
SET `max_id` = max_id + step, `version` = version + 1 WHERE `version` = 0 AND `biz_type` = test;
SELECT `max_id`, `step`, `version` 
FROM `myid` 
WHERE `biz_type` = test;
commit;
```

这时候数据库中的值为：

| id   | max_id                      | step | biz_type | version |
| ---- | --------------------------- | ---- | -------- | ------- |
| 1    | <font color=red>1000</font> | 1000 | test     | 1       |

比如，设置阈值为 50%，当缓存中存在 1000 个 ID，监听器监听到业务应用已经消耗到 500 个后（超过阈值），创建一个新的线程去执行上面的更新 SQL 语句，让数据库中号段范围按照设置的 step 扩大，然后获取新的号段最大值，应用中再生成一批范围为 (1001,2000] 范围的 ID 存入缓存供应用使用，这时候缓存中数据大小为：2000（已经使用了 500，可用 1500）

过程是个循环的过程，每到消耗到一定数据后就会生成新的一批。这里只是对其进行了简单介绍，很多时候为了保证数据库可用性都会采用集群模式，现在通过号码模式生成 ID 的开源框架有很多，比如：

- 美团开源的 Leaf
- 滴滴开源的 TinyId

**优点**

- 趋势递增
- 使用缓存机制，容灾性高，即使数据库不可用还能撑一段时间
- 可以自定义每次扩展的大小，控制 ID 生成速度
- 可以设置生成 ID 的初始范围，方便业务从原有的 ID 方式上迁移过来。



**缺点**

- 数据库宕机会造成整个系统不可用
- ID 号码不够随机，有可能泄露发号数量的信息，不太安全。

所以，采用这种方案我们也经常使用数据库多主模式，保证数据库的高可用性。



# 方案五：基于 Redis 单节点实现分布式 ID

Redis 中存在原子操作指令 `INCR` 或 `INCRBY`，执行后可用于创建初始化值或者在原有数字基础上增加指定数字，并返回执行 INCR 命令之后 key 的值，这样就可以很方便的创建有序递增的 ID。



**优点**

- 实现简单
- 有序递增，方便排序

**缺点：**

- 强依赖于 redis，可能存在单点问题
- 如果 Redis 超时，可能会对业务操作影响
- <font color=red>Redis 持久化有可能会丢数据</font>。如果丢失数据，有可能出现重复 ID 。



# 方案六：使用 Redis 集群实现分布式 ID

使用 Redis 单机生成 ID 存在性能瓶颈，无法满足高并发的业务需求，且一旦 Redis 崩溃或者服务器宕机，那么将导致整个基于它的服务不可用，这是业务中难以忍受的，所以一般时候会用集群的方式来实现 Redis 的分布式 ID 方案。

<font color=red>此方案的思想与方案三：基于数据库集群模式自增生成 ID 的思想是一致的：通过初始值和步长来控制不同 Redis 的ID 生成，确保不生成重复 ID。</font>



使用集群的方式需要设置提前设置 <font color=red>初始值 和 步长</font> 来保证每个节点增加的 ID 不会冲突，正常做法每个节点都配置一个跟节点挂钩的 <font color=red>Lua 脚本</font>，脚本内容中设置好对应节点的 初始值 和 步长，其中初始值是按照节点个数从 1 开始递增分配，而步长则是等于集群中 Master 节点的个数。按照这种方式生成 ID 并获取后，后面的执行逻辑跟单节点 Redis 一样，都是对 ID 进行加工处理操作。

![](/images/distributed/WX20230207-163117@2x.png)



**优点**

- 集群模式高可用
- 趋势递增，方便分类、排序

**缺点：**

- 如果 Redis 超时，可能会对业务操作影响
- 存在网络开销，集群模式需要数据同步，对性能有影响。
- 集群规模固定后，改动规则影响很大，所以扩展比较困难。
- <font color=red>Redis 持久化有可能会丢数据</font>。如果丢失数据，有可能出现重复 ID 。



# 方案七：基于雪花算法模式

今天的主角雪花算法，它是 Twitter 开源的由 64 位整数组成分布式 ID，性能较高，并且在<font color=red>单机上递增</font>。



如下图：

1. **第一位** 占用1bit，由于 long 基本类型在 Java 中是带符号的，整数为 0 负数为 1，一般生成的 ID 都为正数，所以固定为0;
2. **时间戳** 占用41bit，时间戳不是存储当前时间的时间戳，而是存储时间的差值（当前时间-固定的开始时间），这里的的开始时间戳为我们的ID生成器开始使用的时间。通过计算（1L << 41) / (1000L * 60 * 60 * 24 * 365）得出69，总共可以容纳约69年的时间。
3. **工作机器id** 占用10bit，其中高位5bit是数据中心ID，低位5bit是工作节点ID，做多可以容纳1024个节点。 
4. **序列号** 占用12bit，在同一机器同一毫秒内可生成不同的序列号，12 位支持最多能生成 4096 个。

SnowFlake 算法在同一毫秒内最多可以生成多少个全局唯一ID呢：： **同一毫秒的ID数量 = 1024 X 4096 = 4194304**



![](/images/distributed/1572487293728.avif)



雪花算法效率很高，理论上其生成 ID 的 QPS 约为 409.6w/s，这种分配方式可以保证在任何一个机房的任何一台机器在任意毫秒内生成的 ID 都是不同的。



## Snowflake 算法的扩展位

在实际使用过程中，我们往往都会根据具体的业务对雪花算法的组成进行改动，常改动的是 10bit 的 WorkerID 位置，该位置由5位数据中心标识与5位机器标识共同组成，那么这时候可以：

- <font color=red>如果部署的服务都在同一个数据中心，即不考虑数据中心概念，可以将 5bit 数据中心为替换成我们的业务编码。</font>

- <font color=red>如果数据中心不是很多，这时候可以将5bit数据中心位拆成3bit+2bit，其中3bit为数据中心标识，2bit为业务编码，可以设置该值为随机值，放置别人猜测 ID 号。</font>

  

还有很多拆分方法，这里省略，请大家根据业务需求进行拆分



## Snowflake 算法的不足点

根据上面介绍，已经对雪花算法有了大概的了解，不过雪花算法中部分由<font color=red>**时间戳**</font>组成，所以其强依赖机器时钟，如果机器上时钟回拨，会导致发号重复或者服务会处于不可用状态。



为了解决这个问题，网上给出了很多方案：

1. <font color=green>**关闭时钟同步：** 将 ID 生成交给少量服务器，并关闭时钟同步</font>
2. <font color=green>**抛出异常：** 直接抛出异常，交给上层业务处理。</font>
3. <font color=green>**短时间等待：** 如果回拨时间较短，在耗时要求内，比如 5ms，那么可以让时钟等待一小段时间，时间到达后再次进行判断，如果已经超过回拨前的时间则正常执行逻辑，否则接着抛出异常。</font>
4. <font color=green>**使用扩展位预防时钟回拨：** 如果回拨时间很长，那么无法等待，可以调整算法占用的64位，将 1~2位作为回拨位，一旦时钟回拨将回拨位+1，可得到不一样的ID，2位回拨位允许标记三次时钟回拨，基本够使用。如果超出了，再选择抛出异常。</font>



其中比较推荐的就是使用上面介绍的雪花算法扩展位，如利用 WorkerID 作为扩展位，可以让这 10bit 预留出 2bit，让其作为回滚的标识，当发生时钟回拨时候使其值 +1，由于是 2bit 预留位，所以支持最多三次回拨，一般来说够用，毕竟时钟回拨几率比较小，当然如果还发生了，且超过三次后只能抛出进行处理了。



## Snowflake 的 Java 实现示例

这里提供两种方式在 Java 中使用 Snowflake 生成分布式 ID，第一种是使用现成封装好的工具 Hutool，其对 Snowflake 进行了封装，可以直接使用。另一种是自己写代码实现 Snowflake，这种方式可以灵活配置其中的位数分配。



### 方式一：使用 Hutool 工具封装的 Snowflake 工具

通过 Maven 引入 Hutool 工具包：

```xml
<dependency>    
  <groupId>cn.hutool</groupId>    
  <artifactId>hutool-all</artifactId>    
  <version>5.4.2</version>
</dependency>
```



使用 Hutool 中提供的 Snowflake 工具：

```java
public class SnowflakeHutool {    
  public static void main(String[] args) {        
    // 实例化生成 ID 工具对象        
    Snowflake snowflake = IdUtil.getSnowflake(1, 3);        
    long id = snowflake.nextId();    
  }
}
```



### 方式二：自己写代码实现 Snowflake 生成 ID 工具



```java
import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

/**
 * 手动实现 Snowflake 生成 ID 逻辑
 */
public class Snowflake {
    /**
     * 机器id（5位）
     */
    private final long machineId;
    /**
     * 数据中心id（5位）
     */
    private final long datacenterId;
    /**
     * 序列号（12位）
     */
    private long sequence = 0L;
    /**
     * 初始时间戳
     */
    private final long INIT_TIMESTAMP = 1288834974657L;
    /**
     * 机器id位数
     */
    private final long MAX_MACHINE_ID_BITS = 5L;
    /**
     * 数据中心id位数
     */
    private final long DATACENTER_ID_BITS = 5L;
    /**
     * 机器id最大值
     */
    private final long MAX_MACHINE_Id = -1L ^ (-1L << MAX_MACHINE_ID_BITS);
    /**
     * 数据中心id最大值
     */
    private final long MAX_DATACENTER_ID = -1L ^ (-1L << DATACENTER_ID_BITS);
    /**
     * 序列号id最大值
     */
    private final long SEQUENCE_BITS = 12L;
    /**
     * 序列号最大值
     */
    private final long sequenceMask = -1L ^ (-1L << SEQUENCE_BITS);
    /**
     * workerid需要左移的位数（12位）
     */
    private final long WORKER_ID_SHIFT = SEQUENCE_BITS;
    /**
     * 数据id需要左移位数(12序列号)+(5机器id)共17位
     */
    private final long DATACENTER_ID_SHIFT = SEQUENCE_BITS + MAX_MACHINE_ID_BITS;
    /**
     * 时间戳需要左移位数(12序列号)+(5机器id)+(5数据中心id)共22位
     */
    private final long TIMESTAMP_LEFT_SHIFT = SEQUENCE_BITS + MAX_MACHINE_ID_BITS + DATACENTER_ID_BITS;
    /**
     * 上次时间戳，初始值为负数
     */
    private long lastTimestamp = -1L;

    /**
     * 构造方法，进行初始化检测
     *
     * @param machineId    机器ID
     * @param datacenterId 数据ID
     */
    public Snowflake(long machineId, long datacenterId) {
        // 检查数(机器ID)是否大于5或者小于0
        if (machineId > MAX_MACHINE_Id || machineId < 0) {
            throw new IllegalArgumentException(String.format("机器id不能大于 %d 或者小于 0", MAX_MACHINE_Id));
        }        // 检查数(据中心ID)是否大于5或者小于0
        if (datacenterId > MAX_DATACENTER_ID || datacenterId < 0) {
            throw new IllegalArgumentException(String.format("数据中心id不能大于 %d 或者小于 0", MAX_DATACENTER_ID));
        }
        // 配置参数
        this.machineId = machineId;
        this.datacenterId = datacenterId;
    }

    /**
     * 获取下一个生成的分布式 ID
     *
     * @return 分布式 ID
     */
    public synchronized long nextId() {
        // 获取当前时间戳
        long currentTimestamp = timeGen();
        //获取当前时间戳如果小于上次时间戳，则表示时间戳获取出现异常
        if (currentTimestamp < lastTimestamp) {
            // 等待 10ms，如果时间回拨时间短，能在 10ms 内恢复，则正常生产 ID，否则抛出异常
            long offset = lastTimestamp - currentTimestamp;
            if (offset <= 10) {
                try {
                    wait(offset << 1);
                    if (currentTimestamp < lastTimestamp) {
                        throw new RuntimeException("系统时间被回调，无法生成ID");
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException("系统时间被回调，无法生成ID，且等待中断");
                }
            }
        }
        // 判断当前时间戳是否等于上次生成ID的时间戳（同1ms内），是则进行序列号递增+1，如果递增到设置的最大值（默认4096）则等待
        if (lastTimestamp == currentTimestamp) {
            sequence = (sequence + 1) & sequenceMask;
            if (sequence == 0) {
                currentTimestamp = tilNextMillis(lastTimestamp);
            }
        }
        // 如果当前时间戳大于上次生成ID的时间戳，说明已经进入下一毫秒，则设置序列化ID为0
        else {
            sequence = 0;
        }
        // 设置最后时间戳为当前时间戳
        lastTimestamp = currentTimestamp;
        // 生成 ID 并返回结果：
        // (currStamp - INIT_TIMESTAMP) << TIMESTAMP_LEFT_SHIFT     时间戳部分
        // datacenterId << DATACENTER_ID_SHIFT      数据中心部分
        // machineId << WORKER_ID_SHIFT                             机器标识部分
        // sequence                                                 序列号部分
        return ((currentTimestamp - INIT_TIMESTAMP) << TIMESTAMP_LEFT_SHIFT) | (datacenterId << DATACENTER_ID_SHIFT) | (machineId << WORKER_ID_SHIFT) | sequence;
    }

    /**
     * 当某一毫秒时间内产生的ID数超过最大值则进入等待，
     * 循环判断当前时间戳是否已经变更到下一毫秒，
     * 是则返回最新的时间戳
     *
     * @param lastTimestamp 待比较的时间戳
     * @return 当前时间戳
     */
    private long tilNextMillis(long lastTimestamp) {
        long timestamp = timeGen();
        while (timestamp <= lastTimestamp) {
            timestamp = timeGen();
        }
        return timestamp;
    }

    /**
     * 获取系统当前时间
     *
     * @return 系统当前时间（毫秒）
     */
    private long timeGen() {
        return System.currentTimeMillis();
    }

    /**
     * 测试 main 方法
     */
    public static void main(String[] args) {
        // 实例化生成 ID 工具对象
        Snowflake worker = new Snowflake(1, 3);

        // 创建用于存储 id 的集合
        List<Long> idList = new ArrayList<>();
        Set<Long> idSet = new HashSet<>();

        // 标记开始时间
        long start = System.currentTimeMillis();

        // 设置 1000ms 内循环生成 ID
        while (System.currentTimeMillis() - start <= 1000) {
            // 生成 ID 加入集合
            long id = worker.nextId();
            idList.add(id);
            idSet.add(id);
        }

        // 输出1s内生成ID的数量
        System.out.println("生成 ID 总数量：" + idList.size() + "," + idSet.size());
    }
}
```





**优点：**

1. 高性能
2. 趋势递增
3. 可以灵活调整结构
4. 不依赖第三方组件

**缺点：**

1. ID 长度比较大（大概 18位）
2. ID 不连续，生成 ID 是无规则的
3. 强依赖时钟，<font color=red>如果机器时钟回拨，就可能会导致 id 生成重复</font>
4. 不同机器的时钟不是完全一致的，导致全局的 ID 并不是统一向上自增。



# 方案八：使用 Zookeeper 生成 ID

在 Zookeeper 中主要通过节点数据版本号来生成序列号，可以生成 32 位和 64 位的数据版本号，客户端可以使用这个版本号来作为唯一的序列号。在 Zookeeper 中本身就是支持集群模式，所以能保证高可用性，且生成的 ID 为趋势递增且有序，不过在实际使用中很少用 Zookeeper 来充当 ID 生成器，因为 Zookeeper 中存在强一致性，在高并发场景下其性能可能很难满足需求。

不过由于使用 Zookeeper 节点的版本号来充当 ID 号是比较繁琐，需要创建节点获取生成的 ID，然后去掉节点命令前缀，只截取数字部分，最后还要异步执行删除节点（启动新的线程执行删除节点操作，防止占用生成ID线程执行的实际）。过程比较耗时且繁琐，所以，在操作 Zookeeper 时经经常不会采用该方案，常使用 Curator 客户端提供的基于乐观锁的计数器来自增实现 ID 生成，这个过程和数据库自增生成 ID 类似。

**优点：**

- 高可用
- 趋势递增

**缺点：**

- 性能差
- 定期删除之前生成的节点，比较繁琐



# 方案九：使用 MongoDB 创建 ObjectID 生成 ID

# MongoDB 中如何生成 ID 值

在 MongoDB 中每插入一条数据且没有指定 ID 就会生成一个 <font color=red>_id</font> 键作为唯一标识，该键默认是 ObjectID 串，<font color=red>常常可以类似于像数据库插入数据一样往 MongoDB 中插入数据，获取其默认生成的 ObjectID 值来充当分布式 ID。</font>



## MongoDB 的 ObjectId 的组成

在 MongoDB 中默认生成 ObjectId（十六进制）是有一个 12 bit 组词的 BSON，组成类似雪花算法，如下图：



![](/images/distributed/java.webp)

- <font color=green>4 字节时间戳：</font>以 Unix 纪元以来的秒数为单位（精确到秒）
- <font color=green>5 字节随机数</font>
- <font color=green>3 字节递增计数器</font>：初始化为随机值，它能确保相同进程同一秒产生的 ObjectId 也是不一样的。同一秒最多允许每个进程拥有 2563个不同的 ObjectId

## Java 中操作 MongoDB 生成 ID 的实现



引入 Maven 包

```xml
<dependency>    
  <groupId>org.mongodb</groupId>    
  <artifactId>mongo-java-driver</artifactId>  
  <version>3.12.7</version>
</dependency>
```



然后使用插入一条数据生成 ID：

```java
import com.mongodb.MongoClient;
import com.mongodb.client.MongoDatabase;
import org.bson.Document;
import org.bson.types.ObjectId;

public class MongoExample {
    public static void main(String[] args) {
        //连接 MongoDB 服务器，端口号为 27017
        MongoClient mongoClient = new MongoClient("127.0.0.1", 27017);
        // 获取数据库（如果就创建不存在就创建）
        MongoDatabase dbTest = mongoClient.getDatabase("test");
        // 插入一条文档数据（如果就创建不存在就创建）
        Document doc = new Document();
        dbTest.getCollection("myid").insertOne(doc);
        // 获取 ID 值
        ObjectId id = (ObjectId) doc.get("_id");
        // 输出 ID 值
        System.out.println(id);
    }
}
```



**优点**

- 实现简单
- 集群模式易于扩展，没有单点问题

**缺点**

- 性能一般，只能并发量小的业务需求



# 分布式 ID 开源框架

- **滴滴 Tinyid：**   数据库号段模式。 https://github.com/didi/tinyid

- **美团 Leaf：**数据库 + Snowflake。 https://github.com/Meituan-Dianping/Leaf
- **百度 Uid-Generator：**数据库号段模式、Zookeeper + Snowflake。 https://github.com/baidu/uid-generator



这几张流行的开源分布式 ID 的实现都做了如下操作：

- 减少网络延迟，没有使用 Zookeeper、Redis 等作为分布式 ID 的核心组件。
- 可以灵活配置生成的 ID，可以在其中添加跟业务挂钩的业务号，以满足不同业务需求。
- 大部分考虑的是高可用方案，组成统一分布式  ID 分发组件，且组成集群规模，保证可用性。
- 将生成的 ID 存入缓存，这样相当于提前往缓存中存入一批数据，能防止并发突增导致 ID 需求大，也能防止数据库突然不可用。
- 都会设置一个监控器和异步更新缓存中分布式 ID 的多个线程，监控器会监控缓存中的使用比例，达到一定比例后会通知更新缓存的线程执行更新分布式 ID 任务，这样会再往缓存中放入一批可用的 ID 号段



# 总结

对部分方案进行了简单测试，由于没有精细的配置组件环境和参数所以这里的数据不一定准确，只供参考：



| 方案                                | 性能                          | ID 生成速度（单位：s） |
| ----------------------------------- | ----------------------------- | ---------------------- |
| 数据库号段模式生成 ID               | <font color=red>非常高</font> | 100000000+             |
| Snowflake 生成 ID                   | <font color=red>很高</font>   | 4000000+               |
| UUID 生成 ID                        | <font color=red>高</font>     | 710000+                |
| MongoDB 创建 ObjectID 生成 ID       | <font color=blue>一般</font>  | 1500+                  |
| Redis 的 INCR 或 INCRBY 命令生成 ID | <font color=blue>一般</font>  | 2000+                  |
| Zookeeper 的节点 ID                 | <font color=gray>差</font>    | 600+                   |
| 数据库自增生 ID                     | <font color=gray>差</font>    | 300+                   |



根据上面比较，还是比较推荐使用 <font color=red>号段模式 与 Snowflake</font> 两种方案用于生成分布式 ID，具体还是得根据业务实际来选择不同方案。
