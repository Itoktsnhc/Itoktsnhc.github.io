---
title: PartitionedContainer
date: 2021-01-19 15:15:55
tags:
---
实现一个内含多分区、自动合并分区的数据结构
<!-- more -->
## 场景
Azure Event Hub是Azure提供的PaaS服务, 类似Kafka。

![结构图](https://docs.microsoft.com/en-us/azure/event-hubs/media/event-hubs-about/event_hubs_architecture.svg)

每个ConsumerGroup的消费必须与内部的Partition严格对应。相当于从内部的Partition File顺序的读取数据。EventHub Sdk 提供了自动化的在Azure Storage Blob 中记录管理Offset的模式。使用SDK只需要在处理完成对应的EventData后 执行 UpdateCheckPointAsync即可。常见的循环 读取->处理->更新Offset->读取下一个，每次处理完成一个消息都会更新以下CheckPoint。由于AzureStorageBlob单个文件的IOPS相当有限，这个CheckPoint操作很容易成为消费端的瓶颈。  
好一点的模式是合并提交CheckPoint：  
- 1,2,3,4,**5**,6,7,8,9,**10**  

我们处理后CheckPoint的更新。每处理5个消息，更新一次CheckPoint(5,10)。   
## 需求
我们需要合并N次CheckPoint更新为一个。支持超时时间(即超过T时间不满足N个消息被处理了，也执行CheckPoint更新)

### 简单的实现

对EventHub的Offset/Id 对N取余，只有余数==0的情况下才去才去更新。无法实现超时时间。


### 另外的实现

![](/images/itok_20210119_155134.png)  
一个外层Container, Container中包含List[InternalContainer]类型 的InternalContainers,以及指向该List最后一个InternalContainer的_cur指针。每次执行插入后需要判断当前_cur.Length是否超出N，如果超出则向List中追加新的InternalContainer，同时改变_cur指向到最新。  
同时使用Timer来Merge未满但是超时的InternalContainer;

[源码](https://gist.github.com/Itoktsnhc/77a7b318334c4b862dc31768baf24253)