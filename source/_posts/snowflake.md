---
title: 分布式ID生成策略
date: 2019-01-26 15:47:13
categories: 分布式
toc: true
---

最近在整提现到微信零钱的需求，需要在分布式环境下生成全局唯一的流水ID，在借鉴了Twitter的SnowFlake算法以及MongoId的实现之后有了自己的实现，下面记录的是snowflake算法以及MongoId的生成规则。

### SnowFlake算法解析

#### 基本概念

Twitter的SnowFlake算法生成的是64位大小的整数，结构如下图所示

<img src="snowflake.jpg">

+ 最高位：二进制中最高位为1的都是负数，但是生成的ID一般都是正整数，所以这个最高位都为0
+ 41位：用来记录毫秒级的时间戳，41位可以表示2^41个数字，范围为0 至 2^41−1。这41位存储的不是当前时间戳，而是时间戳的差值(当前时间戳 - 开始时间戳)，也就是说41位可以表示2^41个毫秒值，转化为年则是(2^41)/(1000∗60∗60∗24∗365)=69年，开始时间戳一般为应用开始跑的时间
+ 10位：用来记录工作机器id。可以表示2^10=1024台机器，当然也可以采用混编，列如前5位datacenterId标识机器，后5位workerId标识进程
+ 12位：序列号，用来记录同一毫秒内产生的不同id。12位的序列号支持同一个机器在一毫秒内最多生成2^12=4096个ID序号，如果同一毫秒内，序列号已经达到上限，就等下一毫秒同时序列号置为0

由此我们不难发现，SnowFlake生成的ID在整体上是根据时间递增的，并且在分布式系统内，不会出现ID的碰撞(由10位工作机器id来进行区分)。不依赖任何第三方服务，性能也较高，可以根据当前的业务场景来分配bit位，非常灵活

#### 简易实现

``` java
public class SnowflakeIdWorker {

    /** 开始时间截 (2015-01-01) */
    private final long twepoch = 1420041600000L;

    /** 机器id所占的位数 */
    private final long workerIdBits = 5L;

    /** 数据标识id所占的位数 */
    private final long datacenterIdBits = 5L;

    /** 支持的最大机器id，结果是31 (这个移位算法可以很快的计算出几位二进制数所能表示的最大十进制数) */
    private final long maxWorkerId = -1L ^ (-1L << workerIdBits);

    /** 支持的最大数据标识id，结果是31 */
    private final long maxDatacenterId = -1L ^ (-1L << datacenterIdBits);

    /** 序列在id中占的位数 */
    private final long sequenceBits = 12L;

    /** 机器ID向左移12位 */
    private final long workerIdShift = sequenceBits;

    /** 数据标识id向左移17位(12+5) */
    private final long datacenterIdShift = sequenceBits + workerIdBits;

    /** 时间截向左移22位(5+5+12) */
    private final long timestampLeftShift = sequenceBits + workerIdBits + datacenterIdBits;

    /** 生成序列的掩码，这里为4095 (0b111111111111=0xfff=4095) */
    private final long sequenceMask = -1L ^ (-1L << sequenceBits);

    /** 工作机器ID(0~31) */
    private long workerId;

    /** 数据中心ID(0~31) */
    private long datacenterId;

    /** 毫秒内序列(0~4095) */
    private long sequence = 0L;

    /** 上次生成ID的时间截 */
    private long lastTimestamp = -1L;

    /**
     * 构造函数
     * @param workerId 工作ID (0~31)
     * @param datacenterId 数据中心ID (0~31)
     */
    public SnowflakeIdWorker(long workerId, long datacenterId) {
        if (workerId > maxWorkerId || workerId < 0) {
            throw new IllegalArgumentException(String.format("worker Id can't be greater than %d or less than 0", maxWorkerId));
        }
        if (datacenterId > maxDatacenterId || datacenterId < 0) {
            throw new IllegalArgumentException(String.format("datacenter Id can't be greater than %d or less than 0", maxDatacenterId));
        }
        this.workerId = workerId;
        this.datacenterId = datacenterId;
    }

    /**
     * 获得下一个ID (该方法是线程安全的)
     * @return SnowflakeId
     */
    public synchronized long nextId() {
        long timestamp = timeGen();

        //如果当前时间小于上一次ID生成的时间戳，说明系统时钟回退过这个时候应当抛出异常
        if (timestamp < lastTimestamp) {
            throw new RuntimeException(
                    String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", lastTimestamp - timestamp));
        }

        //如果是同一时间生成的，则进行毫秒内序列
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & sequenceMask;
            //毫秒内序列溢出
            if (sequence == 0) {
                //阻塞到下一个毫秒,获得新的时间戳
                timestamp = tilNextMillis(lastTimestamp);
            }
        }
        //时间戳改变，毫秒内序列重置
        else {
            sequence = 0L;
        }

        //上次生成ID的时间截
        lastTimestamp = timestamp;

        //移位并通过或运算拼到一起组成64位的ID
        return ((timestamp - twepoch) << timestampLeftShift) //
                | (datacenterId << datacenterIdShift) //
                | (workerId << workerIdShift) //
                | sequence;
    }

    /**
     * 阻塞到下一个毫秒，直到获得新的时间戳
     * @param lastTimestamp 上次生成ID的时间截
     * @return 当前时间戳
     */
    protected long tilNextMillis(long lastTimestamp) {
        long timestamp = timeGen();
        while (timestamp <= lastTimestamp) {
            timestamp = timeGen();
        }
        return timestamp;
    }

    /**
     * 返回以毫秒为单位的当前时间
     * @return 当前时间(毫秒)
     */
    protected long timeGen() {
        return System.currentTimeMillis();
    }

    public static void main(String[] args) {
        SnowflakeIdWorker idWorker = new SnowflakeIdWorker(0, 0);
        for (int i = 0; i < 1000; i++) {
            long id = idWorker.nextId();
            System.out.println(Long.toBinaryString(id));
            System.out.println(id);
        }
    }
}
```

### MongoID生成规则

熟悉Mongo的同学都知道，Mongo里的_id是24位的字符串并且全局唯一，类型为ObjectId。这个id的生成就是充分的借鉴了SnowFlake算法。在此我们对"5c4a7fcc0ed0240001a79289"这个id进行解析

#### 构成

"5c4a7fcc0ed0240001a79289"这个24位的字符串看起来很长，其实它是由一组十六进制的字符串构成的，每个字节两位的十六进制数字，总共占用12字节的存储空间。官方文档对ObjectId的解释如下图

<img src="mongoid.png">

- Time："5c4a7fcc"，由十六进制转换为十进制可得到1548386252秒级别时间戳
- Machine："0ed024"，这三个字节是所在主机的唯一标识符，一般是机器主机名的散列值，确保了在分布式环境下不冲突，同一台机器生产的这个三个字节都是一样的
- PID："0001"，进程id，确保同一台机器的不同Mongo进程不会产生冲突
- INC："a79289"，自增计数器，用来确保同一秒内产生的ObjectId不冲突，允许同一秒内生成(2^8)^3=16777216条记录

总的来说，前面4个字节是时间戳，记录文档的创建时间；接下来的3个字节是机器的唯一标识符，确保不同的机器产生不同的ObjectId；紧接着2个字节的进程ID，确保同一机器下不同的Mongo进程产生不同的ObjectId；最后通过三个字节的自增计数器，确保同一秒内产生ObjectId的唯一性

#### 客户端驱动源码解析

直接调用客户端驱动的get()方法的话，整体流程如下

``` java
public static ObjectId get() {
    return new ObjectId();
}

public ObjectId() {
    this(new Date());
}

public ObjectId(final Date date) {
    this(dateToTimestampSeconds(date), NEXT_COUNTER.getAndIncrement() & LOW_ORDER_THREE_BYTES, false);
}

private ObjectId(final int timestamp, final int randomValue1, final short randomValue2, final int counter, final boolean checkCounter) {
    if ((randomValue1 & 0xff000000) != 0) {
        throw new IllegalArgumentException("The machine identifier must be between 0 and 16777215 (it must fit in three bytes).");
    }
    if (checkCounter && ((counter & 0xff000000) != 0)) {
        throw new IllegalArgumentException("The counter must be between 0 and 16777215 (it must fit in three bytes).");
    }
    this.timestamp = timestamp;
    this.counter = counter & LOW_ORDER_THREE_BYTES;
    this.randomValue1 = randomValue1;
    this.randomValue2 = randomValue2;
}

private static int dateToTimestampSeconds(final Date time) {
    return (int) (time.getTime() / 1000);
}

static {
    try {
        SecureRandom secureRandom = new SecureRandom();
        RANDOM_VALUE1 = secureRandom.nextInt(0x01000000);
        RANDOM_VALUE2 = (short) secureRandom.nextInt(0x00008000);
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

时间戳直接通过time.getTime() / 1000获得秒级别时间戳，默认情况下机器码和进程号如果不传的话通过SecureRandom进行生成，自增数通过AtomicInteger的getAndIncrement()方法生成

### 最终业务解决方案

需要生成32位的全局唯一流水号，但是他吗的qps无限趋近于0，又赶着上线。于是，别跟我扯什么分布式id生成，直接采用了mongo的_id，在此基础上加上一串固定的数字，同样也能做到全局唯一。对提现主流程新增redis锁，确保同一用户同一时刻只能有一个请求在走主流程，生成id的规则后续可以随时替换。
