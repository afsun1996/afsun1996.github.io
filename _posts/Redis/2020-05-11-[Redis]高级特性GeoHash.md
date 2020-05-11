---
title: "「Redis」Redis高级特性GeoHash"
subtitle: ""
layout: post
author: "afsun"
header-style: text
hidden: true
tags:
  - Redis
---
# GeoHash

- 应用场景:附近的人  附件的一些元素,用作查询附件的元素的一种功能,在redis3.2上引入

在Redis中GeoHash类型可以看做一个set类型

## 算法

将整个地球展开为一个二维平面,然后开始切分第一次将地图分成两半0和1 第二次分为四份 00 01 10 11

这样一直切分下去,地图会被切分成越来越小的块组成的地图,那么每小块的精确度就越高

示例:

![GeoHash%20b6c943c8abd64fe1952a2f6f20ebef40/Untitled.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/11/091600-667095.png)

最后得出维度的二进制为 10111000110001111001

同理得出维度的二进制 11010010110001000100

经纬度合并:11100 11101 00100 01111 00000 01101 01011 00001  (经读在偶数位 纬度在奇数位)

再将二进制数据用Base32转换为最终值:wx4g0ec1

在redis中,会分别对经度和维度进行26此切分,最后融合经纬度得到一个52位的比特位,通过base32进行处理,得到一个11为长度的字符串,

将52位比特的数值当做score插入到zset中,之后的操作就跟操作zset类似了

## 特点

- 用一个字符来代替精度和纬度,在数据库上可以实现一列上应用索引
- GeoHash不是一个点,而是一个区域
- 在redis中使用的话,最好只用单机redis来存储geohash数据
- 编码越长表示的范围约小,就越精确

![GeoHash%20b6c943c8abd64fe1952a2f6f20ebef40/Untitled%201.png](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/11/091602-805451.png)