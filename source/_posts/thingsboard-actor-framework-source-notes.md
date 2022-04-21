---
title: Thingsboard Actor框架源码笔记
tags: JAVA, thingsboard
date: 2022-04-21 09:21:22
---


## 背景

Thingsboard是一个开源可扩展高性能的分布式IoT平台。其代码中大量使用了Actor模型，处理设备消息及执行规则引擎。

本文基于Thingsboard v3.3.1源码，分析其实现的Actor模型。

## Thingsboard Actor System

### 源码位置

Actor框架代码（基类及接口）位于项目中`/common/actor/`路径下
Actor具体业务 (实现类) 位于在`/application`

### 作用及承载的业务


TB中引入Actor模型主要解决了流处理中，实时性，高并发，有序处理的特性。

### 概览

TB的Ator系统将Actor模型定义成两部分接口Actor和ActorRef

- `Actor`接口定义了Actor的生命周期行为
- `ActorRef`则定义了向Actor实例传递消息的能力

```
+----------------------------------------------------+
| interface TbActor                                  |
|----------------------------------------------------|
| +void destroy()                                    |  声明周期
| +void init(TbActorCtx)                             |  生命周期
| +InitFailureStrategy onInitFailure(int, Throwable) |  初始化失败的策略
| +ProcessFailureStrategy onProcessFailure(Throwable)|  处理消息失败的策略
| +boolean process(TbActorMsg)                       |  核心：  处理消息的逻辑
| +ActorRef getActorRef()                            |  暴露向当前Actor传递消息的能力
+----------------------------------------------------+

+----------------------------------------------------+
| abstract class AbstractTbActor                     |  抽象基类，引入了子类可访问的成员ctx
|----------------------------------------------------|  子类可通过该成员与ActorSystem交互
| constructor AbstractTbActor()                      |
|----------------------------------------------------|
| void init(TbActorCtx)                              |
| ActorRef getActorRef()                             |
|----------------------------------------------------|
| TbActorCtx ctx                                     |
+----------------------------------------------------+

(Appliacation)
+----------------------------------------------------+
| abstact class ContextAwareActor                    |  应用层中的抽象类, 核心引入了业务层上下文
|----------------------------------------------------|
| constructor ContextAwareActor(ActorSystemContext)  |
| +boolean doProcess(TbActorMsg)                     |
| *ProcessFailureStrategy doProcessFailure(Throwable)|
| ProcessFailureStrategy onProcessFailure(Throwable) |
| boolean process(TbActorMsg)                        |
+----------------------------------------------------+
| *ActorSystemCtx systemCtx                          |
+----------------------------------------------------+
```

```
+----------------------------------------------------+
| interface TbActorRef                               | TbActorRef引用 对外暴露，可以通过持有该实例向该Actor发送消息
|----------------------------------------------------|
| +void tell(ActorMsg)                               |
| +void tellWithHighPriority(TbActorMsg)             |
| +TbActorId getActorId()                            |
+----------------------------------------------------+

+-----------------------------------------------------------------------------------------+
| interface TbActorCtx                                                                    | ActorSystem和Actor相互暴露能力
|-----------------------------------------------------------------------------------------|
| +void broadcastToChildren(ActorMsg)                                                     |
| +void broadcastToChildren(ActorMsg, Predicate<TbActorId>)                               |
| +TbActorRef getOrCreateChildActor(TbActorId, Supplier<String>, Supplier<TbActorCreator>)|
| +void stop(ActorId)                                                                     |
| +void tell(TbActorId, TbActorMsg)                                                       |
| +TbActorId getSelf()                                                                    |
| +TbActorRef getParentRef()                                                              |
+-----------------------------------------------------------------------------------------+

+-----------------------------------------------------------------------------------------+
| class TbActorMailbox                                                                    | 逻辑实现
|-----------------------------------------------------------------------------------------|
| ...                                                                                     |
+-----------------------------------------------------------------------------------------+
```

### Context?

在类图的说明，我们可以看到TB的Actor System中包含两个Context。
一个是Actor模块中的`TbActorCtx`，一个是Application模块中的Srping Bean(后简称为Bean)——`ActorSystemContext`。

两者有什么区别呢?

`TbActorCtx`是用于ActorSystem中的， 上指ActorSystem接口，下指TbActor。根据其定义接口也可以发现，该上下文主要提供了如下方法：

 - Actor向其他Actor传递消息
 - Parent Actor告诉ActorSystem去创建或销毁子Child Actor

结合实现类的代码我们也可以发现：消息的处理在TbActor#process完成，消息的传递主要由ActorSystem完成，同时也通过TbActorCtx的方法简洁暴露给TbActor调用。

第二个Context是`ActorSystemContext`。前面也提到了这是一个Bean。

观察这个类的里的方法，我们会发现其中注入了大量业务系统的Bean，除此之外会会发现tell及tellWithHighPriority这两个方法。
是不是很眼熟？在看下实现，调用了ActorSystem#tell，并指定发给了RootActor

所以`ActorSystemContext`是一个连接业务系统和Actor系统的上下文，向ActorSystem暴露的业务能力(CRUD能力), 同时向业务系统提供向ActorSystem发送消息的能力

### Actor System

Actor系统接口，`org.thingsboard.server.actors.TbATbActorSystem`

对外部暴露如下特性:

- 执行线程池，并维护这些线程池的生命周期
- Actor系统交互能力(P2P 或 广播)
- 创建Actor实例
- 销毁Actor实例

```java
public interface TbActorSystem {

    /** 资源维护 **/

    // 创建 销毁 线程池
    ScheduledExecutorService getScheduler();

    void createDispatcher(String dispatcherId, ExecutorService executor);

    void destroyDispatcher(String dispatcherId);

    /** 获取实例 **/

    // 获取指定Actor的引用

    TbActorRef getActor(TbActorId actorId);

    List<TbActorId> filterChildren(TbActorId parent, Predicate<TbActorId> childFilter);

    /** 生命周期维护 **/
    // 创建Actor

    TbActorRef createRootActor(String dispatcherId, TbActorCreator creator);

    TbActorRef createChildActor(String dispatcherId, TbActorCreator creator, TbActorId parent);

    // 销毁Actor
    void stop(TbActorRef actorRef);

    void stop(TbActorId actorId);

    // 销毁Actor系统
    void stop();

    /** 传递消息 **/
    // Point to point
    void tell(TbActorId target, TbActorMsg actorMsg);

    void tellWithHighPriority(TbActorId target, TbActorMsg actorMsg);

    // 广播消息
    void broadcastToChildren(TbActorId parent, TbActorMsg msg);

    void broadcastToChildren(TbActorId parent, Predicate<TbActorId> childFilter, TbActorMsg msg);
}
```

实现类: `org.thingsboard.server.actors.DefaultTbActorSystem`


### Actor

TB中Actor实例被抽象成了两部分
1. 维护生命周期及处理消息
2. 维护Actor模型内部收发消息的能力

### 生命周期

```java
// 1. 创建Actor实例
TbActor actor = TbActorCreator.createActor();

// 2-1.  创建ActorRef, TbActorMailBox时TbActorRef的实现类
TbActorMailBox ref = new TbActorMailBox(actor, /* args */);

// 2-2. TbActorMailBox#initActor()方法会调用 TbActor#init(TbActorRef)
ref.initActor();

// 3. 销毁实例
actor.destory
```

生命周期： 初始化及销毁方法

```java
package org.thingsboard.server.actors;

import org.thingsboard.server.common.msg.TbActorMsg;

public interface TbActor {

    /** 处理消息 指业务逻辑 **/
    boolean process(TbActorMsg msg);

    /** 获取引用 **/
    TbActorRef getActorRef();

    /** 生命周期 **/

    default void init(TbActorCtx ctx) throws TbActorException {
    }

    default void destroy() throws TbActorException {
    }

    default InitFailureStrategy onInitFailure(int attempt, Throwable t) {
        return InitFailureStrategy.retryWithDelay(5000L * attempt);
    }

    default ProcessFailureStrategy onProcessFailure(Throwable t) {
        if (t instanceof Error) {
            return ProcessFailureStrategy.stop();
        } else {
            return ProcessFailureStrategy.resume();
        }
    }
}

```

构造类, 提供构造接口

```java
public interface TbActorCreator {

    TbActorId createActorId();

    TbActor createActor();

}
```

### 消息传递

相关逻辑见
- `org.thingsboard.server.actors.TbActorRef`
- `org.thingsboard.server.actors.TbActorCtx`
- `org.thingsboard.server.actors.TbActorMailbox`

```
|TbActorRef|  接口：  定义消息交互能力 tell(TbActorMsg) 表示向当前Actor传递消息
     ||              提供基础能力，对外部系统提供Actor系统交互能力
     ||
     \/
|TbActorCtx|  接口：  定义消息交互能力 tell(TbActorRef, TbActorMsg) 表示向其他TbActorRef传递消息
     ||              该上下文用于TbActor及TbActorSystem的提供公共功能
     ||
     \/
|TbActorMailbox| 类:  定义消息传递的流程，控制触发消息业务处理功能( TbActor#process() )
```

#### TbActorRef

```java
public interface TbActorRef {

    TbActorId getActorId();

    /** 讲消息传递给当前Actor **/

    void tell(TbActorMsg actorMsg);

    void tellWithHighPriority(TbActorMsg actorMsg);

}
```

#### TbActorCtx

```java
public interface TbActorCtx extends TbActorRef {

    TbActorId getSelf();

    TbActorRef getParentRef();

    // 向其他Actor发送消息
    void tell(TbActorId target, TbActorMsg msg);

    void stop(TbActorId target);

    TbActorRef getOrCreateChildActor(TbActorId actorId, Supplier<String> dispatcher, Supplier<TbActorCreator> creator);

    void broadcastToChildren(TbActorMsg msg);

    void broadcastToChildren(TbActorMsg msg, Predicate<TbActorId> childFilter);

    List<TbActorId> filterChildren(Predicate<TbActorId> childFilter);
}
```

#### TbActorMailBox 实现类

```java
public final class TbActorMailbox implements TbActorCtx {

    /** A 成员 **/

    // 常量
    private static final boolean HIGH_PRIORITY = true;
    private static final boolean NORMAL_PRIORITY = false;

    private static final boolean FREE = false;
    private static final boolean BUSY = true;

    private static final boolean NOT_READY = false;
    private static final boolean READY = true;
    // 常量

    private final TbActorSystem system;
    private final TbActorSystemSettings settings;
    private final TbActorId selfId;
    private final TbActorRef parentRef;
    private final TbActor actor;
    private final Dispatcher dispatcher;

    // tell方法非线程安全，使用线程安全的Concurrent包,使用LinkedQueue保证消息顺序
    private final ConcurrentLinkedQueue<TbActorMsg> highPriorityMsgs = new ConcurrentLinkedQueue<>();
    private final ConcurrentLinkedQueue<TbActorMsg> normalPriorityMsgs = new ConcurrentLinkedQueue<>();

    // 线程安全的状态机
    // 判断是否正在处理消息
    private final AtomicBoolean busy = new AtomicBoolean(FREE);
    // 判断Mailbox是否初始化完成
    private final AtomicBoolean ready = new AtomicBoolean(NOT_READY);
    private final AtomicBoolean destroyInProgress = new AtomicBoolean();
    private volatile TbActorStopReason stopReason;

    /** 初始化 **/

    public void initActor() {
        // init非阻塞，使用线程池执行
        dispatcher.getExecutor().execute(() -> tryInit(1));
    }

    private void tryInit(int attempt) {
        try {
            log.debug("[{}] Trying to init actor, attempt: {}", selfId, attempt);
            if (!destroyInProgress.get()) {
                actor.init(this);
                if (!destroyInProgress.get()) {
                    ready.set(READY);
                    // 防止初始化时， 有消息入队
                    tryProcessQueue(false);
                }
            }
        } catch (Throwable t) {
            /** 初始化失败 **/
            log.debug("[{}] Failed to init actor, attempt: {}", selfId, attempt, t);
            int attemptIdx = attempt + 1;
            InitFailureStrategy strategy = actor.onInitFailure(attempt, t);
            if (strategy.isStop() || (settings.getMaxActorInitAttempts() > 0 && attemptIdx > settings.getMaxActorInitAttempts())) {
                log.info("[{}] Failed to init actor, attempt {}, going to stop attempts.", selfId, attempt, t);
                stopReason = TbActorStopReason.INIT_FAILED;
                destroy();
            } else if (strategy.getRetryDelay() > 0) {
                log.info("[{}] Failed to init actor, attempt {}, going to retry in attempts in {}ms", selfId, attempt, strategy.getRetryDelay());
                log.debug("[{}] Error", selfId, t);
                system.getScheduler().schedule(() -> dispatcher.getExecutor().execute(() -> tryInit(attemptIdx)), strategy.getRetryDelay(), TimeUnit.MILLISECONDS);
            } else {
                log.info("[{}] Failed to init actor, attempt {}, going to retry immediately", selfId, attempt);
                log.debug("[{}] Error", selfId, t);
                dispatcher.getExecutor().execute(() -> tryInit(attemptIdx));
            }
            /** 初始化失败 **/
        }
    }


    /** 发送消息, 核心逻辑的入口 **/

    @Override
    public void tell(TbActorMsg actorMsg) {
        enqueue(actorMsg, NORMAL_PRIORITY);
    }

    @Override
    public void tellWithHighPriority(TbActorMsg actorMsg) {
        enqueue(actorMsg, HIGH_PRIORITY);
    }

    // 入队
    private void enqueue(TbActorMsg msg, boolean highPriority) {
        if (!destroyInProgress.get()) {
            /** 消息入队 **/
            if (highPriority) {
                highPriorityMsgs.add(msg);
            } else {
                normalPriorityMsgs.add(msg);
            }
            /*  消息处理核心方法 */
            tryProcessQueue(true);
        } else {
            /* Actor实例销毁时， 处理逻辑 */
            if (highPriority && msg.getMsgType().equals(MsgType.RULE_NODE_UPDATED_MSG)) {
                synchronized (this) {
                    if (stopReason == TbActorStopReason.INIT_FAILED) {
                        destroyInProgress.set(false);
                        stopReason = null;
                        initActor();
                    } else {
                        msg.onTbActorStopped(stopReason);
                    }
                }
            } else {
                msg.onTbActorStopped(stopReason);
            }
        }
    }

    // 线程安全 使用cas锁保证
    // 核心业务 该业务在调用actor#tell方法的线程
    private void tryProcessQueue(boolean newMsg) {
        // 若未初始化完成，则会在initActor方法中执行tryProcessQueue
        if (ready.get() == READY) {
            // 两种情况
            // 1. 新消息入队(newMsg == true)， 则不判断消息队列是否为空, 则触发竞争CAS
            // 2. 非新消息入队(processMailbox中noMoreElements为true), 判断队列里是否有消息
            // 2-1. 如果消息队列有消息，则会触发竞争CAS
            if (newMsg || !highPriorityMsgs.isEmpty() || !normalPriorityMsgs.isEmpty()) {

                if (busy.compareAndSet(FREE, BUSY)) {
                    // 获取锁， 并开始异步执行
                    dispatcher.getExecutor().execute(this::processMailbox);
                } else {
                    log.trace("[{}] MessageBox is busy, new msg: {}", selfId, newMsg);
                }
            } else {
                log.trace("[{}] MessageBox is empty, new msg: {}", selfId, newMsg);
            }
        } else {
            log.trace("[{}] MessageBox is not ready, new msg: {}", selfId, newMsg);
        }
    }

    // 该方法线程不安全，同一时刻同一个actor仅会存在一个proccessMailbox任务,该方法必定在
    // 核心业务
    private void processMailbox() {
        boolean noMoreElements = false;
        // 从两个linkedQueue里poll消息，并处理
        // 处理到最大值(ActorThroughput)或队列中无消息时 停止本次处理
        // 设置Throughout可以防止一个actor占用cpu时间过长
        for (int i = 0; i < settings.getActorThroughput(); i++) {
            TbActorMsg msg = highPriorityMsgs.poll();
            if (msg == null) {
                msg = normalPriorityMsgs.poll();
            }
            if (msg != null) {
                try {
                    log.debug("[{}] Going to process message: {}", selfId, msg);

                    // 真正处理消息的地方, 业务异常应该在process实现中catch
                    actor.process(msg);
                    // 真正处理消息的地方

                } catch (TbRuleNodeUpdateException updateException){
                    /** 规则引擎的逻辑 ignore **/
                    stopReason = TbActorStopReason.INIT_FAILED;
                    destroy();
                    /** 规则引擎的逻辑 ignore **/
                } catch (Throwable t) {
                    /** 消息消费失败处理 **/
                    log.debug("[{}] Failed to process message: {}", selfId, msg, t);
                    ProcessFailureStrategy strategy = actor.onProcessFailure(t);
                    if (strategy.isStop()) {
                        system.stop(selfId);
                    }
                    /** 消息消费失败处理 **/
                }
            } else {
                // 如果消息为空则 设置标记位
                noMoreElements = true;
                break;
            }
        }


        if (noMoreElements) {
            // 该标记位 为 True时
            // 释放busy标记，
            busy.set(FREE);
            dispatcher.getExecutor().execute(() -> tryProcessQueue(false));
        } else {
            // 不释放busy标记, 则新消息入队时，不会执行tryProcessQueue方法
            // 创建新的processMailbox任务(释放cpu，稍后再次消费，配合ActorThoughout)
            dispatcher.getExecutor().execute(this::processMailbox);
        }
    }

    /** 发送消息给其他Actor，调用了TbActorSystem#xxxx的方法 **/

    @Override
    public void tell(TbActorId target, TbActorMsg actorMsg) {
        system.tell(target, actorMsg);
    }

    @Override
    public void broadcastToChildren(TbActorMsg msg) {
        system.broadcastToChildren(selfId, msg);
    }

    @Override
    public void broadcastToChildren(TbActorMsg msg, Predicate<TbActorId> childFilter) {
        system.broadcastToChildren(selfId, childFilter, msg);
    }

    /** fin **/

}

```

