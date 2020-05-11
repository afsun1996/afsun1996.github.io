---
title: "「Redis」redis的基础操作"
subtitle: "Redis的基础操作命令备忘录"
layout: post
author: "afsun"
header-style: text
tags:
  - Redis
---
# 基本操作

# **Redis记录**

## **基本操作**

### **字符串**

1. SET (KEY) (VALUE) 新增一个键值对1
2. SETEX (KEY) (TIME) (VALUE) 新增键值对设置超时时间
3. 
4. SETNX (KEY) (VALUE) 如果KEY不存在则新建这个键值对
5. GET (KEY)	获取这个健对应的值
6. MGET(KEY1 KEY2.....) 批量获取键值对
7. EXISTS (KEY) 是否存在这个健
8. DEL (KEY)	删除这个健对应的键值对
9. EXPIRE (KEY) 对指定的键值对设置超时时间
10. INCR (KEY)	如果健对应的值是数值型的那么自增1
11. INCRBY (KEY) (NUMBER) 对应值加上number

### **列表**

1. RPUSH (KEY) (VALUE1 VALUE2 ...) 向列表中放入数值(从右边压入)
2. LLEN (KEY) 查询键对应的列表的长度
3. LPOP (KEY) 从列表左边返回
4. RPOP (KEY) 从列表右边返回
5. LINDEX (KEY) (INDEX) 获取key对应的列表的第index个的值
6. LRANGE (KEY) (START,END) 获取key对应列表的第start位到第end位的子列
7. LTRIM (KEY) (START,END) 将key对应的列表只保留start位到第end位

### **HASH字典**

1. HSET (KEY) (KEY1 VALUE1) 将键值对设置进key对应的字典中
2. HMSET (KEY) (KEY1 VALUE1 KEY2 VALUE2 .....) 批量设置
3. HGETALL (KEY) 获得这个key对应字典的所有键值对
4. HGET (KEY) (KEY1) 获得key对应字典的key1的值
5. HINCBY(KEY) (KEY1) (NUM) 可以对HASH中的单个的子key进行增加

### **SET无序列表(KEY唯一)**

1. SADD (KEY) (VALUE) 将值传入key对应的set中
2. SMEMBERS (KEY) 返回key对应的set的所有的value
3. SISMEMBER (KEY) (VALUE) 判断这个value是否存在于这个set中
4. SCARD (KEY) 获取这个set的长度
5. SPOP (KEY) 弹出一个VALUE

### **ZSET有序列表**

1. ZADD setName Score Value
2. ZRANGE setName start end 按照score正序排序拿出
3. ZREVRANGE setName start end 按照score逆序排序拿出
4. ZCARD setName 获得这个set的大小
5. ZSCORE setName value 获取set中指定value的score
6. ZRANK setName value 获取set中指定value的排名
7. ZRANGEBYSCORE setName value1 value2 根据value1 和value2的值来获取列表
8. ZREM setName value 删除

## Bloom 布隆过滤器

1. bf.add key value
2. bf.exists key value
3. bf..madd key value1 value2 value3
4. bf.mexists key value1 value2 value3

## GeoHash

1. geo add key  经度 维度 value  将key对应的value的经纬度存入geo中
2. geodis key value1 value2 km  获得key中的value1和value2的距离  (距离单位可以是 m、km、ml、ft，分别代表米、千米、英里和尺)
3. geops key value  获取key-value的经纬度坐标
4. geohash key value base32的编码字符串
5. georadiusbymember key value 距离  count N asc  查询距离key-value在某个范围内的元素正排序
6. georadiusbymember key value 20 km **withcoord withdist withhash** count 3 asc  **withcoord withdist withhash附加的参数  withdist 为距离**

时间参数:

1 EXPIRE key seconds　　//将key的生存时间设置为ttl秒
2 PEXPIRE key milliseconds　　//将key的生成时间设置为ttl毫秒
3 EXPIREAT key timestamp　　//将key的过期时间设置为timestamp所代表的的秒数的时间戳
4 PEXPIREAT key milliseconds-timestamp　　//将key的过期时间设置为timestamp所代表的的毫秒数的时间戳



