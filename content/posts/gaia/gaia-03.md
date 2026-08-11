---
title: "0.3 HeliosEquipment：用透传解耦机台"
date: 2026-07-15
categories: ["Gaia系列"]
tags: ["Gaia", "C++", "架构设计", "笔记"]
draft: false
image: "https://cdn.booljin.top/images/gaia-logo.svg"
---

## 0.3.1 Equipment机台的多样性
由于设计目标不同，实际能力不同，机台千差万别，对应的 *协议* 也是千差万别。

这里说的"协议"是广义的。比如笔者所在的团队，基于 Modbus 协议 和机台通讯，日常工作就是读写指定寄存器，没有 *协议* 的概念。

<!--more-->

但笔者了解了这批寄存器的功能后，发现其实可以将他们分成3类，重新纳入笔者熟悉的 C/S 通讯架构模型：

> |寄存器特点|洞见|类比|
> |---|---|---|
> |只读，如机台配置信息|可以视作系统向机台发送一个查询request|下行request|
> |只写|可以视作系统向机台发送一个写入request|下行request|
> |机台维护状态，系统定期查询|可以视作机台向系统发送一个事件通知|上行notify|

*凡有信息传递，都可以视作协议*，而所有协议，都有一些共同的特点：

* request：总对应一条 response。这意味着它需要 session 管理、超时管理等基础功能
* notify：单方面的事件通知，不需要回复
* 任何一条 request 或者 notify，都一定有一个 name 标识身份

<figure class="img-figure img-original">
  <img src="https://cdn.booljin.top/images/gaia-03-equipment-communication.svg" alt="通讯模型" class="img-original">
  <figcaption class="figure-caption">图1：通讯模型</figcaption>
</figure>


以基于 Modbus 协议的机台通讯为例：

* driver 50ms轮询。假设机台数据刷新周期400ms，Modbus 单次通讯耗时约5ms，50ms的轮询间隔在通讯消耗与实时性之间取得了平衡，既不会过度占用总线，也能在数据刷新后及时感知并向上层发送 notify
* driver 在收到下行 req 后，将其转化成指定的 Modbus 读写操作，完成后返回结果给上层，视为 response

```c++
auto gaia = std::make_unique<gaia::GaiaFramework>();
// 业务自行开发或选用driver，和业务层配对使用。框架只负责透传
auto equipment_driver = std::make_unique<gaia::ModbusEquipmentDriver>(
        "192.168.1.2", 502);
gaia->set_equipment_driver(equipment_driver);
```

## 0.3.2 协议透传：无所为，才能无所不为
Gaia framework致力于打造边缘计算场景下的通用解决方案，自然要极力规避任何程度的 *逻辑耦合*。既然机台通讯协议没有定型，就应该将其剥离。

无论是 request，还是 notify，所谓协议，都有一个 *协议name*，附带一组 *数据信息*。为了避免 c++ 这种强类型语言的限制，笔者将 Gaia framework 中流动的信息指定为 JSON。做出这个决定的前提是，笔者认为是这些信息体量都很小，顶多几十个字段，信息总量小于 1kb，JSON 作为一个泛类型容器，成熟简单。

> 在 Edge 和 Gateway 中，各有一个特例协议，由于数据量大，包含图片，笔者使用 Protobuf 进行通讯，并开了特例接口。这是一个无奈的平衡：一方面团队使用 Protobuf 的意愿不强，另一方面大数据确实不适合用 JSON 传递。这里只稍作说明，后续不再提，因为本系列无疑覆盖 Gaia 的所有细节

### 0.3.2.1 用 MOC 隐藏 JSON，让开发人员专注于逻辑
笔者希望开发人员能专注于用 *熟悉的语言，如c++* 开发逻辑，JSON 会增加开发人员的心智负担。于是笔者写了一个 MOC 工具，基于 YAML 的类型定义，生成 c++ 结构，以及和 JSON 互转的胶水代码
```yaml
Notify_1:
  - name: info1
    type: int
  - name: info2
    type: string
```
最终生成moc_equipment_typedef.h中包含如下片段
```c++
struct Notify_1_t{
    int info1;
    std::string info2;
};

Notify_1_t from_json(json& params){
    ......
}
```

### 0.3.2.2 上行notify
Gaia framework 提供一个 register_notify 接口，开发人员，可以将自己编写的 notify 处理逻辑注册于此，当 EquipmentController 收到指定 notify，就会调用这个 handler。

```c++
namespace helios{
    using NotifyHandler = std::function<void(const nlohmann::json& params)>;
    class EquipmentController{
        ...
        // 如前篇《0.2 从DBProxy到JanusThreadManager: 用Actor模型隐藏多线程开发的复杂度》所言，gaia framework的支持性逻辑均在assistant线程运行
        void register_notify(const std::string& method,
                NotifyHandler handler){
            _assistant->post([this, method, handler](){
                // handler的注册、查询
            });
        }
        void on_driver_notify(const std::string& method,
            const std::string& json_params){
            _assistant->post([this, method, json_params](){
                // 查询handler
                // 如果未找到，输出错误日志并返回
                // 顺利的话继续执行
            });
        }
    };
}
```

有关assistant的内容，可以参考 [前一篇](./gaia_02.md)<!-- cross-ref: gaia_02 -->

开发人员只需要专注于 c++ 版本 handler 的编写
```c++
// 业务层
#include "moc_equipment_typedef.h" // MOC生成
namespace business{
    class EquipmentManager{
        void init(helios::EquipmentController* controller){
            ...
            controller->register_notify("Notify1", 
                [this](const json& json_params){
                    Notify_1_t params = from_json(json_params);
                    on_notify1(params);
                }
            );
        }

        void on_notify1(Notify_1_t params){
            // 开发人员实际需要专注的内容
        }
    };
}
```

### 0.3.2.3 下行request
下发 request 的行为必然会产生 response。但是开发人员未必关心 response。笔者规划了两种发送接口：send/call<sup>[1](#fn1)</sup>
* send 可在任何场景下使用，发送请求后就返回，不关心是否成功，也忽略 response，只打印日志以备查询
* call 有完善的状态追踪，会阻塞调用线程等待成功，故只应在 API<sup>[2](#fn2)</sup>中使用。

```c++
struct EquipmentResult{
    int result;  // 0=ok -1=error -2=timeout
    json resp;
};

class EquipmentController{
    // 不关注response，调用完driver->send即返回
    void send(const std::string& method, 
        const nlohmann::json& params);

    // 业务层会阻塞并等待response
    // 需要追踪response（session管理）
    // 如果超时，需要代为生成超时response
    EquipmentResult call(const std::string& method, 
        const nlohmann::json& params);
};
```
<span id="send-call-domain"></span>
<a id="fn1"></a>**[1]**: 有关 send/call 的介绍，以及会话管理、状态追踪、超时管理等概念，篇幅较长，且是 ArgusEdge 的主题，留待下一篇详述

<a id="fn2"></a>**[2]**: 后续会有针对 API 模块的系列专题,现在只需知道 API 模块有一组线程池，天然适合应对阻塞状况。并且 engine 中调用 call 会退化成 send，确保 engine 不会被阻塞

```c++
Cmd1_req_t req;
json json_req = to_json(req);
gaia->equipment()->send("cmd1", json_req);

//API中
Cmd2_req_t req;
json json_req = to_json(req);
EquipmentResult ret = gaia->equipment()->call("cmd2", json_req);
if(ret.result != 0){
    //异常处理
} else {
    Cmd2_resp_t resp = from_json(ret.resp);
    // 后续处理
}
```
用 send/call 封装后的通讯接口，开发人员只需要专注于逻辑：*发送一个指令* | *处理指令回报*，至于其他复杂逻辑，都被 Gaia framework 接管了。

## 0.3.3 Helios的神谕
透传，意味着框架不干预、不猜测、不绑定。它只是忠实地传递每一条消息，就像阳光普照万物，却不问万物如何生长。

这就是 HeliosEquipment 的设计哲学：它不操纵，只注视；不言语，只记录。它什么都没做，但它的光芒永远闪耀。

就像那位洞见一切的神祇，太阳神赫利俄斯。


<!--
┌──────────────┐      ┌─────┐    ┌──────────────┐
│              ├──────│req 1├───►│              │
│              │      └─────┘    │              │
│              │   ┌─────┐       │              │
│              │───│req 2├──────►│              │
│              │   └─────┘       │              │
│   Helios     │      ┌─────┐    │   Equipment  │
│  Equipment   │◄─────│resp1├────│    Driver    │
│              │      └─────┘    │              │
│              │   ┌─────┐       │              │
│              │◄──│resp2├───────│              │
│              │   └─────┘       │              │
│              │   ┌────────┐    │              │
│              │◄──│notify 1│────│              │
│              │   └────────┘    │              │
└──────────────┘                 └──────────────┘
-->