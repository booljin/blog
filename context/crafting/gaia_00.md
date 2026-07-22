



# 0.2 ArgusEdge:基本设计

## 0.2.1 Edge是什么
工业领域的实际应用，通常会有一些边缘计算节点，负责部分复杂计算。比如一个视觉检测流水线，每个产品通过检测通道时，会被若干个相机拍照，并由边缘计算设备Edge进行分析并上传，最后由检测服务汇总检测结果并进行综合决策。

## 0.2.2 命名彩蛋
Argus阿尔戈斯是希腊神话中的百眼巨人。他们遍布在产线的各处，始终盯着每一个产品。

## 0.2.3 EdgeDriver
<pre>
┌──────────────────────────────────────────────────────────────┐
│                        Edge Driver                           │
│                                                              │
│Functions                    Callbacks                        │
│┌──────────────────────────┐ ┌───────────────────────────────┐│
││Start()                   │ │OnConnect(ConnectionID conn)   ││
││Stop()                    │ │OnDisconnect(ConnectionID conn)││
││Send(ConnectionID conn,   │ │OnData(ConnectionID conn,      ││
││     Byte[] data,         │ │       String cmd,             ││
││     SizeT len)           │ │       Byte[] data)            ││
││Kickoff(ConnectionID conn)│ │                               ││
│└──────────────────────────┘ └───────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
</pre>
Gaia不绑定任何具体Edge版本，无论是基于socket+私有协议，还是基于websocket+jsonrpc,都需要通过自行开发的driver来驱动。它需要实现一些功能（Functions）和一组回调（Callbacks），才能顺利接入GaiaFramework。

|||
|---|---|
|Callbacks||
|OnConnect<br>OnDisconnection|自行管理连接，并且通知Gaia，由Gaia进一步通知业务层
|OnData|从物理连接接收数据，拆除FrameHeader并传递给Gaia<br>上行通道，cmd区分 **handshake** / **notify** / **response** 。所有客户端vs服务端的通讯都必然支持。<br>handshake鉴权是有状态长连接服务标配<br>notify特指从 *服务方（Edge）* 单方面向 *需求方（Gaia）* 发送的信息。该信息不需要回复<br>response是request的回复，*需求方（Gaia）* 发送Request给 *服务方（Edge）* ，并在此接收Response
|Functions||
|Start<br>Stop|一个典型的driver通常会含有一些连接信息，差异都在构造时自行处理<sup>[1](fn1)</sup>, 确保Start和Stop接口统一。|
|Send|下一篇会着重介绍 **ArgusEdge** 的 **会话管理** 和 **Send/Call语义**，Driver的Send只需要添加FrameHeader并发送即可|
|Kickoff|支持业务层主动关闭某个连接|


## 0.2.3

# 0.3 ArgusEdge:会话（Session）管理以及send/call语义







