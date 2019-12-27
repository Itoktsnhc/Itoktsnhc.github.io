---
title: System.Threading.Channels
date: 2019-12-20 10:47:42
tags: 
- "Learning"
- "C#"
- ".net"

categories:
- "Learning"
---

Channel: 一个高性能，线程安全的内存队列，可以用来实现生产者/消费者模式

<!-- more -->

``` graphviz
digraph { 
  node [shape=circle,fontsize=8,fixedsize=true,width=0.9]; 
  edge [fontsize=8]; 
  rankdir=LR;

  "queue" [shape="Mrecord" width=3 color="orange"];

  "Producer" -> "queue";
  "queue" -> "Consumer";
}
```

* 创建一个Channel
nuget包:System.Threading.Channels
static Channel class 主要包含几个方法

``` CSharp
public static Channel<T> CreateBounded<T>(int capacity) {}
public static Channel<T> CreateBounded<T>(BoundedChannelOptions options) {}
public static Channel<T> CreateUnbounded<T>() {}
public static Channel<T> CreateUnbounded<T>(UnboundedChannelOptions options) {}
```
主要区别在于CreateUnbounded创建了一个无边界(无限容量)的Channel;
CreateBounded创建的有边界(容量限制)，可以在BoundedChannelOptions中指定FullMode 来指定当Channel满了之后的行为：

``` CSharp
Channel.CreateBounded<T>(new BoundedChannelOptions(capacity)
{
    FullMode = BoundedChannelFullMode.Wait
});
```
- FullMode有以下几种模式：
    - Wait: 等待直到有空间。
    - DropNewest: 移除channel中最新的元素，让write操作完成。
    - DropOldest: 移除channel中最老的元素，让write操作完成。
    - DropWrite: 忽略这次写入操作。

Unbounded/BoundedChannelOptions中可以设置是否为多消费者/多生产者.
```CSharp
Channel.CreateUnbounded<T>(new UnboundedChannelOptions()
{
    SingleWriter = false,
    SingleReader = true
});

Channel<T> channel = Channel.CreateUnbounded<T>();
Channel.CreateUnbounded<T>(new UnboundedChannelOptions()
{
    AllowSynchronousContinuations = true//
});
ChannelReader<T> reader = channel.Reader;
ChannelWriter<T> writer = channel.Writer;
```

* ChannelReader：
    - bool TryRead(out T item)	
    - ValueTask<T> ReadAsync()	
    - ValueTask<bool> WaitToReadAsync()	
    - Task Completion	
* ChannelWriter：
    - bool TryWrite(T item)
    - ValueTask WriteAsync(T item)
    - ValueTask<bool> WaitToWriteAsync()
    - bool TryComplete() 
    - void Complete()

* 关于ValueTask和Task的区别：[Understanding the Whys, Whats, and Whens of ValueTask](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/)

* TryRead / TryWrite
- 同步方法，主要用来尝试写入/读取channel，当channel为unbounded时，TryWrite永远不会为false，除非channel被关闭。

- ReadAsync / WriteAsync
    

- WaitToReadAsync / WaitToWriteAsync





