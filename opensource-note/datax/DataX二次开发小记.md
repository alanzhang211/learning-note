---
title: DataX二次开发小记
date: 2018-10-17 23:17:34
tags: [2018,数据同步,DataX,大数据]
category: 大数据
---
> 本文为个人理解，如有不对之处，欢迎指正。

## 前言
之前，工作中使用datax作为数据交换组件。也简单的介绍了下datax和源码的基本导读。具体参见[DataX初探](http://alanzhang.me/2018/04/14/DataX%E5%88%9D%E6%8E%A2/#more)。数据开发平台在数据交换同步上，从sqoop、kettle等工具，慢慢地向datax并拢。

## 挑战
datax的扩展性很好，插件式安装配置。在实际使用中，往往针对实际的场景需要定制自己的读或写插件。关于如何编写插件，datax官网上也做了阐述，这里就不在赘述。详细参见：[datax插件开发](https://github.com/alibaba/DataX/blob/master/dataxPluginDev.md)。

<!--more-->

## 目标
拿项目中的一个点需求：**实现mysql的增量数据同步到hive** 来阐插件开发过程。

## 思路
1. 数据变更来源：mysql变更，通过收集binlog日志进行处理，同步到kafka中。
2. 目标数据源：hive增量分区表，通过Hbase作为中间表处理数据变更，hive建立外部表与之关联。

![数据流走向](https://github.com/alanzhang211/learning-note/blob/master/img/mysql-hive.png)

### 插件定制
1. 开发从kafka读插件。
2. 开发kafka到hbase到写插件。
3. 定制hbase读插件。
4. 开发hive写插件。

---
为什么？见下文
---

### 插件说明
#### kafka读插件
由于mysql的binlog会写入到kafka中，所以数据来源需要增加一个可以从kafka中读取的插件`kafkareader`。

**插件json定义**
```
{
  "name": "kafkareader",
  "parameter": {
    "bootstrapServers": "",
    "topic": "",
    "groupId": "",
    "decoder": "text",
    "properties": {},
    "pollTimeoutMS": 100,
    "partitionInitSeekTo": "begin",
    "startOffsets": {
      "0": 0,
      "1": 0
    },
    "endOffsets": {
      "0": 0,
      "1": 0
    }
  }
}
```

#### kafka写hbase插件
由于数据格式定制化，从公司的kafka中读取pb序列化的数据。需要解析数据加工处理。因此，在写插件中
```
public void startWriter(RecordReceiver lineReceiver)
```
方法中，读取记录处理过程
```
convertRecordToPut(Record record)
```
进行了定制处理。

#### hbase读插件
针对分库分表的情况，从kafka读取出来的消息存储与hbase中， `rowkey`的格式为`db.table.pk`。所以同步同一张mysql表，hbase的rowkey可能会出现多组。如果是每天同步，可能还会落到不同的表中。这就需要hbase读插件支持多组table，多组rowkey处理。

**原始插件json格式**

```
{
    "name": "hbase11xwriter",
    "parameter": {
        "hbaseConfig": {
            "hbase.rootdir": "",
            "hbase.cluster.distributed": "",
            "hbase.zookeeper.quorum": ""
        },
        "table": "",
        "mode": "",
        "rowkeyColumn": [
        ],
        "column": [
        ],
        "versionColumn":{
            "index": "",
            "value":""
        },
        "encoding": ""
    }
}
```

**定制版插件json定制**

```
{
    "name": "hbase11reader",
    "parameter": {
        "hbaseConfig": {},
        "tableConfig": [{
                "table": "",
                "range": [{
                        "startRowkey": "",
                        "endRowkey": ""
                    },
                    {
                        "startRowkey": "",
                        "endRowkey": ""
                    }]
        }],
        "date":"",
        "encoding": "",
        "mode": "",
        "column": [],
        "isBinaryRowkey": true
    }
}
```
对比，主要的变化就是从单表`table`变为多表`tableConfig`数组配置项。

后续，讲以此插件的定义过程，讲解插件开发思路。

#### hive写插件
由于要支持hive分区处理。所以，原生的datax实现hive的读、写，底层原理是通过直接操作hdfs的方式处理的（使用hdfsreader、hdfswriter）。

这样，就hive表的分区信息一无所知。因此，这里采用`HCatalog`作为操作hive数据的接口。关于`HCatalog`简单说明：

> HCatalog屏蔽了底层数据存储的位置格式等信息，为上层计算处理流程提供统一的、共享的metadata。并且将数据以表的形式呈现给用户（如Pig,MR,Hive,Streaming..），用户只需提供表名就可以访问底层数据，并不需要关心底层数据的位置，模式等信息。

**插件josn定义**

```
{
  "name": "hivewriter",
  "parameter": {
    "database": "",
    "table": "",
    "partition": "",
    "hadoopConf": [
      ""
    ],
    "kerberosPrincipal": "",
    "kerberosKeytab": "",
    "batchSize": 10000,
    "overwrite": false
  }
}
```
---
进入主题
---

## 插件开发过程
针对上述定制插件中的hbasereader进行讲述。针对读写插件，总结下来就是`一个配置，2个对象`。

![读写组件](https://github.com/alanzhang211/learning-note/blob/master/img/WechatIMG106.png)

### 一个配置
就是插件的json配置，最终在代码层次上会抽象为`Configuration`对象。这是任务执行的依据。

### 2个对象
1. job：作业信息载体。
2. task：作业执行载体。

### 核心方法
#### Job.split
这是一个任务切分处理逻辑，最终会讲总的json，拆分成最小的执行单元配置传递给task。

```
@Override
public List<Configuration> split(int adviceNumber) {
    return null;
}
```
针对此定制插件，就是通过split方法，将`tableConfig`拆分，退化为最原始插件的配置形式（单个table，单组rowkey）。最终，针对task而言，执行配置`split`处理逻辑也不变。

#### Task.startRead
```
@Override
    public void startRead(RecordSender recordSender) {
}
```
这个是读记录过程，也是读插件的核心。经过Job的split处理后，对于task的Configuration处理过程也是和原始的一样。这里要做的就是：**是否需要对读记录进行二次处理加工**。

### 其他
之后，就会将task提交个任务执行容器框架去处理。另外，如果需要统计处理，也可以在`Task.startRead`中调用`TaskPluginCollector`任务收集器进行统计收集。

## 结语
此次开发定制，深入到插件层，对插件的数据走向有了深入了解。同时也对很多组件（kafka、hbase、hive等）有了了解。

---
*实践出真知*
