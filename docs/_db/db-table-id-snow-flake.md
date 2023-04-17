---
layout: post
title: 基于雪花算法生成自增 ID，代码来自 MybatisPlus
categories: db
tags: DB 
date: 2023-04-14
---
基于雪花算法生成自增 ID，代码来自 MybatisPlus，这里把代码 copy 下来，并把所有代码整合进一个类中，方便结合 Mybatis 快速查找使用
<!--more-->

源项目 [在这里](https://gitee.com/yu120/sequence)

```java
import java.lang.management.ManagementFactory;
import java.net.InetAddress;
import java.net.NetworkInterface;
import java.sql.Timestamp;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;
/**
 * 分布式高效有序 ID 生产黑科技(sequence)
 *
 * <p>优化开源项目：https://gitee.com/yu120/sequence</p>
 *
 * @author hubin
 * @since 2016-08-18
 */
public class SnowFlake {

    private static final org.apache.ibatis.logging.Log logger = org.apache.ibatis.logging.LogFactory.getLog(SnowFlake.class);
    /**
     * 时间起始标记点，作为基准，一般取系统的最近时间（一旦确定不能变动）
     */
    private static final long twepoch = 1288834974657L;
    /**
     * 机器标识位数
     */
    private final long workerIdBits = 5L;
    private final long datacenterIdBits = 5L;
    private final long maxWorkerId = ~(-1L << workerIdBits);
    private final long maxDatacenterId = ~(-1L << datacenterIdBits);
    /**
     * 毫秒内自增位
     */
    private final long sequenceBits = 12L;
    private final long workerIdShift = sequenceBits;
    private final long datacenterIdShift = sequenceBits + workerIdBits;
    /**
     * 时间戳左移动位
     */
    private final long timestampLeftShift = sequenceBits + workerIdBits + datacenterIdBits;
    private final long sequenceMask = -1L ^ (-1L << sequenceBits);

    private final long workerId;

    /**
     * 数据标识 ID 部分
     */
    private final long datacenterId;
    /**
     * 并发控制
     */
    private long sequence = 0L;
    /**
     * 上次生产 ID 时间戳
     */
    private long lastTimestamp = -1L;
    /**
     * IP 地址
     */
    private InetAddress inetAddress;

    public SnowFlake(InetAddress inetAddress) {
        this.inetAddress = inetAddress;
        this.datacenterId = getDatacenterId(maxDatacenterId);
        this.workerId = getMaxWorkerId(datacenterId, maxWorkerId);
    }

    /**
     * 有参构造器
     *
     * @param workerId     工作机器 ID
     * @param datacenterId 序列号
     */
    public SnowFlake(long workerId, long datacenterId) {
        if(workerId > maxWorkerId || workerId < 0){
            throw new IllegalArgumentException( String.format("worker Id can't be greater than %d or less than 0", maxWorkerId));
        }
        
        if(datacenterId > maxDatacenterId || datacenterId < 0){
            throw new IllegalArgumentException(String.format("datacenter Id can't be greater than %d or less than 0", maxDatacenterId));
        }

        this.workerId = workerId;
        this.datacenterId = datacenterId;
    }

    /**
     * 获取 maxWorkerId
     */
    protected long getMaxWorkerId(long datacenterId, long maxWorkerId) {
        StringBuilder mpid = new StringBuilder();
        mpid.append(datacenterId);
        String name = ManagementFactory.getRuntimeMXBean().getName();
        if (org.springframework.util.StringUtils.hasText(name)) {
            /*
             * GET jvmPid
             */
            mpid.append(name.split("@")[0]);
        }
        /*
         * MAC + PID 的 hashcode 获取16个低位
         */
        return (mpid.toString().hashCode() & 0xffff) % (maxWorkerId + 1);
    }

    /**
     * 数据标识id部分
     */
    protected long getDatacenterId(long maxDatacenterId) {
        long id = 0L;
        try {
            if (null == this.inetAddress) {
                this.inetAddress = InetAddress.getLocalHost();
            }
            NetworkInterface network = NetworkInterface.getByInetAddress(this.inetAddress);
            if (null == network) {
                id = 1L;
            } else {
                byte[] mac = network.getHardwareAddress();
                if (null != mac) {
                    id = ((0x000000FF & (long) mac[mac.length - 2]) | (0x0000FF00 & (((long) mac[mac.length - 1]) << 8))) >> 6;
                    id = id % (maxDatacenterId + 1);
                }
            }
        } catch (Exception e) {
            logger.warn(" getDatacenterId: " + e.getMessage());
        }
        return id;
    }

    /**
     * 获取下一个 ID
     *
     * @return 下一个 ID
     */
    public synchronized long nextId() {
        long timestamp = timeGen();
        //闰秒
        if (timestamp < lastTimestamp) {
            long offset = lastTimestamp - timestamp;
            if (offset <= 5) {
                try {
                    wait(offset << 1);
                    timestamp = timeGen();
                    if (timestamp < lastTimestamp) {
                        throw new RuntimeException(String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", offset));
                    }
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }
            } else {
                throw new RuntimeException(String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", offset));
            }
        }

        if (lastTimestamp == timestamp) {
            // 相同毫秒内，序列号自增
            sequence = (sequence + 1) & sequenceMask;
            if (sequence == 0) {
                // 同一毫秒的序列数已经达到最大
                timestamp = tilNextMillis(lastTimestamp);
            }
        } else {
            // 不同毫秒内，序列号置为 1 - 2 随机数
            sequence = ThreadLocalRandom.current().nextLong(1, 3);
        }

        lastTimestamp = timestamp;

        // 时间戳部分 | 数据中心部分 | 机器标识部分 | 序列号部分
        return ((timestamp - twepoch) << timestampLeftShift)
                | (datacenterId << datacenterIdShift)
                | (workerId << workerIdShift)
                | sequence;
    }

    protected long tilNextMillis(long lastTimestamp) {
        long timestamp = timeGen();
        while (timestamp <= lastTimestamp) {
            timestamp = timeGen();
        }
        return timestamp;
    }

    protected long timeGen() {
        return SystemClock.now();
    }

    /**
     * 反解id的时间戳部分
     */
    public static long parseIdTimestamp(long id) {
        return (id>>22)+twepoch;
    }

    /**
     * 高并发场景下System.currentTimeMillis()的性能问题的优化
     *
     * <p>System.currentTimeMillis()的调用比new一个普通对象要耗时的多（具体耗时高出多少我还没测试过，有人说是100倍左右）</p>
     * <p>System.currentTimeMillis()之所以慢是因为去跟系统打了一次交道</p>
     * <p>后台定时更新时钟，JVM退出时，线程自动回收</p>
     * <p>10亿：43410,206,210.72815533980582%</p>
     * <p>1亿：4699,29,162.0344827586207%</p>
     * <p>1000万：480,12,40.0%</p>
     * <p>100万：50,10,5.0%</p>
     *
     * @author hubin
     * @since 2016-08-01
     */
    private static class SystemClock {

        private final long period;
        private final AtomicLong now;

        private SystemClock(long period) {
            this.period = period;
            this.now = new AtomicLong(System.currentTimeMillis());
            scheduleClockUpdating();
        }

        private static SystemClock instance() {
            return InstanceHolder.INSTANCE;
        }

        public static long now() {
            return instance().currentTimeMillis();
        }

        public static String nowDate() {
            return new Timestamp(instance().currentTimeMillis()).toString();
        }

        private void scheduleClockUpdating() {
            ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor(runnable -> {
                Thread thread = new Thread(runnable, "System Clock");
                thread.setDaemon(true);
                return thread;
            });
            scheduler.scheduleAtFixedRate(() -> now.set(System.currentTimeMillis()), period, period, TimeUnit.MILLISECONDS);
        }

        private long currentTimeMillis() {
            return now.get();
        }

        private static class InstanceHolder {
            public static final SystemClock INSTANCE = new SystemClock(1);
        }
    }
}
```