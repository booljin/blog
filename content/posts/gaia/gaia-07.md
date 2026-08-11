---
title: "0.7 AthenaAPI（其二）：ApiManager 和 MOC 细节"
date: 2026-07-30
categories: ["Gaia系列"]
tags: ["Gaia", "C++", "架构设计", "笔记"]
draft: false
image: "https://cdn.booljin.top/images/gaia-logo.svg"
---

## 0.7.1 无意中的业务耦合
笔者之前介绍过，每个模块都会有一个 Controller。其他模块都和具体的物理外设有关，天然就有 Driver、连接管理、会话管理等需求，Controller 设计目标非常清晰。但 API 模块的Controller 需要做些什么呢？

<!--more-->

笔者的第一直觉是直接用胶水代码将 handler 注入 Controller 中。第一版代码大致如下
```c++
// 在业务层执行的MOC代码：
api_controller->register("func1", func1);

// 框架中后端调用API的路由代码
cpp_resp = api_controller->call(method, cpp_req);

// 框架中Gateway路由代码
json_resp = cmds[method](json_req);
```
这套代码工作正常，编译、运行、测试都看不出任何问题。但笔者在一次复盘中，无意中发现框架代码目录里出现了 MOC 生成的文件。直觉告诉我：这不对。MOC 是基于业务层代码生成的，不应该蔓延到框架层。

进一步审视构建流程后，笔者发现了问题的根源：为了简化 Gateway 中 JSON 与 c++ 结构的互转，笔者将 MOC 生成的序列化代码直接放在了 Gateway 内部。这个判断看似合理—— JSON 结构只在外部传输时需要——但它导致了一个隐蔽的耦合：框架的二进制文件与业务层的 MOC 代码绑定在了一起。

如果是代码级复用，这不会有什么问题。但如果是二进制文件级别复用，这个耦合就会成为灾难。——一旦框架被多个业务工程以预编译库形式引用，MOC 代码的变更就会导致 ABI 不兼容。

这次复盘让笔者意识到：API 模块需要一个独立的，立足于业务的管理层，来统一管理 handler 的注册、调用路径的分发，以及与 Gateway 之间的数据转换。这个管理层，就是本节要介绍的 ApiManager。

## 0.7.2 BusinessCtx 和 ApiManager
进一步分析发现 Context 也有问题。Context 本身有两个职责：
* 框架层面，需要提供 notify收集器、resp收集器等
* 业务层面，需要提供 handler 执行所必须的业务组件

于是笔者将 Context 拆成了框架层的 ApiContext 和业务层的 BusinessContext 两层
<pre>
┌BusinessContext─┐
│┌──ApiContext──┐│
││connection_id ││
││session_id    ││
││response      ││
││error         ││
││notifys       ││
│└──────────────┘│
│ business_ptr   │
│ framework_ptr  │
│ api_manager_ptr│
│ componet1_ptr  │
│ ......         │
│ conponetN_ptr  │
└────────────────┘
</pre>

拆解 Context 很简单，真正的重头戏是 ApiManager。笔者对其定位是 **统管 handler 的三层胶水代码，并以虚接口的形式注册给框架进行简单调用**
```c++
class ApiManagerInterface {
    ...
    virtual JsonRpcHandler get_jsonrpc_handler(const std::string& method) const = 0;
};
```
笔者发现，由于 ApiManager 本质上由业务层管理，而 *后端调用* *嵌套调用* 本身也是业务层的需求，所以真正上升到框架的需求只有一个：在 Gateway 的 JsonRpc 信息流和实际 handler 的c++数据之间进行互转的接口。

```c++
include "moc_api_handler.inl"

class ApiManager : public gaia::ApiManagerInterface {
    USE_GAIA_API_MOC
    ...
    std::map<std::string, gaia::JsonRpcHandler> jsonrpc_handlers_;
};
```
这个写法借鉴了 Qt 这类依靠 MOC 的技术，同时兼顾了*可拓展*（这个文件随时可编辑）和*简单*：一个 `USE_GAIA_API_MOC` 宏引入所有的 MOC 代码。

而针对每个 handler，胶水代码一共有三套，以同时满足 API 的所有调用路径需求。

## 0.7.3 三套MOC代码
业务开发人员专注开发的是 handler 的核心内容
```c++
bool handle_func1(business::BusinessApiContext& ctx, const Func1Req& req, Func1Resp& resp);
```
在不同的调用路径下，需要不同的胶水函数，才能让业务层无感复用

* MOC方法A：核心调用：
    ```c++
    void call_func1(const Func1Req& req, connection_id, session_id){
        // 1. 构建ctx，所以需要connection_id 和 session_id。
        // 2. 构建闭包，理由是已经获得了 handler 运行的所有参数
        auto closure = [this, req](gaia::ApiContext& ctx) {
            // 4. 调用handler
            Func1Resp resp;
            bool ok = handler_func1(ctx, req, resp);
            if(ok){
                if(ctx.connection_id != gaia::INVALID_FRONTEND_CONNECTION) {
                    // connection_id 有意义，意味着是 Gateway 发来的前端请求，非后端调用
                    // 所以 resp 需要返回。否则可以丢弃
                    nlohmann::json resp_json;
                    to_json(resp_json, resp);
                    ctx.set_response(resp_json);
                }
            }
        };
        // 3. 通知 API线程池 执行任务
        _api_controller->post_task(closure);
    }
    ```
    这个调用路径支持了后端直接发起的 API 调用，但对于前端调用来说，还差一段

* MOC方法B：前端调用的外围包装
    ```c++
    void call_func1_with_jsonrpc(
        const std::string& params_json,
        gaia::connection_id_t connection_id,
        gaia::session_id_t session_id) {
        // 1. json → C++ req
        Func1Req req;
        try {
            nlohmann::json j = nlohmann::json::parse(params_json);
            from_json(j, req);
        } catch (const std::exception& e) {
            // 记录日志并直接返回错误
            auto ctx = std::make_unique<icw::backend::BusinessApiContext>();
            ctx->set_connection_id(connection_id);
            ctx->set_session_id(session_id);
            ctx->set_error(gaia::API_ERR_PARSE_FAILED, "Request parse failed");
            ctx->set_error_data({"detail", e.what()});
            _api_controller->send_error_response(*ctx);
            return;
        }

        // 2. 调用MOC方法A
        call_func1(req, connection_id, session_id);
    }
    ```
    至此，前后端调用的路径都打通，尚缺 API 嵌套调用。

* MOC方法C：嵌套调用
    ```c++
        void call_func1(BusinessApiContext& ctx, const Func1Req& req){
            Func1Resp resp;
            handler_func1(ctx, req, resp);
            // resp 直接丢弃（嵌套调用只需副作用，不需要响应）
            // notify自动添加到原 ctx 的 notify收集器 中
        }
    ```

在所有调用路径都确定之后，这三套 MOC方法 就顺理成章的写完了。
|MOC方法|调用来源|是否创建ctx|是否处理resp|后续处理流程|
|---|---|---|---|---|
|A|后端 or MOC B|是|是|投递至线程池|
|B|前端|否|否|MOC A|
|C|嵌套|否|否|无|

## 0.7.4 2026年AI赋能：让AI直接编写 MOC 工具
本质上，MOC 的过程就是文本替换的过程。笔者在过去的开发生涯中，既做过静态 MOC，也做过动态生成，虽然没什么难度，但耗时是真的。好在2026年的今天，AI编程已经非常普遍。笔者设计好格式后，让大模型直接生成了一个 Python 脚本，帮助生成静态 MOC 文件。相对于动态生成，它有一个巨大的好处：

> 依靠 c++ 的强类型特征，只要集成进编译过程，MOC 代码能通过编译器的编译期检查，就可以确认其安全可靠

当然笔者指的是 MOC 逻辑本身可靠。业务逻辑还是要靠业务开发人员自己保证。

## 0.7.5 Athena的神谕
没有物理设备，没有硬件依赖，纯粹从需求中推导出结构。开发人员写一套 handler，ApiManager 自动适配三条调用路径。这就是 AthenaAPI 的核心：用智慧取代蛮力，用结构消除冗余。

就像那位从宙斯头颅中诞生的智慧女神，雅典娜。