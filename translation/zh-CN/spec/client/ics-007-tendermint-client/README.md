---
ics: 7
title: Tendermint 客户端
stage: 草案
category: IBC/TAO
kind: 实例化
implements: 2
author: Christopher Goes <cwgoes@tendermint.com>
created: 2019-12-10
modified: 2019-12-19
---

## 概要

本规范文档描述了使用 Tendermint 共识的区块链客户端（验证算法）。

### 动机

使用 Tendermint 共识算法的各种状态机可能希望与其他使用 IBC 的状态机或单机进行交互。

### 定义

函数和术语如 [ICS 2](../../core/ics-002-client-semantics) 中所定义。

`currentTimestamp`如 [ICS 24](../../core/ics-024-host-requirements) 中所定义。

Tendermint 轻客户端使用 ICS 8 中定义的通用默克尔证明格式。

`hash`是一种通用的抗碰撞哈希函数，可以轻松的配置。

### 所需属性

该规范必须满足 ICS 2 中定义的客户端接口。

#### 关于“可能被欺骗了”逻辑的注释

“可能被欺骗了”检测的基本思想是，它使我们更加保守，当我们知道网络上其他地方的另一个轻客户端使用了稍微不同的更新模式时，会冻结我们的轻客户端。因为可能已经被欺骗了，即使我们实际没有被欺骗。

考虑三个链`A` ， `B`和`C`的拓扑，以及`A_1`和`A_2`两个链`A`的客户端，它们分别在链`B`和`C`上运行。依次发生以下事件：

- 链`A`在高度`h_0` 处生成一个块（正确）。
- 客户端`A_1`和`A_2`被更新到高度为`h_0`的块。
- 链`A`在高度`h_0 + n` 生成一个块（正确）。
- 客户端`A_1`已更新到高度为`h_0 + n`的块（客户端`A_2`尚未更新）。
- 链`A`生成了第二个 （矛盾的） 高度为`h_0 + k`的区块，并且`k <= n`。

*如果没有* “可能被欺骗了”，则客户端`A_2`会冻结（因为在高度`h_0 + k`处有两个有效块，它们比`A_2`的最新的区块头要新），但是*无法*冻结`A_1` ，因为`A_1`已经超过了`h_0 + k` 。

可以说，这是不利的，因为`A_1`只是“幸运”的被更新了，而`A_2`没有，并且明显一些拜占庭式的错误已经发生，应该由人或治理体系来干预处理。 “可能被欺骗了”的想法是通过让`A_1`从可配置的过去区块头开始以检测不良行为来侦测此类错误（因此，在这种情况下， `A_1`若能够从`h_0`开始检测，那么也将被冻结 ）。

这有一个灵活的参数，即`A_1`希望从多久前开始检查（当已更新到`h_0 + n`，`n`会是多大时，`A_1`仍然会愿意查找`h_0` ）？还存在一个反作用的担忧，即在解除绑定期之后，双签被认为是无成本的，我们并不想为 IBC 客户开放一个拒绝服务攻击的媒介。

因此，必要条件是`A_1`应该查找已存储的最早的区块头，但还应对证据进行“解除期限”检查，如果证据早于解除期限，则应避免冻结客户端（相对于客户端的本地时间戳）。如果担心时钟偏斜，可以添加一个轻微的增量。

## 技术指标

该规范依赖于[Tendermint 共识算法](https://github.com/tendermint/spec/blob/master/spec/consensus/consensus.md)和[轻客户端算法](https://github.com/tendermint/spec/blob/master/spec/consensus/light-client.md)的正确实例化。

### 客户端状态

Tendermint 客户端状态跟踪当前的验证人集合，信任期，解除绑定期，最新区块高度，最新时间戳（区块时间）以及可能的冻结区块高度。

```typescript
interface ClientState {
  validatorSet: List<Pair<Address, uint64>>
  trustingPeriod: uint64
  unbondingPeriod: uint64
  latestHeight: uint64
  latestTimestamp: uint64
  frozenHeight: Maybe<uint64>
}
```

### 共识状态

Tendermint 客户端会跟踪所有先前已验证的共识状态的时间戳（区块时间），验证人集和和承诺根（在取消绑定期之后可以将其清除，但不应该在之前清除）。

```typescript
interface ConsensusState {
  timestamp: uint64
  validatorSet: List<Pair<Address, uint64>>
  commitmentRoot: []byte
}
```

### 区块头

Tendermint 客户端头包括区块高度，时间戳，承诺根，完整的验证人集合以及提交该块的验证人的签名。

```typescript
interface Header {
  height: uint64
  timestamp: uint64
  commitmentRoot: []byte
  validatorSet: List<Pair<Address, uint64>>
  signatures: []Signature
}
```

### 证据

`Evidence`类型用于检测不良行为并冻结客户端，以防止进一步的数据包流。 Tendermint 客户端的`Evidence`包括两个相同高度并且轻客户端认为都是有效的区块头。

```typescript
interface Evidence {
  fromHeight: uint64
  h1: Header
  h2: Header
}
```

### 客户端初始化

Tendermint 客户端初始化要求（主观选择的）最新的共识状态，包括完整的验证人集合。

```typescript
function initialise(
  consensusState: ConsensusState, validatorSet: List<Pair<Address, uint64>>,
  height: uint64, trustingPeriod: uint64, unbondingPeriod: uint64): ClientState {
    assert(trustingPeriod < unbondingPeriod)
    assert(height > 0)
    set("clients/{identifier}/consensusStates/{height}", consensusState)
    return ClientState{
      validatorSet,
      latestHeight: height,
      latestTimestamp: consensusState.timestamp,
      trustingPeriod,
      unbondingPeriod,
      frozenHeight: null
    }
}
```

Tendermint 客户端的`latestClientHeight`函数返回最新存储的高度，该高度在每次验证了新的（较新的）区块头时都会更新。

```typescript
function latestClientHeight(clientState: ClientState): uint64 {
  return clientState.latestHeight
}
```

### 合法性判定式

Tendermint 客户端合法性检查使用[Tendermint 规范中](https://github.com/tendermint/spec/tree/master/spec/consensus/light-client)描述的二分算法。如果提供的区块头有效，那么将更新客户端状态并将新验证的承诺写入存储。

```typescript
function checkValidityAndUpdateState(
  clientState: ClientState,
  header: Header) {
    // assert trusting period has not yet passed. This should fatally terminate a connection.
    assert(currentTimestamp() - clientState.latestTimestamp < clientState.trustingPeriod)
    // assert header timestamp is less than trust period in the future. This should be resolved with an intermediate header.
    assert(header.timestamp - clientState.latestTimeStamp < trustingPeriod)
    // assert header timestamp is past current timestamp
    assert(header.timestamp > clientState.latestTimestamp)
    // assert header height is newer than any we know
    assert(header.height > clientState.latestHeight)
    // call the `verify` function
    assert(verify(clientState.validatorSet, clientState.latestHeight, header))
    // update validator set
    clientState.validatorSet = header.validatorSet
    // update latest height
    clientState.latestHeight = header.height
    // update latest timestamp
    clientState.latestTimestamp = header.timestamp
    // create recorded consensus state, save it
    consensusState = ConsensusState{header.validatorSet, header.commitmentRoot, header.timestamp}
    set("clients/{identifier}/consensusStates/{header.height}", consensusState)
    // save the client
    set("clients/{identifier}", clientState)
}
```

### 不良行为判定式

Tendermint 客户端的不良行为检查决定于在相同高度的两个冲突区块头是否都会通过轻客户端的验证。

```typescript
function checkMisbehaviourAndUpdateState(
  clientState: ClientState,
  evidence: Evidence) {
    // assert that the heights are the same
    assert(evidence.h1.height === evidence.h2.height)
    // assert that the commitments are different
    assert(evidence.h1.commitmentRoot !== evidence.h2.commitmentRoot)
    // fetch the previously verified commitment root & validator set
    consensusState = get("clients/{identifier}/consensusStates/{evidence.fromHeight}")
    // assert that the timestamp is not from more than an unbonding period ago
    assert(currentTimestamp() - evidence.timestamp < clientState.unbondingPeriod)
    // check if the light client "would have been fooled"
    assert(
      verify(consensusState.validatorSet, evidence.fromHeight, evidence.h1) &&
      verify(consensusState.validatorSet, evidence.fromHeight, evidence.h2)
      )
    // set the frozen height
    clientState.frozenHeight = min(clientState.frozenHeight, evidence.h1.height) // which is same as h2.height
    // save the client
    set("clients/{identifier}", clientState)
}
```

### 状态验证函数

Tendermint 客户端状态验证函数对照先前已验证的承诺根检查默克尔证明。

```typescript
function verifyClientConsensusState(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  clientIdentifier: Identifier,
  consensusStateHeight: uint64,
  consensusState: ConsensusState) {
    path = applyPrefix(prefix, "clients/{clientIdentifier}/consensusState/{consensusStateHeight}")
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided consensus state has been stored
    assert(root.verifyMembership(path, consensusState, proof))
}

function verifyConnectionState(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  connectionIdentifier: Identifier,
  connectionEnd: ConnectionEnd) {
    path = applyPrefix(prefix, "connections/{connectionIdentifier}")
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided connection end has been stored
    assert(root.verifyMembership(path, connectionEnd, proof))
}

function verifyChannelState(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  channelEnd: ChannelEnd) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}")
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided channel end has been stored
    assert(root.verifyMembership(path, channelEnd, proof))
}

function verifyPacketData(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64,
  data: bytes) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}/packets/{sequence}")
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided commitment has been stored
    assert(root.verifyMembership(path, hash(data), proof))
}

function verifyPacketAcknowledgement(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64,
  acknowledgement: bytes) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}/acknowledgements/{sequence}")
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided acknowledgement has been stored
    assert(root.verifyMembership(path, hash(acknowledgement), proof))
}

function verifyPacketAcknowledgementAbsence(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}/acknowledgements/{sequence}")
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that no acknowledgement has been stored
    assert(root.verifyNonMembership(path, proof))
}

function verifyNextSequenceRecv(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  nextSequenceRecv: uint64) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}/nextSequenceRecv")
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the nextSequenceRecv is as claimed
    assert(root.verifyMembership(path, nextSequenceRecv, proof))
}
```

### 属性和不变量

正确性保证和 Tendermint 轻客户端算法相同。

## 向后兼容性

不适用。

## 向前兼容性

不适用。更改客户端验证算法将需要新的客户端标准。

## 示例实现

还没有。

## 其他实现

目前没有。

## 历史

2019年12月10日-初始版本 2019年12月19日-最终初稿

## 版权

本文中的所有内容均根据 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 获得许可。
