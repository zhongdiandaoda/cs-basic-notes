# UUID

常见的方式。可以利用数据库也可以利用程序生成，一般来说全球唯一。

**优点：**

1. 简单，代码方便。
2. 生成ID性能非常好，基本不会有性能问题。

3. 全球唯一，在遇见数据迁移，系统数据合并，或者数据库变更等情况下，可以从容应对。

**缺点：**

1. 没有排序，无法保证趋势递增。
2. UUID往往是使用字符串存储，查询的效率比较低。
3. 存储空间比较大，如果是海量数据库，就需要考虑存储量的问题。

# 数据库自增长字段

## mysql

- mysql实现，主要是利用自增主键auto_increment来实现，生成全局唯一id
- 在分布式系统中，使用一个共用的数据库，创建一张表
- 表的主键是自增式主键
- 每次需要一个全局唯一id时候，在该表中插入一条数据，返回插入成功数据的主键来作为全局唯一的id

优点：接入简单；严格自增

缺点：扩展性差；性能受DB吞吐量制约

## redis

- redis的实现，主要依赖于redis单线程，使用redis的原子命令incr就可以生成一个全局唯一的id
- 在分布式系统中，使用一个共用的redis，共用一个主键生成key
- 每次使用命令incr就可以生成一个全局唯一的id

优点：接入简单；严格自增

缺点：扩展性差；性能受DB吞吐量制约

## 多台mysql

使用多台MySQL服务器，每一个服务器设置不同的初始值和自增值。用一个负载均衡器连接这些服务器。

优点：简单；有序

缺点：

- ID不是严格递增的
- 耗费资源

# 推特 Snowflake

最高位不用，永远为0，其余三组bit占位均可浮动，看具体的业务需求而定。默认情况下41bit的时间戳可以支持该算法使用到2082年，10bit的工作机器id可以支持1023台机器，序列号占用12bit。

SnowFlake算法在同一毫秒内最多可以生成多少个全局唯一ID呢：： **同一毫秒的ID数量 = 1024 X 4096 = 4194304**

优点：

- 本地实现，性能高
- 时间戳在高位，自增序列号在低位，保证唯一的同时，时间严格递增
- 不依赖外部存储系统

缺点：

- 依赖本地时钟服务
- 如果服务器发生时钟回拨，基于本地时钟系统获取的时间戳可能会造成id重复

```
public class SnowflakeIdWorker {
    /**
    * 开始时间截 (2015-01-01)
    */
    private final long twepoch = 1420041600000L;
    /**
    * 机器id所占的位数
    */
    private final long workerIdBits = 5L;
    /**
    * 数据标识id所占的位数
    */
    private final long datacenterIdBits = 5L;
    /**
    * 支持的最大机器id，结果是31 (这个移位算法可以很快的计算出几位二进制数所能表示的最大十进制数)
    */
    private final long maxWorkerId = -1L ^ (-1L << workerIdBits);
    /**
    * 支持的最大数据标识id，结果是31
    */
    private final long maxDatacenterId = -1L ^ (-1L << datacenterIdBits);
    /**
    * 序列在id中占的位数
    */
    private final long sequenceBits = 12L;
    /**
    * 机器ID向左移12位
    */
    private final long workerIdShift = sequenceBits;
    /**
    * 数据标识id向左移17位(12+5)
    */
    private final long datacenterIdShift = sequenceBits + workerIdBits;
    /**
    * 时间截向左移22位(5+5+12)
    */
    private final long timestampLeftShift = sequenceBits + workerIdBits + datacenterIdBits;
    /**
    * 生成序列的掩码，这里为4095 (0b111111111111=0xfff=4095)
    */
    private final long sequenceMask = -1L ^ (-1L << sequenceBits);
    /**
    * 工作机器ID(0~31)
    */
    private long workerId;
    /**
    * 数据中心ID(0~31)
    */
    private long datacenterId;
    /**
    * 毫秒内序列(0~4095)
    */
    private long sequence = 0L;
    /**
    * 上次生成ID的时间截
    */
    private long lastTimestamp = -1L;
    /**
    * 构造函数
    * @param workerId     工作ID (0~31)
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
        // 如果当前时间小于上一次ID生成的时间戳，说明系统时钟回退过这个时候应当抛出异常
        if (timestamp < lastTimestamp) {
            throw new RuntimeException(
                String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", lastTimestamp - timestamp));
        }
        // 如果是同一时间生成的，则进行毫秒内序列
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & sequenceMask;
            // 毫秒内序列溢出
            if (sequence == 0) {
                //阻塞到下一个毫秒,获得新的时间戳
                timestamp = tilNextMillis(lastTimestamp);
            }
        }
        // 时间戳改变，毫秒内序列重置
        else {
            sequence = 0L;
        }
        // 上次生成ID的时间截
        lastTimestamp = timestamp;
        // 移位并通过或运算拼到一起组成64位的ID
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
    
    public static void main(String[] args) throws InterruptedException {
        SnowflakeIdWorker idWorker = new SnowflakeIdWorker(0, 0);
        for (int i = 0; i < 10; i++) {
            long id = idWorker.nextId();
            Thread.sleep(1);
            System.out.println(id);
        }
    }
}
```
