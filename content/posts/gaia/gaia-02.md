---
title: "0.2 从 DBCache 到 JanusThreadManager：用 Actor模型 隐藏多线程开发的复杂度"
date: 2026-07-14
categories: ["Gaia系列"]
tags: ["Gaia", "C++", "架构设计", "笔记"]
draft: false
image: "https://cdn.booljin.top/images/gaia-logo.svg"
---

## 0.2.1 不是所有开发人员都能搞定多线程
多线程开发实践中，很多团队的典型做法是仔细甄别所有 *被竞争的资源*，然后加锁保护。一旦处理不慎，不是数据竞争就是死锁，很多宕机都源于此。而规模较大的项目，排查此类问题需要抽丝剥茧，需要较高的代码掌控力，低水平团队不是终日被崩溃折磨，就是彻底抛弃并发优势，退回到单线程编程。

<!--more-->

大多数开发人员，根本搞不清楚自己编写的每一行代码将会运行在哪个线程上，但他们可以准确的将需求翻译成逻辑代码。所以很多技术都致力于将并发编程的复杂度隐藏起来，让普通开发人员专心开发逻辑。笔者原本作为游戏团队主程，这也是我的本职之一：让普通开发人员专注于用 Lua 编写逻辑，剩下的我来搞定。受到启发最多的，是云风的 **skynet** 项目，其中的 service 使用的就是 Actor模型。

## 0.2.2 从 DBCache 到 harborRPC，工业领域的落地尝试
笔者曾经开发过一个数据报表系统。交付时，负责代码审核的同事问了一句，为什么你的代码没有锁？于是笔者跟他解释了整个设计：
* 整个系统有一个独立的线程，维护一个独立的服务
* 外部的请求都会发送到消息队列中
* service 线程不断从消息队列中获取任务并执行。因为只有一个工作线程，所以所有业务从入队那一刻起就是有序的，且没有任何竞争，所以所有业务自然都没有锁
* work thread 如果在一个循环周期里连续处理了多条消息，需要主动释放时间片（sleep(0)即可）
* 整个系统，只有消息队列是有锁的

基于这套设计，整个 DBCache 成为一个独立的服务：它通过消息队列接收外部任务，自主执行，自主管理线程状态。

<figure class="img-figure img-original">
  <img src="https://cdn.booljin.top/images/gaia-02-task-queue.svg" alt="任务队列" class="img-original">
  <figcaption class="figure-caption">图1：DBCache的任务队列</figcaption>
</figure>

以 DBCache 的 service 部分为基础，笔者进一步开发了 [harborRPC](https://github.com/booljin/harborRPC) 这个项目并开源。这个项目旨在线程间解耦，除了提供消息队列和常规的消息注册外，还提供一组 **send/call** 语法，使服务调用像函数调用一样简单。其中 *send* 只发送请求，不等待结果，而 *call* 会阻塞等待执行结果。

```c++
// 开发人员专注于逻辑，如MyService::func
class MyService : public harbor_rpc :: core :: rpc_server<MyService> {
    ......
    void register_cmds() override{
        ...
        register("func", &MyService::func);
        // 可以指定返回值，更像直接调用函数
        register_ex<std::vector<int>>("get_list", &MyService::get_list);
    }
}

// 使用者直接调用
harbor_rpc::client::send("my_service", "func", 10, 20);

std::vector<int> list;
int ret = harbor_rpc::client::call<std::vector<int>>(list, "my_service", "get_list", 5);
```
熟悉 **skynet** 的人应该会发现，这就是 c++ 版的 skynet service。只是由于没有 Lua 和 c++ 语法的差异，代码实现少了一份优雅。

## 0.2.3 JanusThreadManager
harborRPC 是笔者在 Actor模型 上的一次 c++ 实践。虽然它的设计更贴近通用服务框架，而非直接面向工业场景，但其中的 MQ、send/call 等核心思想，为 Gaia framework 的多个模块提供了直接灵感来源。**JanusThreadManager** 就是其中之一。

按照笔者的理解，工业领域不需要复杂的 service。甚至由于边缘计算节点 *Edge* 承载了大量耗时计算，主业务只需要汇总结果并进行决策，业务压力很小。于是笔者规划了两个常规线程：

* engine：所有需要业务关注的外部事件，都会投递到这个线程，于是业务层被限制在这个线程（API 除外，在 API模块 篇详述）。业务开发人员可以专心编写逻辑，不用担心多线程的问题
* assistant：Gaia 框架内部多个模块需要进行会话管理、定时器管理等。这些逻辑都会在 assistant 中执行，以此规避外部设备的 **时序不确定性**

```c++
using Task = std::function<void()>;
class ThreadManager {
    ...
    void ThreadManager::post(Task task){
        // 1. 加锁
        // 2. 队列长度监控
        // 3. push task
        _task_queue.push(std::move(task));
    }

    void ThreadManager::worker_thread_func() {
        while(_running){
            // 0. 记录时间点
            // 1. 处理定时器
            process_expired_timers();
            // 2. 处理任务
            while(true){
                // 3.
                if(/*本轮处理超时*/ || /*队列为空*/)
                    break;
                // 4. 
                get_task();
                do_task();
            }
            if(/*队列为空*/)
                sleep(10ms);
            else
                sleep(0ms);
        }
    }
};
```
两种 sleep 时间，是基于经验：
* 当队列中还有消息，意味着本轮循环已经 **超时**，必须暂时让出时间片防止整个系统卡死
* 如果队列为空，意味着当前系统负载可接受。10ms的延迟在大多数场景下可以接受，是cpu占用和及时响应之间的一个平衡点。大多数工业场景实时性不会特别高，否则很可能不是缩短 sleep 时间，而是需要采用实时操作系统等更有效的技术

# 0.2.4 Janus的神谕
此处展示了 JanusThreadManager 的关键逻辑。JanusThreadManager 是 Gaia framework 的线程基石，engine 承载业务，assistant 管理底层。业务开发人员只需要专注于业务逻辑，无需关心锁，无需关心线程。

这就是 JanusThreadManager 的设计哲学：一面向上托住业务，一面向下管住底层。将多线程的复杂性挡在门外，只留下一个干净的单线程世界。

就像那位两面回望的神祇，雅努斯。


<!--
User1   User2                                  
  ▲       ▲                                    
  │       │                                    
  ▼       ▼                                    
 ┌──────────┐                                  
 │Task1     │        ┌────────────────────────┐
 ├──────────┤        │                        │
 │Task2     ├───────►│  work thread           │
 ├──────────┤        │                        │
 │Task3     │        └────────────────────────┘
 ├──────────┤                                  
 │.....     │                                  
 └──────────┘                                  
-->