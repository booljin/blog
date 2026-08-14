---
title: "0.6 AthenaAPI（其一）：API模块的演进"
date: 2026-07-29
categories: ["Gaia系列"]
tags: ["笔记", "C++", "架构设计"]
draft: false
image: "https://cdn.booljin.top/images/gaia-logo.svg"
---

## 0.6.1 前端调用 API
在 [第五篇](./gaia-05.md#gaia-05-03)<!-- cross-ref: gaia-05 --> 中，笔者提过将前后端彻底解耦，并且规划了多前端系统。前后端需要交互，自然需要API——API模块因此成为 Gaia framework中最基础的模块之一。

**只是第一版 API 非常简单**。

<!--more-->

前后端开发人员协商接口格式：cmd名字，req字段，resp字段。通过YAML定义，MOC 生成结构体和胶水代码，后端开发人员在 handler 里专注于业务逻辑——这是笔者一直的承诺。

```yaml
  func1:
    req:
      - name: param1
        type: string
    resp:
      - name: data1
        type: array
        items: int
```
```c++
bool handle_func1(business::BusinessApiContext& ctx, const Func1Req& req, Func1Resp& resp);
```
handler 在 *API专属工作线程池* 运行，允许阻塞，可以放心使用 Edge 等模块的阻塞接口访问外设。看起来唯一令人不满的是 handler 允许但不应该直接修改 business 数据，因为 **business 数据必须在 engine 线程修改** 是 Gaia framework 的默认规则。在其他技术栈如 *Lua* 中，笔者可以轻易限制这些行为，但在 c++ 中需要费一点事。这会在后续内容中进一步提及。

第一版顺利达成了预期目标。前端发请求，Gateway路由，API handler执行，返回结果。每个 handler 只负责一件事，代码清晰，开发效率很高。

**直到有一天，笔者需要在 engine 中与 Equipment 沟通，并获得结果**

## 0.6.2 后端调用 API
笔者的第一反应，是给 Equipment 增加一组新接口，在发送完后注册一个回调，在回调里处理返回值。但是这样的坏处显而易见：笔者必须接受 *精心设计的 send/call 接口实用性不足，必须再提供一组接口给其他场景使用*，这不是一个优雅的解决方案。

更根本的问题是，在已有的API handler中，已经有 *handler_func1* 可以提供业务所需功能了。有没有办法复用这个功能？

在反思的过程中，笔者逐渐意识到一个问题：由于以 *前后端分离* 作为设计目标，笔者潜意识里把 *前端调用* 当成 API 的唯一合法场景。

在 Qt 或 MFC 这类框架中，API 并不是 *对外暴露接口* 的代名词。它就是一个功能入口——引擎暴露给业务开发人员的调用契约，而不关心调用者的身份是后端还是前端。那么，为什么我的 API handler 不能被 engine 线程调用呢？req/resp 定义就是它的调用契约，handler 逻辑就是它的任务，甚至它已经有 ctx，可以识别自己的调用路径！

于是思路非常清晰：笔者需要让 handler 可以被业务层在 engine 中调用，并且也投递到 API专属工作线程池 运行，就像前端调用一样。实现方式也非常明确：MOC 生成第二套胶水代码，而 handler 本身不需要任何修改。

```c++
    // engine中执行的逻辑代码...
    api_manager->call_func1(func1_req); // API 模块会在 API 工作线程池中执行逻辑
```

只有一个小问题：我的逻辑推演继续下去，API 拆成 **最小可复用逻辑单元**，那么除了 *业务层调用API* 外，API handler 本身可以调用 API handler 么？我的这套逻辑可以闭环么？我需要继续推演下去。

## 0.6.3 API handler 嵌套调用
API 的默认执行路径是 *生成 API Task --> 投递到 API专属工作线程池 --> 等待空闲线程 --> 执行handler*。通常工作线程不会太多，2-4个足够。

handler 被设计成支持阻塞，Gaia framework 的设计目标也是鼓励开发人员用最符合直觉的方式开发逻辑，尽量避免用回调和异步来打断开发思路。于是如果 API handler 需要嵌套调用，笔者无法 *让子调用作为新 API Task，占用新的工作线程，并阻塞父 handler*——这会导致 API专属工作线程池 被迅速耗尽，从而导致其饿死。而且，线程调度开销也是无法回避的问题。

等等，什么是灯下黑？就在笔者信马由缰的思考一轮，并得出沮丧的结论后，忽然发现，handler 本身就是函数！为什么不能直接调用这个函数？

> 每个 API handler 都需要 req，这没有问题，在任何调用场景下，调用方必然知道如何构建 req，否则整个调用根本没有意义。
>
> 每个 API handler 都会生成一个 resp。这和 req 有所不同。resp 几乎是为前端调用定制的，而 *后端调用API* 和 *嵌套调用API* 都应该忽略

发现这个洞察后，事情就变得简单了，再次通过第三套 MOC 代码，直接调用 handler，并忽略 resp，就可以在 API handler 中嵌套调用其他 API handler 了。

```c++
bool handle_func1(business::BusinessApiContext& ctx, const Func1Req& req, Func1Resp& resp){
    ...
    ctx->api_manager()->call_func2(ctx, func2_req);
}
```
这个和上例中唯一的区别是，此接口需要传入父调用的 ctx，这是让整个逻辑闭环的关键：它串起了 **三种 API 调用路径** 和 [上一篇](./gaia-05.md)<!-- cross-ref: gaia-05 -->中提到的 *notify收集器* 机制

## 0.6.4 ctx：闭环的关键
至此，API handler 的三种调用类型已经全部清楚：
|调用路径|特点|
|---|---|
|前端调用|有前端 *session_id* 和 *connection_id*, ctx 在 gateway 路由时创建|
|后端调用|没有有效的 *session_id* 和 *connection_id*，不需要回复前端。ctx 在 engine 胶水代码中创建|
|嵌套调用|沿用原ctx，维持其特征|

```c++
class ApiContext {
    ...
    connection_id_t connection_id_ = 0;
    session_id_t session_id_ = 0;
    // Notify 收集器
    std::vector<CollectedNotify> notifies_;
};
```

除了各个 handler 所需的资源外，ctx中包含 session_id，connection_id，用于识别调用路径是前端还是后端调用。值得说明的是 **notify收集器**。这源于笔者的进一步逻辑推演：
> resp 和 req 强相关，如果前端没有发送 req， 则无法正确处理 resp，此时 resp 需要丢弃。但是如果一个 API handler 中明确表示需要发送 notify，在 resp 丢弃的同时，notify 怎么处理？

笔者认为，每一个 API handler 的调用路径虽然可能不同，但是开发人员编写的逻辑需要认真对待。如果一个开发人员在 handler 中明确产生 notify，Gaia framework 不会直接丢弃，而是统一收集到 notify收集器 中，并在后续统一处理：
* 前端调用：notify 按照预期，在发送完 resp 后按照生成顺序统一发送
* 后端调用：由于缺少有效前端目标，notify 队列中的 notify_back 类会被丢弃。但广播类 notify 还是会正确发送
* 嵌套调用：由于沿用父调用 ctx，notify 都会收集到父调用的收集器中，并随父调用策略决定后续逻辑

## 0.6.5 未完待续
下一篇，笔者将会剖析 MOC 的细节，展示胶水层的运作机制。

to be continued...