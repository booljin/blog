---
title: "0.5 HermesGateway：用 cmd 和 notify 彻底解耦前后端"
date: 2026-07-17
categories: ["Gaia系列"]
tags: ["Gaia", "C++", "架构设计", "笔记"]
draft: false
image: "https://cdn.booljin.top/images/gaia-logo.svg"
---

## 0.5.1 引子：事件驱动/EDA
**事件驱动/EDA** 是理解整个 Gaia framework 的钥匙，也是笔者多年来服务器开发的核心思想。从 Janus 的 Actor模型，到 Argus 的 send/call，再到本篇的 cmd/notify，本质上都是事件驱动在不同层面的体现。限于篇幅，本文不展开讨论事件驱动本身，而是聚焦于它在前后端通讯中的具体应用——cmd 和 notify。关于事件驱动模型的专题，笔者计划在未来单独成文。

<!--more-->

## 0.5.2 cmd 和 notify：模块解耦的前提和结果
> 一个能力极强的游戏开发人员，策划、代码、美术、音乐一把抓，当然可能有很好的质量和效率，但这也产生了耦合。当他工作状态出了问题，所有任务都陷入停滞。
> 
> 一个标准的工作室，会设立独立的策划、程序、美术等岗位，这样即使程序这两天闹肚子，其他工作也能先推进，等程序身体恢复了，再回来接着开发。虽然人员成本和沟通成本上升了，但是整个项目的风险显著下降。

以一个程序的视角，还原几个典型的沟通切片：
* 策划写完策划案，拆分出程序需求并交付开发（cmd 的 request）
* 开发完成（cmd 的 response）
* 程序进行了一些拓展，告诉策划未来可以支持更多可能性（notify）

笔者当前所在的公司，有过几次不是很成功的解耦。大体来说是两种问题：
> * 物理形式上解耦，而不是在概念和逻辑上解耦。规定某些文件某些代码属于后端，但没有任何隔离，所以没过多久，原本后端的代码里重新爬进了前端逻辑
> 
> * 隔离手段复杂，定义、发送 cmd 的步骤繁琐，开发人员很快就想尽方法绕过去

笔者作为10多年的游戏服务器主程，积累了一些自己的方法论。[第四篇](./gaia_04.md)中的 send/call，就是笔者针对隔离手段复杂的解决方法之一。一个 call 函数调用，就像本地函数调用一样自然，开发人员不需要理解背后的复杂逻辑，就能专注于实现自己的业务。但这肯定不够，更多设计会在后续的 API 模块中持续展开。

send/call 语义也好，API 模块也好，都是跨越那条通讯鸿沟的桥梁，而那条鸿沟本身，保证了解耦的彻底性：用 **cmd** 和 **notify** 彻底分割两个模块。

## 0.5.3 多前端 {#gaia-05-03}
笔者所在的团队一直有 *UI设计能力不足* 和 *客户价值感低* 的困扰，归根到底是传统的 GUI 程序设计理念中，只有一套前端，当团队希望将更多的调试功能塞进去时，不得不面对几个重大问题
* 调试功能无法带来客户价值，甚至会严重削弱客户价值
* 在统一 GUI 设计中，任何功能，哪怕是调试功能，也必须符合统一风格，否则会有严重割裂
* 每项功能都有成本，也有负担，越来越多的功能会导致越来越大的维护成本

在游戏团队中，笔者开发过一套基于 *Golang+微服务+K8s* 的分布式系统，用于支撑SDK接入、区服间自维护、GM系统等非游戏核心服务。笔者深知，不同用户关注的内容完全不同，从最初 *通过身份系统控制呈现内容*，到后来 *为特定身份的人定制特定前端*，笔者团队尝试了很多解决方案。一个典型的场景是：老板不关心服务器状态，也不关心GM系统，只想看到当前的总收入，笔者只需要给他定制一个极简的，打开即看的界面即可。

笔者针对当前团队的痛点，提出的也是类似解决方案

<figure class="img-figure img-original">
  <img src="https://cdn.booljin.top/images/gaia-05-multi-frontend.svg" alt="多前端" class="img-original">
  <figcaption class="figure-caption">图1：多前端</figcaption>
</figure>

|类型|特点|
|---|---|
|standard|标准前端，专注于客户价值，有完整统一的设计语言|
|debug|调试前端，可能有很多粗糙功能，内部使用，可以专注于开发效率而不是界面设计|
|cli|命令行前端，可以用于开发MCP，供大模型调用|
|其他|任何为了特殊场景而开发的前端|

多个前端的存在，意味着 Gateway 必须承担起 *路由* 和 *分发* 的职责。前端发来的 cmd，需要被正确地路由到后端；后端产生的 notify，需要被按需分发给订阅了对应主题的前端。这就是 HermesGateway 的核心功能——用 cmd 和 notify 这对语义，彻底解耦前后端。

上行通道cmd 经过 HermesGateway 路由后，会进入 API 模块进行处理，这是后续多篇文章的主题，本篇完全略过，聚焦于下行通道notify。

## 0.5.4 notify的订阅机制
当系统产生了一些需要通知相关方的信息，就会产生一个notify。比如前面游戏团队的例子，程序员生病请假，就需要主动通知策划和主程，以便相关方调整计划安排，至于美术可能就没必要通知。在一个工业系统中，实时生产活动可能会导致 *统计数据变更* *最新实时监测数据变化* *产生新的日志信息* 等众多连锁反应，需要推送 notify 给各个前端。

除了生产活动本身可能产生 notify 外，还有一个常被忽略的场景：某个前端发来的 cmd 本身，可能也会产生 notify。例如一个前端发送了 *系统运行* 的指令，这会导致整个系统的运行状态发生了重大变化。这类运行状态变化，笔者将其定义为 **快照Snapshot** 变化，这是需要全量同步的，否则部分前端和后端之间状态会脱钩。

有一些 notify，例如实时监测数据，可能涉及图片等大块数据，全量推送会显著消耗通讯带宽，于是笔者设计了一套 **订阅机制** ，来规避无意义的带宽消耗。

每个前端在连接之后，可以通过一条 **handshake** 请求进行频道订阅
```c++
void FrontendController::on_driver_request(connection_id_t connection_id,
                                       session_id_t session_id,
                                       const std::string& method,
                                       const std::string& params_json) {
    // 内置方法：session.subscribe
    if (method == "session.subscribe") {
        handle_subscribe(connection_id, session_id, params_json);
        return;
    }

    // 其他方法：调用 API Handler
    ...
}

void FrontendController::handle_subscribe(connection_id_t connection_id,
                                      session_id_t session_id,
                                      const std::string& params_json) {
    // 解析频道列表
    std::set<std::string> channels;
    nlohmann::json params = nlohmann::json::parse(params_json);
    if (params.contains("channels") && params["channels"].is_array()) {
        for (const auto& ch : params["channels"]) {
            if (ch.is_string()) {
                channels.insert(ch.get<std::string>());
            }
        }
    }

    // 直接存储，不验证、不添加、不过滤
    connection_mgr_.set_subscription(connection_id, channels);
    ...
}
```
笔者认为工业领域通常都是局域网内访问，网络环境安全，只需要稳定简单的功能，所以没有开发任何鉴权功能。当然在服务集群中，鉴权功能也是可以解耦的，这是另一个话题，笔者未来可能会在其他笔记中介绍自己的经验。

对于频道订阅，笔者也没有开发复杂的逻辑。笔者认为：
* 前后端开发人员自行约定频道，频道名为字符串
* Gaia framework 不校验任何频道名，不对拼写错误或任何逻辑错误负责。如果前后端频道名没对上，前端将收不到这条 notify
* 如果前端没有订阅任何频道，默认收不到任何 notify
* 后端具有强制发送 notify 的权限。通过保留频道名 *`MANDATORY`* 让 notify 无视订阅列表，直接发送给任何前端。笔者认为后端是系统基础，理应具有更高权限

```c++
void FrontendController::notify(...) {
    // post到assistant线程，具体内容参考前几篇
    _assistant->post([this, 
                ...
                event,  /*notify 事件*/
                data,   /*notify 信息*/
                channel,/*notify 频道*/
                is_broadcast/*是否广播*/](){
        std::vector<ConnectionID_t> targets;    // 待发送前端列表
        if(is_broadcast){   // 广播
            if (channel == "MANDATORY") {       // 强制频道
                targets = online_connections();  // 全量发送
            } else {
                targets = subscribed_connections(channel);// 按频道发送
            }
        } else {            // 单播：notify_back
            if (resp.connection_id != INVALID_FRONTEND_CONNECTION) {
                targets = {resp.connection_id};
            }
        }
        ...
    });
}
```

大多数 notify 所属的频道，都是通过 YAML 定义，并通过 MOC 集成进系统的。笔者反复强调，目标是让开发人员专注于逻辑，而 yaml——handler 体系是 API 模块的核心功能之一：**通过 MOC 实现声明式定义**，留待后续介绍，这里只给出一小段 YAML 切片作为示例：
```yaml
methods:
  snapshot:
    channel: MANDATORY  # 强制推送，所有前端自动接收
    notify:
      - name: run_state
        type: int
      - name: scheme
        type: string
      ...
```

## 0.5.5 notify_back: API响应流程解耦
上文伪代码展示了 notify 发送时的分层策略。这里有一个上文没提到的概念：*notify_back*。在一个 C/S 系统中，notify 并不总是广播，有时候需要定向发送。在通用网络服务中，定向发送可以解决很多问题。比如在游戏服务器中，同一副本的同步消息只需要发送给副本中的玩家，而不需要同步给全服。在工业系统中，笔者暂未发现这种需求，但笔者还是设计了 *原路返回的notify*，即 notify_back，这是源于另一个原因。

假设有一组信息，定义了系统快照Snapshot。其中有两个信息：
> 1. 运行状态：Start/Stop 这个状态的变化需要全量推送给所有前端
> 2. 当前生产方案：优先级权重和运行状态不同，不需要全量推送给多个前端

假设有多条 cmd，都会实际影响 Snapshot 中的生产方案，一个合理的假设是多条 cmd 的 response 都包含 Snapshot，这样一来，这些 response 都必须嵌入完整的 Snapshot，形成了不必要的耦合。

笔者的做法是：cmd 的结果只返回必要信息。如果 cmd 执行过程中产生了 *连锁反应* ，则将连锁反应作为独立流程，产生独立notify，另行发送。而这个 *另行发送* 其实就是 *notify_back* 。在整个调用链中，所有 notify 都会依序存入 *notify收集器* （下一篇会介绍）。在发送完 response 后，依序处理每一个 notify。，

除了后端，前端也同时具备了信息解耦能力：只需要具备 Snapshot Notify 的解析能力即可，不需要在每一条可能引发快照变化的消息中都专门处理 Snapshot 数据。

<figure class="img-figure img-original">
  <img src="https://cdn.booljin.top/images/gaia-05-resp.svg" alt="resp" class="img-original">
  <figcaption class="figure-caption">图2：返回格式对比</figcaption>
</figure>

前端依次收到多条独立消息，可以依序调用解析流程


> *一个cmd resp + 若干个 notify* ，本质上和 gRPC 的 *服务器流式调用* 是一样的

上述伪代码中还引出了另一个问题：当 notify_back 的目标是 *`INVALID_FRONTEND_CONNECTION`* 时，整个notify都会被忽略。这是为后端内部调用 API 提供的能力，这是未来API模块中重点介绍的功能之一，本篇暂时略过。

## 0.5.6 Hermes的神谕
设立前后端的鸿沟，用 cmd 和 notify 来沟通。设立操作结果和状态变更的边界，用 notify_back 来沟通。

这就是 HermesGateway 的核心：它不滞留，不窥探，不裁决。它只是忠实的将每一条信息从一端送到另一端，步履不停。

就像那位穿行于边界，永不停歇的信使，赫尔墨斯。


<!--
                                              ┌──────────────────────────────────┐
                                              │            business              │
                                              └────────────────┬─────────────────┘
┌──────────────────────────────────┐                           │ notify           
│               API                │                           ▼                  
└──────────────────────────────────┘          ┌──────────────────────────────────┐
                 ▲                            │  gateway                         │
                 │ cmd                        │                                  │
┌────────────────┴─────────────────┐          │      _subscriptions              │
│             gateway              │          │                                  │
└──────────────────────────────────┘          └────────────────┬─────────────────┘
                 ▲                                             │                  
    ┌────────────┼────────────┐                   ┌────────────┤                  
    │            │            │                   ▼            ▼                  
┌───┴────┐   ┌───┴────┐   ┌───┴────┐          ┌────────┐   ┌────────┐   ┌────────┐
│standard│   │ debug  │   │   cli  │          │standard│   │ debug  │   │   cli  │
└────────┘   └────────┘   └────────┘          └────────┘   └────────┘   └────────┘
上行通道cmd                                    下行通道notify




┌─────────┐     │     ┌──────────┐
│resp data│     │     │ resp data│
│         │     │     └──────────┘
│snapshot │     │     ┌──────────┐
│other    │     │     │ snapshot │
│         │     │     └──────────┘
│         │     │     ┌──────────┐
│         │     │     │  other   │
└─────────┘     │     └──────────┘
                │                 
    NO          │         YES     


-->