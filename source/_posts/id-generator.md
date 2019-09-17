---
title: Id生成器的调查
date: 2019-09-12 15:17:02
tags: 
- "Snowflake"
- "Azure Service Bus"
- "Azure Sql database"
- "Queue"
categories:
- "Learning"
---

几种常见的ID生成器
<!-- more -->

# SQL SEQUENCE
大部分的数据库都提供了sequence功能，能够提供自增功能：
``` SQL
-- Create
CREATE SEQUENCE IdGenerator as bigint
    START WITH 1
    INCREMENT BY 1;
-- Get value
SELECT NEXT VALUE FOR IdGenerator

-- delete sequence
DROP SEQUENCE IdGenerator
```
- Pros and cons
    - Pros
        - 基于数据库、比较稳定、emmmm 没了
    - Cons
        - 性能问题
        - 额外的服务调用（延迟问题）
        - 贵

# ID池形式（队列、生产-消费模式）

可以使用 Azure Service Bus 、Azure Event Hub等消息属性中带有sequenceNumber的服务来实现。
- 生产者: 按照一定周期检查队列中的消息数量，如果低于某一个数值，则开始向队列中发送消息。直到到达某个上限。
- 消费者: 当有获取ID的请求时，从队列中出队最新的消息，将消息的SequenceNumber作为Id返回

- Pros and Cons
    - Pros 
        - 性能比较好
        - 和数据库相比，队列服务一般性价比较高
    - Cons
        - 性能不够高
        - 额外的队列服务费用
        - 额外的服务调用（延迟问题）


# 以SnowFlake为代表的time-based 生成器

- Pros and Cons
    - Pros 
        - 性能相当较好
        - 无需额外组件，没有额外调用
    - Cons
        - 基于时间，如果如果发生问题回拨、会发生重复。
            - 时间回拨如果发生，可以用类似workerId池的方式更换WorkerId解决。
        - 数字过大
            - 典型的问题，和Js的集成需要特殊处理。


# 改进的ID池形式【需要单例，分布式可以使用Orleans类似框架支持】

1. 每次程序启动后，从Storage（可以是blob、s2、oos、数据库等）中获取上一次的checkPoint，然后从该Checkpoint 向后取一个数字的范围作为IdPool（如10-10000）。将偏移以后的数值作为CheckPoint写入Storage。
2. 当需要获取Id的时候，从IdPool这个范围中取。
3. 当IdPool的范围小于某个值的时候，重复1的动作以扩充IdPool池

Pros and Cons
    - Pros 
        - 仅在池不足的时候需要访问存储，对外部依赖极小
        - 性能很好，基本就是内存操作
    - Cons
        - Id生成服务需要单例，可能存在单点故障的情况。可以用类似Orleans等框架避免