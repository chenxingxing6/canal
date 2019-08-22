[![codecov](https://codecov.io/gh/alibaba/canal/branch/master/graph/badge.svg)](https://codecov.io/gh/alibaba/canal)
![maven](https://img.shields.io/maven-central/v/com.alibaba.otter/canal.svg)
![license](https://img.shields.io/github/license/alibaba/canal.svg)
[![average time to resolve an issue](http://isitmaintained.com/badge/resolution/alibaba/canal.svg)](http://isitmaintained.com/project/alibaba/canal "average time to resolve an issue")
[![percentage of issues still open](http://isitmaintained.com/badge/open/alibaba/canal.svg)](http://isitmaintained.com/project/alibaba/canal "percentage of issues still open")

## 一、前言(个人学习笔记)

早期，阿里巴巴 B2B 公司因为存在杭州和美国双机房部署，存在跨机房同步的业务需求 ，主要是基于trigger的方式获取增量变更。从 2010 年开始，公司开始逐步尝试数据库日志解析，获取增量变更进行同步，由此衍生出了增量订阅和消费业务，从此开启一段新纪元。

mysql本身支持主从的【master slave】，原理：master产生的binlog日志记录了所有的增删语句，将binlog发送到slave节点，进行执行，完成数据的同步。

基于日志增量订阅和消费的业务包括
- 数据库镜像
- 数据库实时备份
- 索引构建和实时维护(拆分异构索引、倒排索引等)
- 业务 cache 刷新
- 带业务逻辑的增量数据处理

![avatar](https://raw.githubusercontent.com/chenxingxing6/canal/master/mytest/img1.png)  
![avatar](https://raw.githubusercontent.com/chenxingxing6/canal/master/mytest/img2.png)  
![avatar](https://raw.githubusercontent.com/chenxingxing6/canal/master/mytest/img3.png) 

- 1.canal伪装成slave,向Master发送dump协议
- 2.master收到dump请求，开始推送binary log给canal
- 3.canal解析binary log对象
 
---
## 二、安装canal [mac版本]
下载：https://github.com/alibaba/canal/releases   

![avatar](https://raw.githubusercontent.com/chenxingxing6/canal/master/mytest/img4.png)

##### 2.1解压
![avatar](https://raw.githubusercontent.com/chenxingxing6/canal/master/mytest/img5.png)

##### 2.2修改配置
```html
vim conf/examples/instance.properties
#一定要注释掉下面这个参数，这样就会扫描全库 
#canal.instance.defaultDatabaseName = test
```
![avatar](https://raw.githubusercontent.com/chenxingxing6/canal/master/mytest/img6.png)

##### 2.3启动
找到canal.instance.parser.parallelThreadSize = 16注释掉，java才能启动成功  
![avatar](https://raw.githubusercontent.com/chenxingxing6/canal/master/mytest/img7.png)

---

## 三、Java代码测试
![avatar](https://raw.githubusercontent.com/chenxingxing6/canal/master/mytest/img8.png)

![avatar](https://raw.githubusercontent.com/chenxingxing6/canal/master/mytest/img9.png)


https://github.com/chenxingxing6/canal/blob/master/example/src/main/java/com/demo/my/MyTest.java
```java
package com.example.canal;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.alibaba.otter.canal.client.CanalConnector;
import com.alibaba.otter.canal.client.CanalConnectors;
import com.alibaba.otter.canal.common.utils.AddressUtils;
import com.alibaba.otter.canal.protocol.CanalEntry;
import com.alibaba.otter.canal.protocol.Message;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.net.InetSocketAddress;
import java.net.SocketAddress;
import java.util.List;
import java.util.concurrent.TimeUnit;

/**
 * User: lanxinghua
 * Date: 2019/8/22 16:24
 * Desc: canal测试
 */
public class SimpleCanalTest {
    private static final Logger logger = LoggerFactory.getLogger("canal");

    public static void main(String[] args) {
        test01();
    }

    public static void test01(){
        new Thread(() -> {
            // 创建连接
            SocketAddress socketAddress = new InetSocketAddress(AddressUtils.getHostIp(), 11111);
            CanalConnector connector = CanalConnectors.newSingleConnector(socketAddress, "example", "", "");

            int batchSize = 1000;
            try {
                System.out.println("监听......");
                connector.connect();
                connector.subscribe(".*\\..*");
                connector.rollback();
                while (true){
                    // 获取指定的数据
                    Message msg = connector.getWithoutAck(batchSize);
                    long batchId = msg.getId();
                    int size = msg.getEntries().size();
                    if (batchId != -1 && size > 0){
                        printEntry(msg.getEntries());
                    }
                    connector.ack(batchId);
                    TimeUnit.SECONDS.sleep(1);
                }
            }catch (Exception e){
                logger.error("线程异常", e);
            }finally {
                connector.disconnect();
            }
        }).start();
    }

    private static void printEntry(List<CanalEntry.Entry> entrys) {
        for (CanalEntry.Entry entry : entrys) {
            // 操作事务忽略
            if (entry.getEntryType() == CanalEntry.EntryType.TRANSACTIONBEGIN || entry.getEntryType() == CanalEntry.EntryType.TRANSACTIONEND) {
                continue;
            }
            // System.out.println(entry.toString());

            CanalEntry.RowChange rowChage = null;  // 执行事件信息
            String database = null; // 数据库
            String table = null;    // 执行的表
            try {
                rowChage = CanalEntry.RowChange.parseFrom(entry.getStoreValue());
                database = entry.getHeader().getSchemaName();
                table = entry.getHeader().getTableName();
            } catch (Exception e) {
                logger.error("获取数据失败", e);
            }

            // 获取执行的事件
            CanalEntry.EventType eventType = rowChage.getEventType();
            for (CanalEntry.RowData rowData : rowChage.getRowDatasList()) {
                // 删除
                if (eventType == CanalEntry.EventType.DELETE) {
                    delete(rowData.getBeforeColumnsList(), database, table);
                }
                // 新增
                else if (eventType == CanalEntry.EventType.INSERT) {
                    insert(rowData.getAfterColumnsList(), database, table);
                }
                // 修改
                else if (eventType == CanalEntry.EventType.UPDATE){
                    update(rowData.getAfterColumnsList(), database, table);
                }
                // 改表结构
                else if (eventType == CanalEntry.EventType.ALTER){
                    System.out.println("修改表结构");
                }
                else {
                    System.out.println("类型不匹配");
                }
            }
        }
    }

    /**
     * 数据库执行了添加操作
     * @param columns
     * @param database
     * @param table
     */
    private static void insert(List<CanalEntry.Column> columns, String database, String table){
        JSONObject json=new JSONObject();
        for (CanalEntry.Column column : columns) {
            json.put(column.getName(), column.getValue());
        }
        System.out.println("数据库："+database+"==>表："+table+"==>添加数据："+JSON.toJSONString(json));
    }

    /**
     * 数据库执行了修改操作
     * @param columns
     * @param database
     * @param table
     */
    private static  void update( List<CanalEntry.Column> columns,String database,String table){
        JSONObject json=new JSONObject();
        for (CanalEntry.Column column : columns) {
            json.put(column.getName(), column.getValue());
        }
        System.out.println("数据库："+database+"==>表："+table+"==>修改数据："+JSON.toJSONString(json));
    }

    /**
     * 数据库执行了删除操作
     * @param columns
     * @param database
     * @param table
     */
    private static  void delete( List<CanalEntry.Column> columns,String database,String table){
        JSONObject json=new JSONObject();
        for (CanalEntry.Column column : columns) {
            json.put(column.getName(), column.getValue());
        }
        System.out.println("数据库："+database+"==>表："+table+"==>删除数据："+JSON.toJSONString(json));
    }
}
```
![avatar](https://raw.githubusercontent.com/chenxingxing6/canal/master/mytest/img10.png)

```html
1.Connection获取上一次解析成功的位置（如果第一次启动，则获取初始制定的位置或者是当前数据库的binlog位点）  
2.Connection建立连接，发生BINLOG_DUMP命令  
3.Mysql开始推送Binary Log  
4.接收到的Binary Log通过Binlog parser进行协议解析，补充一些特定信息  
5.传递给EventSink模块进行数据存储，是一个阻塞操作，直到存储成功   
6.存储成功后，定时记录Binary Log位置  
```
![avatar](https://raw.githubusercontent.com/chenxingxing6/canal/master/mytest/img11.png)


