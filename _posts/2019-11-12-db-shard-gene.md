---
bg: "tools.jpg"
layout: post
title:  "分库基因生成示例"
date:   2019-11-12 02:00:00 +0800
categories: posts
tags: ['demo']
author: jasonlu
---

## 什么是分库基因
***
在前一篇分库分表的文章中，提到分片策略时。我们提到了一种把userId混入orderId的方法。利用这一方法，在orderId中就带有userId的信息，同样能达到利用userId分片的效果。这个userId就是分库基因。

<br/>
<br/>

## 分库基因的原理
***
一个数对2^n取模，其结果就等于这个数的二进制的最后n位数。

放到这里来讲，当我们根据userId对分片数2^n取模时，其结果只由userId这个数的二进制的最后n位数决定（在上一篇文章中，以分片数为4来计算，即分片结果由userId的二进制数的最后2位决定）。

<br/>
<br/>

## 示例
***
以分片数为2^2为例，我们可以在雪花算法的基础上进行修改。

首先利用雪花算法生成一个orderId，并传入userId，先对orderId的最后2位清零。然后再把userId的最后两位数字保留，其余清零。最后把处理后的orderId和userId按位或运算，得到最后结果。

```
public long geneGenerate(long orderId, long userId) {
        // 数字2为最后清零的位数
        long preOrderId = ((orderId >> 2) << 2);
        // 获取userId的最后2位数，即userId % 4 = userId & 3（一个数对2^n取模，等于该数和2^n-1按位与运算）
        long gene = userId & 3;
        // 按位或运算得到结果
        long postOrderId = preOrderId | gene;
        return postOrderId;
    }
```

完整代码示例：

```
public class IdWorker {

    public static IdWorker idWorker = new IdWorker(1, 1);

    public static Long getId() {
        return idWorker.nextId();
    }

    /**
     * 开始时间截
     */
    private final long twepoch = 1489111610226L;

    /**
     * 机器id所占的位数
     */
    private final long workerIdBits = 5L;

    /**
     * 数据标识id所占的位数
     */
    private final long dataCenterIdBits = 5L;

    /**
     * 支持的最大机器id，结果是31 (这个移位算法可以很快的计算出几位二进制数所能表示的最大十进制数)
     */
    private final long maxWorkerId = -1L ^ (-1L << workerIdBits);

    /**
     * 支持的最大数据标识id，结果是31
     */
    private final long maxDataCenterId = -1L ^ (-1L << dataCenterIdBits);

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
    private final long dataCenterIdShift = sequenceBits + workerIdBits;

    /**
     * 时间截向左移22位(5+5+12)
     */
    private final long timestampLeftShift = sequenceBits + workerIdBits + dataCenterIdBits;

    /**
     * 生成序列的掩码，这里为4095
     */
    private final long sequenceMask = -1L ^ (-1L << sequenceBits);

    /**
     * 工作机器ID(0~31)
     */
    private long workerId;

    /**
     * 数据中心ID(0~31)
     */
    private long dataCenterId;

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
     *
     * @param workerId     工作ID (0~31)
     * @param dataCenterId 数据中心ID (0~31)
     */
    public IdWorker(long workerId, long dataCenterId) {
        if (workerId > maxWorkerId || workerId < 0) {
            throw new IllegalArgumentException(String.format("workerId can't be greater than %d or less than 0", maxWorkerId));
        }
        if (dataCenterId > maxDataCenterId || dataCenterId < 0) {
            throw new IllegalArgumentException(String.format("dataCenterId can't be greater than %d or less than 0", maxDataCenterId));
        }
        this.workerId = workerId;
        this.dataCenterId = dataCenterId;
    }


    /**
     * 获得下一个ID (线程安全)
     *
     * @return SnowflakeId
     */
    public synchronized Long nextId() {
        long timestamp = timeGen();

        //如果当前时间小于上一次ID生成的时间戳，说明系统时钟回退过这个时候应当抛出异常
        if (timestamp < lastTimestamp) {
            throw new RuntimeException(
                    String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", lastTimestamp - timestamp));
        }

        //如果是同一时间生成的，则进行毫秒内序列
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 4) & sequenceMask;
            //毫秒内序列溢出
            if (sequence == 0) {
                //阻塞到下一个毫秒,获得新的时间戳
                timestamp = tilNextMillis(lastTimestamp);
            }
        }
        //时间戳改变，毫秒内序列重置
        else {
            //不同毫秒内，序列号置为0 （改了这里，当前毫秒数为双数设置为0，为单数时设为1）
            sequence = timestamp & 1L;
        }

        //上次生成ID的时间截
        lastTimestamp = timestamp;

        //移位并通过或运算拼到一起组成64位的ID
        return ((timestamp - twepoch) << timestampLeftShift) //
                | (dataCenterId << dataCenterIdShift) //
                | (workerId << workerIdShift) //
                | sequence;
    }

    /**
     * 阻塞到下一个毫秒，直到获得新的时间戳
     *
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
     *
     * @return 当前时间(毫秒)
     */
    protected long timeGen() {
        return System.currentTimeMillis();
    }
    
    /**
     * 传入主键id和基因id，生成带有基因的主键id
     *
     */
    public long geneGenerate(long orderId, long userId) {
        // 数字2为最后清零的位数
        long preOrderId = ((orderId >> 2) << 2);
        // 获取userId的最后2位数，即userId % 4 = userId & 3（一个数对2^n取模，等于该数和2^n-1按位与运算）
        long gene = userId & 3;
        // 按位或运算得到结果
        long postOrderId = preOrderId | gene;
        return postOrderId;
    }
    
}
```
