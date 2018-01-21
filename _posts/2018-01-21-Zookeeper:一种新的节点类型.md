---
layout: post
published: true
title: Zookeeper:一种新的节点类型
mathjax: false
featured: true
comments: false
categories:
  - zookeeper
tags: zookeeper
---

# Zookeeper:一种新的节点类型

Zookeeper最初发布的时候，可以创建以下几种节点类型，也各自有不同的用途，网上目前资料挺多

- 持久节点
- 持久顺序节点
- 临时节点
- 临时顺序节点

最近在阅读源码的时候发现处理Zookeeper在3.5.1以后增又陆续加了以下几种类型，下面主要介绍下容器节点这种类型出现的原因，及实现过程。

- ***容器节点***
- 带生存时间的持久节点
- 带生存时间的持久顺序节点

## 问题背景
在使用Zookeeper过程中，一个问题就是无用父节点的回收删除问题。例如在使用组管理、主节点选取时都需要创建父节点，然后在此节点下不同的参与者(主机、应用、服务、)创建挂载各自对应的Znode，当任务结束后，挂载在父节点下临时节点随着Session的结束都会被删除掉，但这些父节点会一直存在。渐渐的，Zookeeper服务器就会充斥这大量这种没有子节点的无用的节点，可能会影响Zookeeper服务器的稳定性。 而之前Zookeeper API并没有提供一种机制来清理这些无子节点。

## 解决方案
在Zookeeper实现容器节点之前，Apache Curator有一种变通的解决方案，主要是提供了类Reaper来负责查找并删除这些无孩子的父节点

容器节点的的主要特点：当其下挂载的最后一个孩子删除后，其自身也会在某个时刻被删除掉。当新创建一个容器节点，它还没挂载过任何子节点的话，不会被删除。

## 实现过程

### 客户段

####  Zookeeper API

没有新增类似这种的```zkclient.createContainer()```的方法,而是新增了一种节点的创建类型```CreateMode.Container ```，复用之前的```zkclient.create()```方法，简化API使用，并且可以支持multitransaction/batch操作

#### CREATE 请求的封装与发送
```Zookeeper```请求一般由两部分组成，```RequestHeader```与```Record```(可以理解成```RequestBody```)，如```CreateRequest, CreateTTLRequest```等.

创建Container Znode时，根据```CreateMode```设置下```RequestHeader```的类型,其它同创建基本节点

```
private void setCreateHeader(CreateMode createMode, RequestHeader h) {
        if (createMode.isTTL()) {
            h.setType(ZooDefs.OpCode.createTTL);
        } else {
            h.setType(createMode.isContainer() ? ZooDefs.OpCode.createContainer : ZooDefs.OpCode.create2);
        }
    }
```

### 服务端 `TODO待完善`
服务端收到请求后，将请求交给处理器链

#### 请求的封装、处理、应答

* 请求的封装转化(`Request->transaction`)新增事务类型`CreateContainerTxn`
`PrepRequestProcessor`在处理请求...



* 事务的处理
    ```DataTree.processTxn(TxnHeader header, Record txn) ```



#### 节点的定时清理：ContainerManager

* 构造一个定时执行的任务任务

```
    public void start() {
        if (task.get() == null) {
            TimerTask timerTask = new TimerTask() {
                @Override
                public void run() {
                    try {
                        checkContainers();
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        LOG.info("interrupted");
                        cancel();
                    } catch ( Throwable e ) {
                        LOG.error("Error checking containers", e);
                    }
                }
            };
            if (task.compareAndSet(null, timerTask)) {
                timer.scheduleAtFixedRate(timerTask, checkIntervalMs,
                        checkIntervalMs);
            }
        }
    }

```
* 检查清理过程

```
    public void checkContainers()
            throws InterruptedException {
        long minIntervalMs = getMinIntervalMs();
        for (String containerPath : getCandidates()) {
            long startMs = Time.currentElapsedTime();

            ByteBuffer path = ByteBuffer.wrap(containerPath.getBytes());
            Request request = new Request(null, 0, 0,
                    ZooDefs.OpCode.deleteContainer, path, null);
            try {
                LOG.info("Attempting to delete candidate container: %s",
                        containerPath);
                requestProcessor.processRequest(request);
            } catch (Exception e) {
                LOG.error(String.format("Could not delete container: %s" ,
                        containerPath), e);
            }

            long elapsedMs = Time.currentElapsedTime() - startMs;
            long waitMs = minIntervalMs - elapsedMs;
            if (waitMs > 0) {
                Thread.sleep(waitMs);
            }
        }
    }
```



## 反思

从上面过程分析可以参考下，如果自己要新添加一种节点类型，需要做那些工作。
