---
title: "「Redis」Redis实现限流器"
subtitle: ""
layout: post
author: "afsun"
header-style: text
tags:
  - Redis
---
# 限流

- 当系统的处理能力有限时，如何阻止计划外的请求继续对系统施压，这是一个需要重视的问题除了控制流量，限流还有一个应用目的是用于控制用户行为，避免垃圾请求。

# ZSET方式实现(简单限流)

![Untitled%207b13dd0fd31640ba9f2cd1510e7a268d/Untitled.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/11/091344-95207.png)

滑动窗口法

该算法由上图所示,滑动窗口就是限制多少秒内的时间区间

思路: 

- 移除对应set中时间窗口外的数据
- 每次调用请求之前会通过redis的**ZRANGEBYSCORE**方法来获取该时间窗口内的请求次数 ,如果大于等于设定值 则不予调用
- 向给redis中的该请求set中插入一条数据score为请求的毫秒时间戳,设置过期时间为窗口时间+1秒左右

## 代码实现

```java
public class RedisLimiting {

    @Autowired
    JedisPool jedisPool;

    public boolean isAllowRequest(String userID,String actionKey,int milSec,long requestNum){
        Jedis jedis = jedisPool.getResource();
        Pipeline pipelined = jedis.pipelined();
        try {
            // 设置主键key
            String key = String.format("limit:%s-%s",actionKey,userID);
            long nowTime = System.currentTimeMillis();
            // 开启管道事务
            pipelined.multi();
            // 加入到set中
            pipelined.zadd(key,nowTime,""+nowTime);
            // 移除时间窗口外的数据
            pipelined.zremrangeByRank(key,0,nowTime-milSec);
            // 获得目前的请求数量
            Response<Long> zcard = pipelined.zcard(key);
            // 设置过期时间
            pipelined.expire(key,milSec+1000);
            pipelined.exec();
            return zcard.get()<=requestNum;
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            try {
                pipelined.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
            jedis.close();
        }

        return false;
    }
}
```

## 缺陷

采用ZSET来存储访问的次数,如果限制的访问次数较多的话,列如一秒钟限制1000万次访问,那就要在redis存储1000万个数据,浪费内存

# 漏斗算法实现(最常用)

![Untitled%207b13dd0fd31640ba9f2cd1510e7a268d/Untitled%201.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/11/091329-89272.png)

漏桶算法

将限流看做一个漏斗,源头的水流速过快就会往外溢出并不会加速下流,漏斗的剩余空间就表示可用的次数,漏斗的流速表示每毫秒可调用次数,漏斗容量代表能进行调用的次数

## 代码实现

```java
package com.afsun.Config;

import org.springframework.beans.factory.annotation.Autowired;
import redis.clients.jedis.JedisPool;

import java.util.concurrent.ConcurrentHashMap;

/**
 * @program: springproject
 * @description:
 * @author: afsun
 * @create: 2020-01-09 14:13
 */
public class FunnelRateLimiter {

    @Autowired
    JedisPool jedisPool;

    private  ConcurrentHashMap<String,Funnel> list = new ConcurrentHashMap<>();

    /**
     * 漏斗类
     */
    static class Funnel{
        // 总容量
        private long capcity;
        // 流速
        private double leakRate;
        // 剩余容量
        private long leftQuota;
        // 上次漏水时间
        private long lastTime;

        public Funnel(long capcity, double leakRate {
            this.capcity = capcity;
            this.leakRate = leakRate;
        }

        // 释放空间
        public void makeQuota(){
            // 当前毫秒时间戳
            long nowTs = System.currentTimeMillis();
            // 两次漏水间隔
            long detalTs = nowTs - lastTime;
            // 空出的容量
            long deltaQuota = (long) (detalTs * leakRate);
            // 时间间隔太长,全部漏出
            if (deltaQuota <0 ){
                leakRate = capcity;
                lastTime = nowTs;
                return;
                // 时间太短 没有漏水
            }else if (deltaQuota<1){
                return ;
            }
            // 设置剩余容量
            leftQuota+=deltaQuota;
            if (leftQuota>capcity)leftQuota=capcity;
            lastTime = nowTs;
        }

        // 倒水
        public boolean watring(int quota){
            makeQuota();
            if (leftQuota>=quota){
                leftQuota -=quota;
                return true;
            }
            return false;
        }
    }

    /**
     *  流速 = n毫秒内能调用次数/n
     *  容量 = 调用次数
     * @param userID
     * @param actionKey
     * @param milSec
     * @param requestNum
     * @return
     */
    public boolean isAllowRequest(String userID,String actionKey,int milSec,long requestNum){
        // 设置主键key
        String key = String.format("limit:%s-%s",actionKey,userID);
        Funnel funnel = list.get(key);
        if (funnel == null) {
            funnel = new Funnel(requestNum,requestNum/milSec);
            list.put(key,funnel);
        }
        return funnel.watring(1);
    }

}
```

## 令牌桶算法

![Untitled%207b13dd0fd31640ba9f2cd1510e7a268d/Untitled%202.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/11/091331-64221.png)

令牌桶过程

令牌桶算法算是漏斗算法的改进版,为了处理短时间的突发流量而做了优化,令牌桶算法主要由三部分组成令牌流、数据流、令牌桶.

令牌流: 固定的速率生产令牌

数据流: 服务的请求流

令牌桶: 装在令牌的桶

- 令牌流源源不断的生产令牌放入到桶中,如果桶满了则丢弃令牌  请求服务会去令牌桶中拿令牌如果没有获取到令牌则会拒绝访问,只有拿到令牌的请求才会被执行.

这样当访问量较小的时候会留存令牌的数量,这样当突发的大量请求过来时会有足够的令牌调用,如果访问量一直较大的话访问的速率就跟令牌的生产速率有关,变成漏桶一样的固定速率

(思考??? 为什么是优化)

![Untitled%207b13dd0fd31640ba9f2cd1510e7a268d/Untitled%203.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/11/091332-224638.png)

![Untitled%207b13dd0fd31640ba9f2cd1510e7a268d/Untitled%204.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/11/091328-714322.png)

## 桶的缺陷

在分布式系统中,需要把Funnel类的数据结构放在redis的HASH结构中,一次漏水计算需要:

- 从redis拿出Funnel的数据
- 内存中计算
- 放入redis中

这三个操作无法原子化,如果在代码中加锁的话会影响效率代码复杂度加高

# Redis_Cell实现

- Redis 4.0 提供了一个限流 Redis 模块，它叫 redis-cell。该模块也使用了漏斗算法，并提供了原子的限流指令

请求

![Untitled%207b13dd0fd31640ba9f2cd1510e7a268d/Untitled%205.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/11/091326-877859.png)

响应

```bash
cl.throttle laoqian:reply 15 30 60

1. (integer) 0 # 0 表示允许，1 表示拒绝
2. (integer) 15 # 漏斗容量 capacity
3. (integer) 14 # 漏斗剩余空间 left_quota
4. (integer) -1 # 如果拒绝了，需要多长时间后再试(漏斗有空间了，单位秒)
5. (integer) 2 # 多长时间后，漏斗完全空出来(left_quota==capacity，单位秒)
```