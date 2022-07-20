---
ics: '721'
title: 非同质化通证转移
stage: draft
category: IBC/APP
requires: 25, 26
kind: 实例化
author: Haifeng Xi <haifeng@bianjie.ai>
created: '2021-11-10'
modified: '2022-05-18'
---

> This standard document follows the same design principles of [ICS 20](../ics-020-fungible-token-transfer) and inherits most of its content therefrom, while replacing `bank` module based asset tracking logic with that of the `nft` module.

## 概览

This standard document specifies packet data structure, state machine handling logic, and encoding details for the transfer of non-fungible tokens over an IBC channel between two modules on separate chains. The state machine logic presented allows for safe multi-chain `classId` handling with permissionless channel opening. This logic constitutes a *non-fungible token transfer bridge module*, interfacing between the IBC routing module and an existing asset tracking module on the host state machine, which could be either a Cosmos-style native module or a smart contract running in a virtual machine.

### 动机

Users of a set of chains connected over the IBC protocol might wish to utilize a non-fungible token on a chain other than the chain where the token was originally issued -- perhaps to make use of additional features such as exchange, royalty payment or privacy protection. This application-layer standard describes a protocol for transferring non-fungible tokens between chains connected with IBC which preserves asset non-fungibility, preserves asset ownership, limits the impact of Byzantine faults, and requires no additional permissioning.

### 定义

The IBC handler interface &amp; IBC routing module interface are as defined in [ICS 25](../../core/ics-025-handler-interface) and [ICS 26](../../core/ics-026-routing-module), respectively.

### 所需属性

- Preservation of non-fungibility (i.e., only one instance of any token is *live* across all the IBC-connected blockchains).
- Permissionless token transfers, no need to whitelist connections, modules, or `classId`s.
- Symmetric (all chains implement the same logic, no in-protocol differentiation of hubs &amp; zones).
- 容错：防止由于链 `B` 的拜占庭行为造成源自链 `A` 的通证的拜占庭式通货膨胀。

## 技术规范

### 数据结构

仅需要一个数据包数据类型： `NonFungibleTokenPacketData` ，该类型指定了类别 id，类别 uri，通证 id，通证 uri，发送地址和接收地址。

```typescript
interface NonFungibleTokenPacketData {
  classId: string
  classUri: string
  tokenIds: []string
  tokenUris: []string
  sender: string
  receiver: string
}
```

`classId` 唯一标识了正被转移的通证在发送链中所属的类别/系列集合。例如，对于兼容 ERC-1155 的智能合约，这可能是通证 ID 的高 128 位字符串表示形式。

`classUri` 是可选的，但这对于和 OpenSea 等 NFT 市场的跨链互操作性是非常有益的，在这些 NFT 市场中可以添加 [ 类别/系列集合元数据](https://docs.opensea.io/docs/contract-level-metadata) 以提供更好的用户体验。

`tokenIds` 唯一标识了正被转移的特定类别的一些通证。例如，对于兼容 ERC-1155 的智能合约，`tokenId` 可以是通证 ID 的低 128 位字符串表示形式。

每个 `tokenId` 在 `tokenUris` 中都有一个对应的条目，它指的是链外资源，通常是包含通证元数据的不可变 JSON 文件。

当通证使用 ICS-721 协议进行跨链传送时，会开始累积通证已传输的通道记录。这些记录信息会被编码到`classId` 字段中。

ICS-721 通证类别以 `{ics721Port}/{ics721Channel}/{classId}` 的形式表示，其中 `ics721Port` 和 `ics721Channel` 标识了通证到达的当前链上的通道。如果 `{classId}` 包含 `/`，那么它也必须采用 ICS-721 形式，以表明通证具有多跳记录。需要注意的是，这要求在非 IBC 通证的 `classId` 中禁止使用 `/`（斜杠字符）。

发送链可以充当源 zone 或目标 zone。当一条链不是通过最后一个前缀端口和通道对发送通证时，它充当了源 zone。当通证从源 zone 被发出，目标端口和通道将作为 `classId` 的前缀，（收到通证后） 将另一个跃点添加到通证记录中。当一条链通过最后一个前缀端口和通道发送通证时，它就充当了目标 zone。当通证从目标 zone 被发出， `classId` 上的最后一个前缀端口和通道对会被移除，（收到通证后）将撤消通证记录中的最后一个跃点。

例如，假设发生以下转移步骤：

A -&gt; B -&gt; C -&gt; A -&gt; C -&gt; B -&gt; A

1. A(p1,c1) -&gt; (p2,c2)B : A 是源 zone。B 中的 `classId` : 'p2/c2/nftClass'
2. B(p3,c3) -&gt; (p4,c4)C : B 是源 zone。C 中的 `classId` : 'p4/c4/p2/c2/nftClass'
3. C(p5,c5) -&gt; (p6,c6)A : C 是源 zone。A 中的 `classId` : 'p6/c6/p4/c4/p2/c2/nftClass'
4. A(p6,c6) -&gt; (p5,c5)C : A 是目标 zone。C 中的 `classId` : 'p4/c4/p2/c2/nftClass'
5. C(p4,c4) -&gt; (p3,c3)B : C 是目标 zone。B 中的 `classId` : 'p2/c2/nftClass'
6. B(p2,c2) -&gt; (p1,c1)A : B 是目标 zone。A 中的 `classId` : 'nftClass'

The acknowledgement data type describes whether the transfer succeeded or failed, and the reason for failure (if any).

```typescript
type NonFungibleTokenPacketAcknowledgement = NonFungibleTokenPacketSuccess | NonFungibleTokenPacketError;

interface NonFungibleTokenPacketSuccess {
  // 这是使用二进制 0x01 base64 进行编码的
  success: "AQ=="
}

interface NonFungibleTokenPacketError {
  error: string
}
```

需要注意的是，当 `NonFungibleTokenPacketData` 和 `NonFungibleTokenPacketAcknowledgement` 序列化到数据包中，两者都必须是 JSON 编码（而非 Protobuf 编码的）。

非同质化通证转移桥接模块为每个 NFT 通道维护一个单独的托管地址。

```typescript
interface ModuleState {
  channelEscrowAddresses: Map<Identifier, string>
}
```

### 子协议

此处描述的子协议应在“非同质化通证转移桥接”模块中实现，并具有对 NFT 资产跟踪模块和 IBC 路由模块的访问权限。

NFT 资产跟踪模块应实现以下功能：

```typescript
function SaveClass(
  classId: string,
  classUri: string) {
  // 创建一个由 classId 标识的新 NFT 类别
}
```

```typescript
function Mint(
  classId: string,
  tokenId: string,
  tokenUri: string,
  receiver: string) {
  // 创建一个由 <classId,tokenId> 标识的新 NFT
  // 接收方成为新铸造的 NFT 的所有者
}
```

```typescript
function Transfer(
  classId: string,
  tokenId: string,
  receiver: string) {
  // 将由 <classId,tokenId> 标识的 NFT 传输给接收方
  // 接收方成为 NFT 的新所有者
}
```

```typescript
function Burn(
  classId: string,
  tokenId: string) {
  // 销毁由 <classId,tokenId> 标识的 NFT
}
```

```typescript
function GetOwner(
  classId: string,
  tokenId: string) {
  // 返回由 <classId,tokenId> 标识的 NFT 现任所有者
}
```

```typescript
function GetNFT(
  classId: string,
  tokenId: string) {
  // 返回由 <classId,tokenId> 标识的 NFT
}
```

```typescript
function HasClass(classId: string) {
  // 如果由 classId 标识的 NFT 类别已存在，则返回 true;
  // 否则返回 false
}
```

```typescript
function GetClass(classId: string) {
  // 返回由 classId 标识的 NFT 类别
}
```

#### 端口及通道设置

`setup` 功能必须在创建模块时（可能是在初始化区块链本身时）恰好只调用一次，以绑定到适当的（由模块拥有的）端口。

```typescript
function setup() {
  capability = routingModule.bindPort("nft", ModuleCallbacks{
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

一旦 `setup` 功能已被调用，可以通过 IBC 路由模块在各自链上的非同质化通证转移模块实例之间创建通道。

此规范仅定义数据包处理语义，并以“模块本身无需担心在任何时间点，哪些 connection 或通道可能不存在”的方式对其进行定义。

#### 路由模块回调

##### 通道生命周期管理

当且仅当满足以下条件时，机器 A 和 B 都能接受来自另一台机器上任何模块的新通道：

- 创建的通道是无序的。
- 版本字符串为 `ics721-1`.

```typescript
function onChanOpenInit(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string) {
  // 仅允许无序通道
  abortTransactionUnless(order === UNORDERED)
  // 版本为 "ics721-1"
  abortTransactionUnless(version === "ics721-1")
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
  counterpartyVersion: string) {
  // 仅允许无序通道
  abortTransactionUnless(order === UNORDERED)
  // 版本为 "ics721-1"
  abortTransactionUnless(counterpartyVersion === "ics721-1")
}
```

```typescript
function onChanOpenAck(
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  counterpartyVersion: string) {
  // 端口已被验证
  // 版本为 "ics721-1"
  abortTransactionUnless(counterpartyVersion === "ics721-1")
  // 分配托管地址
  channelEscrowAddresses[channelIdentifier] = newAddress()
}
```

```typescript
function onChanOpenConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // accept channel confirmations, port has already been validated, version has already been validated
  // allocate an escrow address
  channelEscrowAddresses[channelIdentifier] = newAddress()
}
```

```typescript
function onChanCloseInit(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // 中止并返回错误，以防止用户关闭通道
  abortTransactionUnless(FALSE)
}
```

```typescript
function onChanCloseConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // 无需执行任何操作
}
```

##### 数据包中继

- 当一个非同质化通证从其源链转出时，桥接模块会在发送链上托管该通证，并在接收链上铸造相应的凭证。
- 当一个非同质化通证被转回其源链时，桥接模块会在发送链上销毁该通证，并在接收链上取消对相应锁定通证的托管。
- 当数据包超时时，其中所表示的通证会被取消托管或适当地铸造回发送方 - 具体取决于通证是从其源链转出还是转回。
- Acknowledgement data is used to handle failures, such as invalid destination accounts. Returning an acknowledgement of failure is preferable to aborting the transaction since it more easily enables the sending chain to take appropriate action based on the nature of the failure.

`createOutgoingPacket` must be called by a transaction handler in the module which performs appropriate signature checks, specific to the account owner on the host state machine.

```typescript
function createOutgoingPacket(
  classId: string,
  tokenIds: []string,
  sender: string,
  receiver: string,
  source: boolean,
  destPort: string,
  destChannel: string,
  sourcePort: string,
  sourceChannel: string,
  timeoutHeight: Height,
  timeoutTimestamp: uint64) {
  prefix = "{sourcePort}/{sourceChannel}/"
  // we are source chain if classId is not prefixed with sourcePort and sourceChannel
  source = classId.slice(0, len(prefix)) !== prefix
  tokenUris = []
  for (let tokenId in tokenIds) {
    // ensure that sender is token owner
    abortTransactionUnless(sender === nft.GetOwner(classId, tokenId))
    if source {
      // escrow token
      nft.Transfer(classId, tokenId, channelEscrowAddresses[sourceChannel])
    } else {
      // we are sink chain, burn voucher
      nft.Burn(classId, tokenId)
    }
    tokenUris.push(nft.GetNFT(classId, tokenId).GetUri())
  }
  NonFungibleTokenPacketData data = NonFungibleTokenPacketData{classId, nft.GetClass(classId).GetUri(), tokenIds, tokenUris, sender, receiver}
  ics4Handler.sendPacket(Packet{timeoutHeight, timeoutTimestamp, destPort, destChannel, sourcePort, sourceChannel, data}, getCapability("port"))
}
```

路由模块收到数据包后调用`onRecvPacket` 。

```typescript
function onRecvPacket(packet: Packet) {
  NonFungibleTokenPacketData data = packet.data
  // construct default acknowledgement of success
  NonFungibleTokenPacketAcknowledgement ack = NonFungibleTokenPacketAcknowledgement{true, null}
  err = ProcessReceivedPacketData(data)
  if (err !== null) {
    ack = NonFungibleTokenPacketAcknowledgement{false, err.Error()}
  }
  return ack
}

function ProcessReceivedPacketData(data: NonFungibleTokenPacketData) {
  prefix = "{packet.sourcePort}/{packet.sourceChannel}/"
  // we are source chain if classId is prefixed with packet's sourcePort and sourceChannel
  source = data.classId.slice(0, len(prefix)) === prefix
  for (var i in data.tokenIds) {
    if source {
      // unescrow token to receiver
      nft.Transfer(data.classId.slice(len(prefix)), data.tokenIds[i], data.receiver)
    } else { // we are sink chain
      prefixedClassId = "{packet.destPort}/{packet.destChannel}/" + data.classId
      // create NFT class if it doesn't exist already
      if (nft.HasClass(prefixedClassId) === false) {
        nft.SaveClass(data.classId, data.classUri)
      }
      // mint voucher to receiver
      nft.Mint(prefixedClassId, data.tokenIds[i], data.tokenUris[i], data.receiver)
    }
  }
}
```

在路由模块发送的数据包被确认后，该模块将调用`onAcknowledgePacket`。

```typescript
function onAcknowledgePacket(
  packet: Packet,
  acknowledgement: bytes) {
  // if the transfer failed, refund the tokens
  if (!ack.success)
    refundToken(packet)
}
```

当路由模块发出的数据包超时（使得数据包没有被目标链接收），路由模块会调用 `onTimeoutPacket` 。

```typescript
function onTimeoutPacket(packet: Packet) {
  // 数据包超时，因此退回通证
  refundToken(packet)
}
```

`refundToken` 会在两种情况下被调用，`onAcknowledgePacket` 失败时和 `onTimeoutPacket` 时，用以将托管的通证退还给原始发送者。

```typescript
function refundToken(packet: Packet) {
  NonFungibleTokenPacketData data = packet.data
  prefix = "{packet.sourcePort}/{packet.sourceChannel}/"
  // we are the source if the classId is not prefixed with the packet's sourcePort and sourceChannel
  source = data.classId.slice(0, len(prefix)) !== prefix
  for (var i in data.tokenIds) { {
    if source {
      // unescrow token back to sender
      nft.Transfer(data.classId, data.tokenIds[i], data.sender)
    } else {
      // we are sink chain, mint voucher back to sender
      nft.Mint(data.classId, data.tokenIds[i], data.tokenUris[i], data.sender)
    }
  }
}
```

```typescript
function onTimeoutPacketClose(packet: Packet) {
  // 不会发生，只允许无序通道
}
```

#### Reasoning

##### 正确性

该实现保留了通证的非同质性和可兑换性。

- 非同质性：在所有基于 IBC 连接的区块链中，任何通证只有一个 *活跃* 实例）。
- 可兑换性：如果通证已被发送到交易对手链上，仍可以在源链上的同一 `classId` 和 `tokenId` 中将其兑换回去。

#### 可选附录

- Each chain, locally, could elect to keep a lookup table to use short, user-friendly local `classId`s in state which are translated to and from the longer `classId`s when sending and receiving packets.
- Additional restrictions may be imposed on which other machines may be connected to &amp; which channels may be established.

## 进一步讨论

Extended and complex use cases such as royalties, marketplaces or permissioned transfers can be supported on top of this specification. Solutions could be modules, hooks, [IBC middleware](../ics-030-middleware) and so on. Designing a guideline for this is out of the scope.

It is assumed that application logic in host state machines will be responsible for metadata immutability of IBC tokens minted according to this specification. For any IBC token, NFT applications are strongly advised to check upstream blockchains (all the way back to the source) to ensure its metadata has not been modified along the way. If it is decided, sometime in the future, to accommodate NFT metadata mutability over IBC, we will update this specification or create an entirely new specification -- by using advanced DID features perhaps.

## 向后兼容性

不适用。

## 向前兼容性

This initial standard uses version "ics721-1" in the channel handshake.

此标准的未来版块可以在通道握手中使用其他版本，并安全地更改数据包数据格式和数据包处理程序的语义。

## 示例实现

即将到来。

## 其他实现

即将到来。

## 历史

日期 | 描述
--- | ---
2021 年 11 月 10 日 | 初始草案 - 改编自 ICS 20 规范
2021 年 11 月 17 日 | 进行修订，以更好地适应智能合约
2021 年 11 月 17 日 | 从 ICS 21 更名为 ICS 721
2021 年 11 月 18 日 | Revised to allow for multiple tokens in one packet
2022 年 2 月 10 日 | 进行修订，以纳入 IBC 团队的反馈
2022 年 3 月 3 日 | 进行修订，使 TRY 回调与 PR#629 保持一致
2022 年 3 月 11 日 | 添加示例，以说明前缀概念
2022 年 3 月 30 日 | 添加 NFT 模块定义，并修复伪代码错误
2022 年 5 月 18 日 | 添加了有关 NFT 元数据可变性的段落

## 版权

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).
