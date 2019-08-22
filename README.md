[![codecov](https://codecov.io/gh/alibaba/canal/branch/master/graph/badge.svg)](https://codecov.io/gh/alibaba/canal)
![maven](https://img.shields.io/maven-central/v/com.alibaba.otter/canal.svg)
![license](https://img.shields.io/github/license/alibaba/canal.svg)
[![average time to resolve an issue](http://isitmaintained.com/badge/resolution/alibaba/canal.svg)](http://isitmaintained.com/project/alibaba/canal "average time to resolve an issue")
[![percentage of issues still open](http://isitmaintained.com/badge/open/alibaba/canal.svg)](http://isitmaintained.com/project/alibaba/canal "percentage of issues still open")

## 一、前言

早期，阿里巴巴 B2B 公司因为存在杭州和美国双机房部署，存在跨机房同步的业务需求 ，主要是基于trigger的方式获取增量变更。从 2010 年开始，公司开始逐步尝试数据库日志解析，获取增量变更进行同步，由此衍生出了增量订阅和消费业务，从此开启一段新纪元。

mysql本身支持主从的【master slave】，原理：master产生的binlog日志记录了所有的增删语句，将binlog发送到slave节点，进行执行，完成数据的同步。

基于日志增量订阅和消费的业务包括
- 数据库镜像
- 数据库实时备份
- 索引构建和实时维护(拆分异构索引、倒排索引等)
- 业务 cache 刷新
- 带业务逻辑的增量数据处理

![avatar](http://baidu.com/pic/doge.png)  
![avatar](http://baidu.com/pic/doge.png)  
![avatar](http://baidu.com/pic/doge.png) 

- 1.canal伪装成slave,向Master发送dump协议
- 2.master收到dump请求，开始推送binary log给canal
- 3.canal解析binary log对象
 
---
## 二、安装canal [mac版本]
下载：https://github.com/alibaba/canal/releases
