---
title: Spring Data JPA
date: 2022-10-30T14:45:27Z
lastmod: 2022-10-30T14:45:27Z
---

# Spring Data JPA

## Spring Data Jpa注解

### @NoRepositoryBean 注解的作用

　　 中间存储库接口用注释@NoRepositoryBean。确保将注释添加到所有存储库接口，Spring Data不应在运行时为其创建实例，此接口只作为父接口。

### @Repository

## Spring Data JPA 自定义Id生成器

　　自增主键采用SnowFlake生成
SnowflakeId 实现IdentifierGenerator类

```java
import org.hibernate.engine.spi.SessionImplementor;
import org.hibernate.id.IdentifierGenerator;
import java.io.Serializable;
/**
 * Twitter_Snowflake<br>
 * SnowFlake的结构如下(每部分用-分开):<br>
 * 0 - 0000000000 0000000000 0000000000 0000000000 0 - 00000 - 00000 - 000000000000 <br>
 * 1位标识，由于long基本类型在Java中是带符号的，最高位是符号位，正数是0，负数是1，所以id一般是正数，最高位是0<br>
 * 41位时间截(毫秒级)，注意，41位时间截不是存储当前时间的时间截，而是存储时间截的差值（当前时间截 - 开始时间截)
 * 得到的值），这里的的开始时间截，一般是我们的id生成器开始使用的时间，由我们程序来指定的（如下下面程序IdWorker类的startTime属性）。41位的时间截，可以使用69年，年T = (1L << 41) / (1000L * 60 * 60 * 24 * 365) = 69<br>
 * 10位的数据机器位，可以部署在1024个节点，包括5位datacenterId和5位workerId<br>
 * 12位序列，毫秒内的计数，12位的计数顺序号支持每个节点每毫秒(同一机器，同一时间截)产生4096个ID序号<br>
 * 加起来刚好64位，为一个Long型。<br>
 * SnowFlake的优点是，整体上按照时间自增排序，并且整个分布式系统内不会产生ID碰撞(由数据中心ID和机器ID作区分)，并且效率较高，经测试，SnowFlake每秒能够产生26万ID左右。
 */
public class SnowflakeId implements IdentifierGenerator{
    // ==============================Fields===========================================
    /**
     * 开始时间截 (2015-01-01)
     */
    private static final long TWEPOCH = 1420041600000L;
    /**
     * 机器id所占的位数
     */
    private static final long WORKER_ID_BITS = 5L;
    /**
     * 数据标识id所占的位数
     */
    private static final long DATA_CENTER_ID_BITS = 5L;
    /**
     * 支持的最大机器id，结果是31 (这个移位算法可以很快的计算出几位二进制数所能表示的最大十进制数)
     */
    private static final long MAX_WORKER_ID = -1L ^ (-1L << WORKER_ID_BITS);
    /**
     * 支持的最大数据标识id，结果是31
     */
    private static final long MAX_DATA_CENTER_ID = -1L ^ (-1L << DATA_CENTER_ID_BITS);
    /**
     * 序列在id中占的位数
     */
    private static final long SEQUENCE_BITS = 12L;
    /**
     * 机器ID向左移12位
     */
    private static final long WORKER_ID_SHIFT = SEQUENCE_BITS;
    /**
     * 数据标识id向左移17位(12+5)
     */
    private static final long DATA_CENTER_ID_SHIFT = SEQUENCE_BITS + WORKER_ID_BITS;
    /**
     * 时间截向左移22位(5+5+12)
     */
    private static final long TIMESTAMP_LEFT_SHIFT = SEQUENCE_BITS + WORKER_ID_BITS + DATA_CENTER_ID_BITS;
    /**
     * 生成序列的掩码，这里为4095 (0b111111111111=0xfff=4095)
     */
    private static final long SEQUENCE_MASK = -1L ^ (-1L << SEQUENCE_BITS);
    /**
     * 工作机器ID(0~31)
     */
    private static long workerId;
    /**
     * 数据中心ID(0~31)
     */
    private static long datacenterId;
    /**
     * 毫秒内序列(0~4095)
     */
    private static long sequence = 0L;
    /**
     * 上次生成ID的时间截
     */
    private static long lastTimestamp = -1L;
    //==============================Constructors=====================================
    public SnowflakeId() {
    }
    private static SnowflakeId snowflakeId=null;
    public static synchronized SnowflakeId getInstance(){
        if(snowflakeId==null){
            snowflakeId=new SnowflakeId();
        }
        return snowflakeId;
    }
    /**
     * 构造函数
     *
     * @param workerId     工作ID (0~31)
     * @param datacenterId 数据中心ID (0~31)
     */
    private SnowflakeId(long workerId, long datacenterId) {
        if (workerId > MAX_WORKER_ID || workerId < 0) {
            throw new IllegalArgumentException(String.format("worker Id can't be greater than %d or less than 0", MAX_WORKER_ID));
        }
        if (datacenterId > MAX_DATA_CENTER_ID || datacenterId < 0) {
            throw new IllegalArgumentException(String.format("datacenter Id can't be greater than %d or less than 0", MAX_DATA_CENTER_ID));
        }
        SnowflakeId.workerId = workerId;
        SnowflakeId.datacenterId = datacenterId;
    }
    // ==============================Methods==========================================
    /**
     * 获得下一个ID (该方法是线程安全的)
     *
     * @return SnowflakeId
     */
    public static synchronized long getId() {
        long timestamp = timeGen();
        //如果当前时间小于上一次ID生成的时间戳，说明系统时钟回退过这个时候应当抛出异常
        if (timestamp < lastTimestamp) {
            throw new RuntimeException(
                    String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", lastTimestamp - timestamp));
        }
        //如果是同一时间生成的，则进行毫秒内序列
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & SEQUENCE_MASK;
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
        return ((timestamp - TWEPOCH) << TIMESTAMP_LEFT_SHIFT)
                | (datacenterId << DATA_CENTER_ID_SHIFT)
                | (workerId << WORKER_ID_SHIFT)
                | sequence;
    }
    /**
     * 阻塞到下一个毫秒，直到获得新的时间戳
     *
     * @param lastTimestamp 上次生成ID的时间截
     * @return 当前时间戳
     */
    protected static long tilNextMillis(long lastTimestamp) {
        long timestamp = timeGen();
        while (timestamp <= lastTimestamp) {
            timestamp = timeGen();
        }
        return timestamp;
    }
    /**
     * 返回以毫秒为单位的当前时间
     *
     * @return 当前时间(毫秒)
     */
    protected static long timeGen() {
        return System.currentTimeMillis();
    }
    @Override
    public Serializable generate(SessionImplementor s, Object obj) {
        return getId();
    }
}
```

## PhysicalNamingStrategy

　　根据业务需要制定自己的命名规范，通过使用PhysicalNamingStrategy可以实现这些规则，而不需要将表名和列名通过@Table和@Column等注解显式指定

### 使用方法

　　继承PhysicalNamingStrategy接口重写一系列接口

```java
public class PhysicalNamingStrategyStandardImpl implements PhysicalNamingStrategy, Serializable {
   /**
    * Singleton access
    */
   public static final PhysicalNamingStrategyStandardImpl INSTANCE = new PhysicalNamingStrategyStandardImpl();

   @Override
   public Identifier toPhysicalCatalogName(Identifier name, JdbcEnvironment context) {
      return name;
   }

   @Override
   public Identifier toPhysicalSchemaName(Identifier name, JdbcEnvironment context) {
      return name;
   }

   @Override
   public Identifier toPhysicalTableName(Identifier name, JdbcEnvironment context) {
      return name;
   }

   @Override
   public Identifier toPhysicalSequenceName(Identifier name, JdbcEnvironment context) {
      return name;
   }

   @Override
   public Identifier toPhysicalColumnName(Identifier name, JdbcEnvironment context) {
      return name;
   }
}
```
