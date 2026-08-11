---
title: "0.4 ArgusEdge：用 send/call 隐藏会话管理细节"
date: 2026-07-16
categories: ["Gaia系列"]
tags: ["Gaia", "C++", "架构设计", "笔记"]
draft: false
image: "https://cdn.booljin.top/images/gaia-logo.svg"
---

## 0.4.1 会话管理：通讯中绕不开的难点
很多朋友都接触过各种语音助手，比如小爱音箱。*你问一句，它答一句，双方在欢乐友好的气氛中进行友好对话*，这是很多人最初对智能助手的想象。但当前现实远没有这么美好。

<!--more-->

* 场景1：你说 “小爱同学”，等了一会儿它没反应，你继续喊它，还没反应，你再喊，它回答 “我在”。然后你开始说问题，才说了几个字，它又说 “我在”，你还在愣神，“我在” 又响起来了
* 场景2：你家3个房间都有小爱音箱，你对着客厅里的喊了一声，结果另两个房间传来 “我在”，而客厅里那位纹丝不动。

小爱的问题本质上是 **多路并发请求的会话标识丢失** —— 设备无法区分 *这是我该回的* 还是 *那是别人该回的*。工业场景中，当我们向多个边缘节点同时发送指令时，同样面临这个问题：谁的回复对应谁的请求？超时了怎么办？重复回复了怎么处理？在任何通讯场景，会话管理都是绕不开的难点。这也是OSI七层模型中，会话层占有一席之地的根本原因。

更进一步，如果需要给多个节点同时发送消息，除了一个一个串行发送外，能不能并行发送？串行发送的好处是简单，但如果单个节点需要等待2s后回复，则随着节点数增加，延迟会线性增加。如果能一股脑发送，然后一起等结果，可以在多节点通讯中显著节约时间，但复杂度也随着增加了。

<figure class="img-figure img-original">
  <img src="https://cdn.booljin.top/images/gaia-04-serial-vs-parallel.svg" alt="串行发送 vs 并行发送" class="img-original">
  <figcaption class="figure-caption">图1：串行发送 vs 并行发送</figcaption>
</figure>


笔者有十多年游戏服务器开发经验，深知这类问题一直是高并发服务器开发的经典问题。ArgusEdge 就是笔者针对工业领域多节点通讯给出的自己的答卷——用一组 send/call 接口，将这一切复杂性隐藏在框架之内。

## 0.4.2 send/call 速览
先给出一组示例，展示 send/call 使用
```c++
// ==============================================================
// send: 发送完立即返回，不等待回复，没有会话管理。即使收到回复也会丢弃
// ==============================================================
gaia->edge()->send(group/* vector<ConditionID_t>，指定向哪些edge节点发送信息 */,
    "cmd_name"/* cmd name */,
    buff/* 业务层已经处理好的发送数据 */);

// =====================================
// 介绍call之前需要先介绍一下call的返回结构
// =====================================
struct BatchCallResult{
    ...
    std::map<ConnectionID_t, CallResult> results;
};
struct CallResult{
    ...
    ConnectionID_t edge_id;
    int status; // 成功、失败、超时、离线
    std::string resp;
    std::string error;
};

// ==========================================================
// call: 用法1，类似于send，给若干个节点发送相同信息，但会等待结果
// ==============================================================
BatchCallResult ret;
ret = gaia->edge()->call(group/* vector<ConditionID_t>，指定想哪些edge节点发送信息 */,
    "cmd_name"/* cmd name */,
    buff/* 业务层已经处理好的发送数据 */,
    5000/* 超时时间，如果不设置则使用默认时间。在规定时间内没有收到足够resp，call也会返回 */);

// ==============================================================
// call: 用法2，给不同节点发送不同信息，并一起等待
// ==============================================================
// 此处借鉴了cuda等异步等待模型的接口设计，将多次call关联到一个同步点
// 统一等待执行结果
auto sync = gaia->edge()->create_sync_point();
// 构造任务
gaia->edge()->call(edge1, "cmd_name", buff1, sync);
gaia->edge()->call(edge2, "cmd_name", buff2, sync);
ret = gaia->edge()->wait(sync, 5000/* 超时时间 */);
```

先回应一下 [上一篇的悬念](./gaia_03.md#send-call-domain)<!-- cross-ref: gaia_03 --> ，ArgusEdge 和 HeliosEquipment 的 send/call 同名但不同域。后者面向机台，前者面向边缘节点，两者 session 管理机制共通，但接口参数不同。

## 0.4.3 基于 session id 的 call
<pre>
call的时序图

Business Thread                      Assistant Thread
─────────────────────────────────────────────────────
call(cmd, sync)          
  │                                  
  ├──► Create CallTask              
  │     Post to Assistant           
  │                                 ├──► Create Session
  │                                 ├──► Driver->Send()
  │                                 ├──► Register SessionMap
  │                                  
  │         ──── async ────         
  │                                 
wait(sync, timeout) ◄────┐          ├──► On Response
  │                      │          │     └─► Lookup SessionMap
  │                      │          │     └─► Update Session State
  │                      │          │
  │                      │          ├──► Check Timeout (timer)
  │                      │          │     └─► Lookup SessionMap
  │                      │          │     └─► Mark Timeout
  │                      │          │
  │                      │          ├──► All Done?
  │                      └──────────┘     └─► Yes: Wake Business
  │                                  
return BatchCallResult
</pre>

session_id 是 req-resp 配对的唯一依据。req 携带 session_id，resp原样返回。

session_id 生成器避开了 0，这是留给 send 使用的。edge 返回 session=0 的resp，自然会被忽略，不会向业务层传导。
```c++
uint32_t IPUSessionMgr::next_session_id() {
    uint32_t id = next_session_id_++;
    if (next_session_id_ == 0) {
        next_session_id_ = 1;
    }
    return id;
}
```

#### assistant 线程在 call 中的价值
正如 [第二篇](./gaia_02.md)<!-- cross-ref: gaia_02 --> 所言，assistant 是笔者针对 **时序不确定** 问题的解决方案。在 call 这个场景中有非常典型的应用。

session 管理模块中的多个数据都有竞争风险，比如 session_map，在网络消息、定时器逻辑中都要访问并修改。传统逻辑必须加锁。并且由于 *On Response* 事务和 *Check Timeout* 事务本身都比较复杂，为了保证事务原子性，锁的范围较大。

但在 Gaia 中，所有外部触发时间，都在 assistant 线程中按照入队顺序有序执行。每个业务都不需要担心其他事务和自己争抢数据，任何数据都不需要加锁。

另一个比较容易搞乱思绪的，是时序边界的处理。时序边界问题本质上是 *谁先入队* 的问题：如果 *response* 先入队，则正常返回；如果 *超时* 先入队，则返回超时，后续 response 因 session 已移除而被丢弃。assistant 的串行执行保证了这种判定的确定性。

#### call 方法的小遗憾以及 **退化机制**
笔者的 send/call 语义借鉴了云风 skynet 中的相关语义。但是 skynet 使用 Lua 实现相关语义，使用 Lua 的 Coroutine协程，相关实现优雅且不会真的阻塞，而是被 *挂起*。待到有了回应，会重新唤醒这个暂时挂起的流程并继续下去。但是对于调用方来说，**挂起** 和 **重新唤醒** 是透明的，就像是一次简单的函数调用。

多年以前，笔者曾经做过一套 *挂起*——*唤醒* 系统，将后续流程作为 callback 记录到 session_map 中。它可以务实的完成相关功能，但有个小缺点，开发流程是割裂的。整个通讯分为 *发送请求* 和 *处理相应* 两部分，且需要各自编写逻辑。笔者认为这对 **专注于开发逻辑** 的开发人员并不友好。

C++20 引入了协程，但在笔者评估时，其生态和编译器支持尚不成熟。最终选择了 std::promise / std::future 作为阻塞-唤醒机制——虽然不如协程优雅，但胜在稳定可靠。

```c++
BatchCallResult EdgeController::execute_batch_call(...){
    ...
    auto future = _session_mgr.submit_batch_call(
        patch, timeout_ms
        );
    auto status = future.wait_for(
        std::chrono::milliseconds(
            timeout_ms + 1000)
        );
    if (status == std::future_status::timeout) {
        // 超时处理
    } else {
        // 正常处理
    }
}

std::future<SessionBatchResult> IPUSessionMgr::submit_batch_call(...){
    ...
    return promise.get_future();
}

// if all done
promise.set_value(real_data);
```

这里有一个问题：业务层代码有两种运行场景：
* api：运行于专用线程池，允许阻塞，且需要阻塞等待结果，才能执行后续逻辑，阻塞是被鼓励的
* engine：这是不应该阻塞的！否则可能造成严重的实时性故障，整个系统无法有效运转

笔者认为，任何时候，都不应该只依赖于开发人员 *小心的正确使用* 接口，而是 **将问题解决在未发生之前**。针对这个问题，笔者设计了 call 的 **退化机制** 。如果检测到当前 call 是在 engine 中调用的，则将其改成 send 调用，仅输出警告日志以备排查，不阻塞线程。

```c++
BatchCallResult IPUGroupController::execute_batch_call(...){
    if(_engine && _engine->is_current_thread){
        // Engine 线程检测 → 退化处理
        // 每个结果都设为"成功"
        return result;
    }
}
```
退化之后的 call，所有结果都是 *成功*，因为此时确实不知道最终结果会如何，只能保证表面上不影响业务进展。打印的日志留下了线索，开发人员应该靠日志的提醒来完善业务逻辑，比如将 call 改成 API 调用，而这在后面有专题来介绍。

## 0.4.4 Argus的神谕
至此，send/call 的完整链路已经清晰：send 无状态、call 有状态、assistant 保序、退化机制兜底。业务层只需调用一行接口，无需关心 session 的创建、匹配、超时。

这就是 ArgusEdge 用 send/call 隐藏会话管理细节的全部含义。它永不松懈的盯着每一个边缘节点，确保所有通讯都能有序、正确，确保每一步都不会踏错。

就像那位永远醒着的看守者，百眼巨人阿尔戈斯。


<!--
┌─────────┐          ┌─────────┐            ┌─────────┐                                   
│         │ ───────► │         │            │         │                                   
│         │   2 s    │ Edge 1  │            │         │                                   
│         │ ◄─────── │         │            │         │ ───────────┬────────┬────────┐    
│         │          └─────────┘            │         │            │        │        │    
│         │          ┌─────────┐            │         │            ▼        ▼        ▼    
│  Argus  │ ───────► │         │   how?     │  Argus  │         ┌──────┐ ┌──────┐ ┌──────┐
│  Edge   │   4 s    │ Edge 2  │ ────────►  │  Edge   │         │Edge 1│ │Edge 2│ │Edge 3│
│         │ ◄─────── │         │            │         │         └──┬───┘ └──┬───┘ └──┬───┘
│         │          └─────────┘            │         │    2 s     │        │        │    
│         │          ┌─────────┐            │         │  ◄─────────┴────────┴────────┘    
│         │ ───────► │         │            │         │                                   
│         │   6 s    │ Edge 3  │            │         │                                   
│         │ ◄─────── │         │            │         │                                   
└─────────┘          └─────────┘            └─────────┘                                   
左：串行发送，延迟随节点数线性增长；右：并行发送，总延迟取决于最慢节点。
-->