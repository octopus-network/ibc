---
ics: 27
title: 链间账户
stage: 草案
category: IBC/APP
requires: 25, 26
kind: 实例化
author: Tony Yun <yunjh1994@everett.zone>, Dogemos <josh@tendermint.com>, Sean King <sean@interchain.io>
created: 2019-08-01
modified: 2019-12-02
---

## 概要

该标准文档指定了不同链之间 IBC 通道之上的帐户管理系统的数据包数据结构，状态机处理逻辑和编码详细信息。

### 动机

ICS-27链间帐户标准规定了基于IBC的跨链账户管理协议。具备ICS-27功能的区块链可以通过交易（而不是用私钥签名）在其他具备ICS-27功能的区块链上创建并管理账户。链间账户保留了普通账户的所有功能（例如：质押，投票，发交易），但是是由另外一条链通过IBC的方式管理的，这使得控制链能够完全操控它在宿主链上注册的链间账户。

### 定义
- 宿主链：链间账户在宿主链上注册。宿主链监听来自控制链的IBC数据包，数据包内含有链间账户可执行的控制指令（例如：Cosmos SDK信息）。 
- 控制链：控制链在宿主链上注册并管理账户。控制链通过向宿主链发送IBC数据包来控制宿主链上的账户。
- 链间账户：链间账户是宿主链上的账户。链间账户拥有普通账户的所有功能。但链间账户并不通过私钥签发交易，而是通过解析控制链向宿主链发送IBC数据包触发交易。
- 链间账户所有者：链间账户所有者是控制链上的账户。每个宿主链上的链间账户都隶属于的各自位于控制链上的链间账户所有者。

IBC 处理程序接口和 IBC 路由模块接口分别在 [ICS 25](../../core/ics-025-handler-interface) 和 [ICS 26](../../core/ics-026-routing-module) 中所定义。

### 所需属性

- 无需许可：链间账户可以由任意的参与者创建，并且无需第三方的许可（例如：链上治理）。需要注意的是：不同的创建方法可能有不同的许可方案，IBC协议对于这些方案的安全性没有做出规定。
- 故障容忍：控制链无法管理其它控制链注册的控制账户。比如，如果一条控制链受到了分叉攻击，只有分叉链注册的链间账户会受到影响。
- 发送给链间账户的交易必须被顺序执行。链间账户执行交易的顺序必须和控制链发送交易的顺序一致。
- 如果一条通道被关闭了，控制链必须有能力通过创建新通道来重新访问已注册的链间账户。
- 每个链间账户都隶属于控制链上的一个所有者账户。只有所有者账户有权控制隶属于自己的链间账户。相应的权限控制由控制链实施执行。
- 每条控制链都必须存储隶属于自己的所有链间账户的地址。
- 宿主链必须有能力限制链上的链间账户的功能（例如，宿主链可以决定链上的链间账户能不能参与质押）。

## 技术指标

### 总体设计

一条链可以应用链间账户协议的两部分（控制端协议和宿主端协议）或其中任意一部分。在其它宿主链上注册了链间账户的一条控制链，并不一定要允许其他控制链在自身注册链间账户，反之亦然。

该标准规定了注册链间账户和发送交易数据的总体方法，其中的交易数据将会代表所有者账户被链间账户执行。宿主链负责反序列化并执行交易数据；控制链在发送交易数据之前必须知道宿主链会如何处理交易数据，这是在创建通道的过程中控制链和宿主链通过握手达成的。

### 控制链智能合约

#### **RegisterInterchainAccount**

`RegisterInterchainAccount` 是注册链间账户的入口。它可以用所有者账户地址生成新的控制者portID。它可以绑定控制者portID并且调用04-channel的`ChanOpenInit`。控制者portID如果已经被占用，`RegisterInterchainAccount`会返回错误。
`ChannelOpenInit`事件将被触发并被如中继器链下进程检测到。链间账户通过`OnChanOpenTry`这一步骤在宿主链上注册。 这一个方法必须在`OPEN`的连接被创建之后才能使用给定的连接ID被调用。调用者必须提供完整的通道版本，其中必须包括带有完整元数据的ICA版本并且可能包括其他中间件的版本，其中中间件的作用是在通道两端包裹ICA。这将会需要通道两端的中间件信息。所以，建议ICA认证的应用自动构建ICA版本并且允许用户额外的中间件更新版本号。

```typescript
function RegisterInterchainAccount(connectionId: Identifier, owner: string, version: string) returns (error) {
}
```

#### **SendTx**

`SendTx` 这个方法被用来发送IBC数据包，数据包中包含链间账户所有者给链间账户的指令（信息）。

```typescript
function SendTx(
  capability: CapabilityKey, 
  connectionId: Identifier,
  portId: Identifier, 
  icaPacketData: InterchainAccountPacketData, 
  timeoutTimestamp uint64) {
    // 检查portId和connectionId是否有活跃的通道， 即链间账户是否已经用该portId和connectionId注册过了
    activeChannelID, found = GetActiveChannelID(portId, connectionId)
    abortTransactionUnless(found)

    // 验证时间戳
    abortTransactionUnless(timeoutTimestamp <= currentTimestamp())

    // 验证ICA数据包
    abortTransactionUnless(icaPacketData.type == EXECUTE_TX)
    abortTransactionUnless(icaPacketData.data != nil)

    // 通过活跃的通道向宿主链发送ICA数据包
    handler.sendPacket(
      capability,
      portId, // source port ID
      activeChannelID, // source channel ID 
      0,
      timeoutTimestamp,
      icaPacketData
    )
}
```

### 宿主链智能合约

#### **RegisterInterchainAccount**

在通过握手创建通道的过程中，`RegisterInterchainAccount`在执行`OnChanOpenTry`时被调用。
```typescript
function RegisterInterchainAccount(counterpartyPortId: Identifier, connectionID: Identifier) returns (nil) {
   // 确认有同样counterpartyPortId和connectionID账户是否已经被创建
   // 根据counterpartyPortId和connectionID创建确定性的链上新地址
   // 调用SetInterchainAccountAddress()
}
```

#### **AuthenticateTx**

`AuthenticateTx` 在执行`ExecuteTx`之前被调用。
`AuthenticateTx` 检查一则信息的签名者是链间账户，并且该链间账户与发送IBC数据包的对端通道portID相关联。

```typescript
function AuthenticateTx(msgs []Any, connectionId string, portId string) returns (error) {
    // GetInterchainAccountAddress(portId, connectionId)
    // if interchainAccountAddress != msgSigner return error
}
```

#### **ExecuteTx**

执行所有者账户在控制链上发送的每则信息。

```typescript
function ExecuteTx(sourcePort: Identifier, channel Channel, msgs []Any) returns (resultString, error) {
  // 验证每则信息
  // 根据原链端口和通道的connectionID获取链间账户
  // 验证链间账户是否被信息的签名者授权
  // 执行每则信息
  // 返回交易的结果
}
```

### 实用方法

```typescript
// 根据portID和connectionID设置活跃的通道的ID
function SetActiveChannelID(portId: Identifier, connectionId: Identifier, channelId: Identifier) returns (error){
}

// 根据portID和connectionID返回活跃的通道的ID
function GetActiveChannelID(portId: Identifier, connectionId: Identifier) returns (Identifier, boolean){
}

// 存储链间账户的地址于账户状态中
function SetInterchainAccountAddress(portId: Identifier, connectionId: Identifier, address: string) returns (string) {
}

// 从账户状态中获取链间账户
function GetInterchainAccountAddress(portId: Identifier, connectionId: Identifier) returns (string, bool){
}
```

### 注册与操控流程

#### 注册链间账户的流程

要注册链间账户，我们需要一个链下进程（中继器）来监听`ChannelOpenInit`事件，并且有能力根据给定的连接完成创建通道的握手。

1. 控制链根据一个给定的*链间账户所有者地址*将一个新的IBC端口和控制链portID绑定。
这个端口将被用来在控制链和宿主链之间为一对特定的所有者账户/链间账户创建通道。只有账户的`{owner-account-address}`与绑定的端口相匹配才会被授权相应的通道（该通道是根据控制链的portID创建的）上发送IBC数据包。每条控制链将自己执行端口注册和对于控制链的访问。

2. 控制链触发一个事件，该事件标志着用这个端口号在某一连接上创建一条新通道。
  The controller chain emits an event signaling to open a new channel on this port given a connection. 
3. 监听`ChannelOpenInit`事件的中继器将会中继该事件以继续创建通道的握手。
4. 宿主链的`OnChanOpenTry`事件在链上的回调过程中，一个链间账户将被注册，并且链间账户地址与所有者账户地址的映射将会被存储在链间账户的状态（这将用来授权宿主链上的交易的执行）中。
5. 控制链的`OnChanOpenAck`事件在链上的回调过程中，一条链间账户在宿主链上在`OnChanOpenTry`过程中注册的记录会被设置到所有者的状态中，记录中包含portID与链间账户地址的映射。实施细节请参见以下的[metadata negotiation](#Metadata-negotiation)。 // Todo:
6. 控制链的`OnChanOpenAck`事件和宿主链的`OnChanOpenConfirm`事件在各自的链上回调的过程中，链间账户/所有者的[活跃的通道](#活跃的通道)被写进链间账户/所有者的状态。

#### 活跃的通道

控制链和宿主链必须为每个链间账户记录一个`活跃的通道`。`活跃的通道`在创建通道的握手过程中被设置。这是一项安全机制，允许控制链重新获取在通道意外关闭之后，能够重新访问宿主链上的链间账户。

以下是一个活跃通道的数据结构样例：

```typescript
{
 // 控制链
 SourcePortId: `icacontroller-<owner-account-address>`,
 SourceChannelId: `<channel-id>`,
 // 宿主链
 CounterpartyPortId: `icahost`,
 CounterpartyChannelId: `<channel-id>`,
}
```
如果一条通道关闭了，控制链可以使用和之前的通道同样的端口和底层连接，通过握手创建一条新通道， 来取代现有的活跃的通道。ics27通道只能在两种情况下被关闭：在超时（如果通道是有序通道）或者轻客户端受到攻击（可能性很小）。因此控制链必须有以下两种功能： 创建新的ICS027通道；重置某一对端口号（包含`{owner-account-address}`）和连接对应的活跃通道。

控制链和宿主链必须验证任何新通道和之前活跃通道有一样的元数据，由此保证链间账户的参数保持不变，即使是活跃账户被替代了之后。元数据的`地址`不必被验证，因为它在INIT阶段的值预计为空，并且宿主链将在TRY阶段重新生成同样的地址，因为它预计将通过控制者端口ID和连接ID（两个ID都必须和之前的保持一致）生成确定性的链间账户地址。

### 数据结构

每个链必须实现以下接口以支持链间帐户。 `IBCAccountModule`接口的`createOutgoingPacket`方法定义了创建特定类型的传出数据包的方式。类型指示应如何为主机链构建和序列化 IBC 帐户交易。通常，类型指示主机链的构建框架。 `generateAddress`定义了如何使用标识符和盐（salt）确定帐户地址的方法。建议使用盐生成地址，但不是必需的。如果该链不支持用确定性的方式来生成带有盐的地址，则可以以其自己的方式来生成。 `createAccount`使用生成的地址创建帐户。新的链间帐户不得与现有帐户冲突，并且链应跟踪是哪个对方链创建的新的链间帐户，以验证`authenticateTx`中交易签名人的权限。 `authenticateTx`验证交易并检查交易中的签名者是否具有正确的权限。成功通过身份认证后， `runTx`执行交易。

```typescript
type Tx = object

interface IBCAccountModule {
  createOutgoingPacket(chainType: Uint8Array, data: any)
  createAccount(address: Uint8Array)
  generateAddress(identifier: Identifier, salt: Uint8Array): Uint8Array
  deserialiseTx(txBytes: Uint8Array): Tx
  authenticateTx(tx: Tx): boolean
  runTx(tx: Tx): uint32
}
```

对方链使用`RegisterIBCAccountPacketData`注册帐户。使用通道标识符和盐可以确定性的定义链间帐户的地址。 `generateAccount`方法用于生成新的链间帐户的地址。建议通过`hash(identifier+salt)`生成地址，但是也可以使用其他方法。此函数必须通过标识符和盐生成唯一的确定性地址。

```typescript
interface RegisterIBCAccountPacketData {
  salt: Uint8Array
}
```

`RunTxPacketData`用于在链间帐户上执行交易。交易字节包含交易本身，并以适合于目标链的方式进行序列化。

```typescript
interface RunTxPacketData {
  txBytes: Uint8Array
}
```

`IBCAccountHandler`接口允许源链接收在链间帐户上执行交易的结果。

```typescript
interface InterchainTxHandler {
  onAccountCreated(identifier: Identifier, address: Address)
  onTxSucceeded(identifier: Identifier, txBytes: Uint8Array)
  onTxFailed(identifier: Identifier, txBytes: Uint8Array, errorCode: Uint8Array)
}
```

### 子协议

本文所述的子协议应在“链间帐户桥”模块中实现，并可以访问应用的路由和编解码器（解码器或解组器），和访问 IBC 路由模块。

### 端口和通道设置

创建模块时（可能是在初始化区块链本身时），必须调用一次`setup`函数，以绑定到适当的端口并创建托管地址（为模块拥有）。

```typescript
function setup() {
  relayerModule.bindPort("interchain-account", ModuleCallbacks{
    onChanOpenInit,
    onChanOpenTry,
    onChanOpenAck,
    onChanOpenConfirm,
    onChanCloseInit,
    onChanCloseConfirm,
    onSendPacket,
    onRecvPacket,
    onTimeoutPacket,
    onAcknowledgePacket,
    onTimeoutPacketClose
  })
}
```

调用`setup`功能后，即可通过 IBC 路由模块在独立链上的链间帐户模块实例之间创建通道。

管理员（具有在主机状态机上创建连接和通道的权限）负责建立与其他状态机的连接，并创建与其他链上该模块（或支持此接口的另一个模块）的其他实例的通道。该规范仅定义了数据包处理语义，并以这样一种方式定义它们：模块本身无需担心在任何时间点可能存在或不存在哪些连接或通道。

### 路由模块回调

### 通道生命周期管理

当且仅当以下情况时，机器`A`和`B`接受另一台机器上任何模块的新通道：

- 另一个模块绑定到“链间帐户”端口。
- 正在创建的通道是有序的。
- 版本字符串为空。

```typescript
function onChanOpenInit(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string) {
  // only ordered channels allowed
  abortTransactionUnless(order === ORDERED)
  // only allow channels to "interchain-account" port on counterparty chain
  abortTransactionUnless(counterpartyPortIdentifier === "interchain-account")
  // version not used at present
  abortTransactionUnless(version === "")
}
```

```typescript
function onChanOpenTry(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string,
  counterpartyVersion: string) {
  // only ordered channels allowed
  abortTransactionUnless(order === ORDERED)
  // version not used at present
  abortTransactionUnless(version === "")
  abortTransactionUnless(counterpartyVersion === "")
  // only allow channels to "interchain-account" port on counterparty chain
  abortTransactionUnless(counterpartyPortIdentifier === "interchain-account")
}
```

```typescript
function onChanOpenAck(
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  version: string) {
  // version not used at present
  abortTransactionUnless(version === "")
  // port has already been validated
}
```

```typescript
function onChanOpenConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // accept channel confirmations, port has already been validated
}
```

```typescript
function onChanCloseInit(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // no action necessary
}
```

```typescript
function onChanCloseConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // no action necessary
}
```

### 数据包中继

用简单的文字描述就是`A`和`B`之间，链 A 想要在链 B 上注册一个链间帐户并控制它。同样，也可以反过来。

```typescript
function onRecvPacket(packet: Packet): bytes {
  if (packet.data is RunTxPacketData) {
    const tx = deserialiseTx(packet.data.txBytes)
    abortTransactionUnless(authenticateTx(tx))
    return runTx(tx)
  }

  if (packet.data is RegisterIBCAccountPacketData) {
    RegisterIBCAccountPacketData data = packet.data
    identifier = "{packet/sourcePort}/{packet.sourceChannel}"
    const address = generateAddress(identifier, packet.salt)
    createAccount(address)
    // Return generated address.
    return address
  }

  return 0x
}
```

```typescript
function onAcknowledgePacket(
  packet: Packet,
  acknowledgement: bytes) {
  if (packet.data is RegisterIBCAccountPacketData)
    if (acknowledgement !== 0x) {
      identifier = "{packet/sourcePort}/{packet.sourceChannel}"
      onAccountCreated(identifier, acknowledgement)
    }
  if (packet.data is RunTxPacketData) {
    identifier = "{packet/destPort}/{packet.destChannel}"
    if (acknowledgement === 0x)
        onTxSucceeded(identifier: Identifier, packet.data.txBytes)
    else
        onTxFailed(identifier: Identifier, packet.data.txBytes, acknowledgement)
  }
}
```

```typescript
function onTimeoutPacket(packet: Packet) {
  // Receiving chain should handle this event as if the tx in packet has failed
  if (packet.data is RunTxPacketData) {
    identifier = "{packet/destPort}/{packet.destChannel}"
    // 0x99 error code means timeout.
    onTxFailed(identifier: Identifier, packet.data.txBytes, 0x99)
  }
}
```

```typescript
function onTimeoutPacketClose(packet: Packet) {
  // nothing is necessary
}
```

## 向后兼容性

不适用。

## 向前兼容性

不适用。

## 示例实现

cosmos-sdk 的伪代码：https://github.com/everett-protocol/everett-hackathon/tree/master/x/interchain-account 以太坊上的链间账户的 POC：https://github.com/everett-protocol/ethereum-interchain-account

## 其他实现

（其他实现的链接或描述）

## 历史

2019年8月1日-讨论了概念

2019年9月24日-建议草案

2019年11月8日-重大修订

2019年12月2日-较小修订（在以太坊上添加更多具体描述并添加链间账户）

## 版权

本文中的所有内容均根据 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 获得许可。
