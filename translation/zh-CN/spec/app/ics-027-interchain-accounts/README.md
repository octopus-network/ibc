---
ics: '27'
title: 链间账户
stage: 草案
category: IBC/APP
requires: 25, 26
kind: 实例化
author: Tony Yun <yunjh1994@everett.zone>, Dogemos <josh@tendermint.com>, Sean King <sean@interchain.io>
created: '2019-08-01'
modified: '2020-07-14'
---

## 概要

该标准指定了不同链之间 IBC 通道之上的帐户管理系统的数据包数据结构，状态机处理逻辑和编码详细信息。

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
- 每个链间账户都隶属于控制链上的一个所有者账户。只有所有者账户有权控制隶属于自己的链间账户。相应的权限控制由控制链施行。
- 每条控制链都必须存储隶属于自己的所有链间账户的地址。
- 宿主链必须有能力限制链上的链间账户的功能（例如，宿主链可以决定链上的链间账户能不能参与质押）。

## 技术指标

### 总体设计

一条链可以应用链间账户协议的两部分（控制端协议和宿主端协议）或其中任意一部分。在其它宿主链上注册了链间账户的一条控制链，并不一定要允许其他控制链在自身注册链间账户，反之亦然。

该标准规定了注册链间账户和发送交易数据的总体方法，其中的交易数据将会代表所有者账户被链间账户执行。宿主链负责反序列化并执行交易数据；控制链在发送交易数据之前必须知道宿主链会如何处理交易数据，这是在创建通道的过程中控制链和宿主链通过握手达成的。

### 控制链智能合约

#### **RegisterInterchainAccount**

`RegisterInterchainAccount` 是注册链间账户的入口。它可以用所有者账户地址生成新的控制者portID。它可以绑定控制者portID并且调用04-channel的`ChanOpenInit`。控制者portID如果已经被占用，`RegisterInterchainAccount`会返回错误。 `ChannelOpenInit`事件将被触发并被如中继器链下进程检测到。链间账户通过`OnChanOpenTry`这一步骤在宿主链上注册。 这一个方法必须在`OPEN`的连接被创建之后才能使用给定的连接ID被调用。调用者必须提供完整的通道版本，其中必须包括带有完整元数据的ICA版本并且可能包括其他中间件的版本，其中中间件的作用是在通道两端包裹ICA。这将会需要通道两端的中间件信息。所以，建议ICA认证的应用自动构建ICA版本并且允许用户额外的中间件更新版本号。

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
// 检查以确保帐户尚未注册
// 在给定交易对手端口 ID 和底层连接 ID 的情况下，在链上创建一个新地址
// 调用 SetInterchainAccountAddress()
}
```

#### **AuthenticateTx**

`AuthenticateTx` 在执行`ExecuteTx`之前被调用。 `AuthenticateTx` 检查一则信息的签名者是链间账户，并且该链间账户与发送IBC数据包的对端通道portID相关联。

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
// 为给定的 portID 和 connectionID 设置活动通道。
function SetActiveChannelID(portId: Identifier, connectionId: Identifier, channelId: Identifier) returns (error){
}

// 根据 portID 和 connectionID，返回活动通道的 ID（如果存在）。
function GetActiveChannelID(portId: Identifier, connectionId: Identifier) returns (Identifier, boolean){
}

// 在状态中存储链间账户的地址。
function SetInterchainAccountAddress(portId: Identifier, connectionId: Identifier, address: string) returns (string) {
}

// 从状态中检索链间帐户。
function GetInterchainAccountAddress(portId: Identifier, connectionId: Identifier) returns (string, bool){
}
```

### 注册与操控流程

#### 注册链间账户的流程

要注册链间账户，我们需要一个链下进程（中继器）来监听`ChannelOpenInit`事件，并且有能力根据给定的连接来握手，从而创建通道。

1. 控制链根据一个给定的*链间账户所有者地址*将一个新的IBC端口和控制链portID绑定在一起。**

控制链和宿主链必须为每个链间账户记录一个`活跃的通道`。`活跃的通道`在创建通道的握手过程中被设置。这是一项安全机制，允许控制链重新获取在通道意外关闭之后，能够重新访问宿主链上的链间账户。

1. The controller chain emits an event signaling to open a new channel on this port given a connection.
2. 监听`ChannelOpenInit`事件的中继器将继续为创建通道而进行握手。
3. During the `OnChanOpenTry` callback on the host chain an interchain account will be registered and a mapping of the interchain account address to the owner account address will be stored in state (this is used for authenticating transactions on the host chain at execution time).
4. During the `OnChanOpenAck` callback on the controller chain a record of the interchain account address registered on the host chain during `OnChanOpenTry` is set in state with a mapping from portID -&gt; interchain account address. See [metadata negotiation](#Metadata-negotiation) section below for how to implement this.
5. During the `OnChanOpenAck` &amp; `OnChanOpenConfirm` callbacks on the controller &amp; host chains respectively, the [active-channel](#Active-channels) for this interchain account/owner pair, is set in state.

#### 活跃的通道

以下是一个活跃通道的数据结构样例：

控制链上的活动通道的数据结果示例：

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

控制链和宿主链必须验证任何新通道和之前活跃通道有一样的元数据，由此保证链间账户的参数保持不变，即使是活跃账户被替代了之后。元数据的`地址`不必被验证，因为它在INIT阶段的值预计为空，并且宿主链将在TRY阶段重新生成同样的地址，因为它预计将通过控制者端口ID和连接ID（两个ID都必须和之前的保持一致）生成确定性的链间账户地址。

每个链必须实现以下接口以支持链间帐户。 `IBCAccountModule`接口的`createOutgoingPacket`方法定义了创建特定类型的传出数据包的方式。类型指示应如何为主机链构建和序列化 IBC 帐户交易。通常，类型指示主机链的构建框架。 `generateAddress`定义了如何使用标识符和盐（salt）确定帐户地址的方法。建议使用盐生成地址，但不是必需的。如果该链不支持用确定性的方式来生成带有盐的地址，则可以以其自己的方式来生成。 `createAccount`使用生成的地址创建帐户。新的链间帐户不得与现有帐户冲突，并且链应跟踪是哪个对方链创建的新的链间帐户，以验证`authenticateTx`中交易签名人的权限。 `authenticateTx`验证交易并检查交易中的签名者是否具有正确的权限。成功通过身份认证后， `runTx`执行交易。

#### **元数据协商**

对方链使用`RegisterIBCAccountPacketData`注册帐户。使用通道标识符和盐可以确定性的定义链间帐户的地址。 `generateAccount`方法用于生成新的链间帐户的地址。建议通过`hash(identifier+salt)`生成地址，但是也可以使用其他方法。此函数必须通过标识符和盐生成唯一的确定性地址。

`RunTxPacketData`用于在链间帐户上执行交易。交易字节包含交易本身，并以适合于目标链的方式进行序列化。

`IBCAccountHandler`接口允许源链接收在链间帐户上执行交易的结果。

#### **元数据协商总结**

本文所述的子协议应在“链间帐户桥”模块中实现，并可以访问应用的路由和编解码器（解码器或解组器），和访问 IBC 路由模块。

- **INIT**

发起者：控制链

数据报：ChanOpenInit

作用于链：控制链

版本：

```json
{
  "Version": "ics27-1",
  "ControllerConnectionId": "self_connection_id",
  "HostConnectionId": "counterparty_connection_id",
  "Address": "",
  "Encoding": "requested_encoding_type",
  "TxType": "requested_tx_type",
}
```

用简单的文字描述就是`A`和`B`之间，链 A 想要在链 B 上注册一个链间帐户并控制它。同样，也可以反过来。

- **TRY**

发起者：中继者

数据报：ChanOpenTry

cosmos-sdk 的伪代码：https://github.com/everett-protocol/everett-hackathon/tree/master/x/interchain-account 以太坊上的链间账户的 POC：https://github.com/everett-protocol/ethereum-interchain-account

版本：

```json
{
  "Version": "ics27-1",
  "ControllerConnectionId": "counterparty_connection_id",
  "HostConnectionId": "self_connection_id",
  "Address": "interchain_account_address",
  "Encoding": "negotiated_encoding_type",
  "TxType": "negotiated_tx_type",
}
```

2019年8月1日-讨论了概念

- **ACK**

发起者：中继者

数据报：ChanOpenAck

2019年12月2日-较小修订（在以太坊上添加更多具体描述并添加链间账户）

交易对手版本：

```json
{
  "Version": "ics27-1",
  "ControllerConnectionId": "self_connection_id",
  "HostConnectionId": "counterparty_connection_id",
  "Address": "interchain_account_address",
  "Encoding": "negotiated_encoding_type",
  "TxType": "negotiated_tx_type",
}
```

Comments: On the ChanOpenAck step, the ICS27 application on the controller chain must verify the version string chosen by the host chain on ChanOpenTry. The controller chain must verify that it can support the negotiated encoding and tx type selected by the host chain. If either is unsupported, then it must return an error and abort the handshake. If both are supported, then the controller chain must store a mapping from the channel's portID to the provided interchain account address and return successfully.

#### 控制流程

Once an interchain account is registered on the host chain a controller chain can begin sending instructions (messages) to the host chain to control the account.

1. The controller chain calls `SendTx` and passes message(s) that will be executed on the host side by the associated interchain account (determined by the controller side port identifier)

Cosmos SDK 伪代码示例：

```golang
interface InterchainTxHandler {
  onAccountCreated(identifier: Identifier, address: Address)
  onTxSucceeded(identifier: Identifier, txBytes: Uint8Array)
  onTxFailed(identifier: Identifier, txBytes: Uint8Array, errorCode: Uint8Array)
}
```

1. The host chain upon receiving the IBC packet will call `DeserializeTx`.

2. The host chain will then call `AuthenticateTx` and `ExecuteTx` for each message and return an acknowledgment containing a success or error.

Messages are authenticated on the host chain by taking the controller side port identifier and calling `GetInterchainAccountAddress(controllerPortId)` to get the expected interchain account address for the current controller port. If the signer of this message does not match the expected account address then authentication will fail.

### 数据结构

`InterchainAccountPacketData` contains an array of messages that an interchain account can execute and a memo string that is sent to the host chain as well as the packet `type`. ICS-27 version 1 has only one type `EXECUTE_TX`.

```proto
message InterchainAccountPacketData  {
    enum type
    bytes data = 1;
    string memo = 2;
}
```

The acknowledgment packet structure is defined as in [ics4](https://github.com/cosmos/ibc-go/blob/main/proto/ibc/core/channel/v1/channel.proto#L135-L148). If an error occurs on the host chain the acknowledgment contains the error message.

```proto
message Acknowledgement {
  // 响应包含结果或错误，并且必须为非空
  oneof response {
    bytes  result = 21;
    string error  = 22;
  }
}
```

### 自定义逻辑

ICS-27 relies on [ICS-30 middleware architecture](../ics-030-middleware) to provide the option for application developers to apply custom logic on the success or fail of ICS-27 packets.

Controller chains will wrap `OnAcknowledgementPacket` &amp; `OnTimeoutPacket` to handle the success or fail cases for ICS-27 packets.

### 端口和通道设置

The interchain account module on a host chain must always bind to a port with the id `icahost`. Controller chains will bind to ports dynamically, as specified in the identifier format [section](#identifer-formats).

The example below assumes a module is implementing the entire `InterchainAccountModule` interface. The `setup` function must be called exactly once when the module is created (perhaps when the blockchain itself is initialized) to bind to the appropriate port.

```typescript
function setup() {
  capability = routingModule.bindPort("icahost", ModuleCallbacks{
      onChanOpenInit,
      onChanOpenTry,
      onChanOpenAck,
      onChanOpenConfirm,
      onChanCloseInit,
      onChanCloseConfirm,
      onRecvPacket,
      onTimeoutPacket,
      onAcknowledgePacket,
      onTimeoutPacketClose
    })
    claimCapability("port", capability)
}
```

一旦调用了`setup`函数，就可以通过 IBC 路由模块创建通道。

### 通道生命周期管理

链间帐户模块将接受来自另一台机器上任何模块的新通道，当且仅当：

- 正在创建的通道是有序的。
- 控制链正在进行通道初始化。

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

```typescript
// 控制器端口 ID 必须具有以下格式：`icacontroller-{ownerAddress}`
function validateControllerPortParams(portIdentifier: Identifier) {
  split(portIdentifier, "-")
  abortTransactionUnless(portIdentifier[0] === "icacontroller")
  abortTransactionUnless(IsValidAddress(portIdentifier[1]))
}
```

### 结束握手

```typescript
function onChanCloseInit(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
 	// 不允许用户发起的通道关闭链间账户通道
  return err
}
```

```typescript
function onChanCloseConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
}
```

### 数据包中继

路由模块收到数据包后调用`onRecvPacket` 。

```typescript
function onTimeoutPacketClose(packet: Packet) {
  // nothing is necessary
}
```

在路由模块发送的数据包被确认后，该模块将调用`onAcknowledgePacket`。

```typescript
function onAcknowledgePacket(
  packet: Packet,
  acknowledgement: bytes) {
    // 调用底层应用的 OnAcknowledgementPacket 回调
    // 更多信息请参见 ICS-30 中间件
}
```

```typescript
function onTimeoutPacket(packet: Packet) {
    // 调用底层应用的 OnTimeoutPacket 回调
    // 更多信息请参见 ICS-30 中间件
}
```

### 标识符格式

链间账户通道两侧的端口标识符必须遵循的这些格式，才能被正确的链间账户模块接受。

Controller Port Identifier: `icacontroller-{owner-account-address}`

Host Port Identifier: `icahost`

## 示例实现

ICS-27 的 Cosmos-SDK 实现的代码库：https://github.com/cosmos/ibc-go

## 未来的改进

未来的链间账户可能会通过引入一种 IBC 通道类型来大大简化，该通道类型是有序通道，但不会在超时时关闭通道，而是继续接收下一个数据包。如果核心 IBC 提供了这种通道类型，链间账户可能请求使用这种通道类型并删除与“活动通道”相关的所有逻辑和状态。元数据格式中的底层连接标识符的引用也可以被删除，由此元数据格式可以得到简化。

The "active channel" setting and unsetting is currently necessary to allow interchain account owners to create a new channel in case the current active channel closes during channel timeout. The connection identifiers are part of the metadata to ensure that any new channel that gets opened are established on top of the original connection. All of this logic becomes unnecessary once the channel is ordered **and** unclosable, which can only be achieved by the introduction of a new channel type to core IBC.

## 历史

Aug 1, 2019 - Concept discussed

2019年9月24日-建议草案

2019年11月8日-重大修订

2019年12月2日-较小修订（在以太坊上添加更多具体描述并添加链间账户）

2020年7月14日-主要修订

2021年4月27日-重新设计ics27规范

2021年11月11日-根据代码实现的最新变化更新翻译

2021年12月14日-根据审计和维护人员的审查对规范进行修订

## 版权

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).
