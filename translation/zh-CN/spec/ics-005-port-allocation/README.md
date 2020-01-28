---
ics: 5
title: 端口分配
stage: 草案
requires: 24
required-by: 4
category: IBC/TAO
kind: 接口
author: Christopher Goes <cwgoes@tendermint.com>
created: 2019-06-20
modified: 2019-08-25
---

## 概要

该标准指定了端口分配系统，模块可以通过该系统绑定到由 IBC 处理程序分配的唯一命名的端口。 然后可以将端口用于创建通道，并且可以被最初绑定到端口的模块转移或释放。

### 动机

区块链间通信协议旨在促进模块之间的通信，其中模块是独立的，可能相互不信任，在自治账本上执行的自成一体的代码。为了提供所需的端到端语义，IBC 处理程序必须实现对特定模块许可的通道。 该规范定义了实现该模型的*端口分配和所有权*系统。

关于哪种模块逻辑可以绑定到特定的端口名称的约定可能会出现，例如用于处理同质通证的“bank”或用于链间抵押的“staking”。 这类似于 HTTP 服务器的 80 端口的惯用用法-该协议无法强制将特定的模块逻辑实际上绑定到惯用端口，因此用户必须自己检查。可以创建具有伪随机标识符的临时端口以用于临时协议处理。

模块可以绑定到多个端口，并连接到单独计算机上另一个模块绑定的多个端口。任何数量的（唯一标识的）通道都可以同时使用一个端口。通道是在两个端口之间的端到端的，每个端口必须事先已被模块绑定，然后模块将控制该通道的一端。

（可选）主机状态机可以选择将端口绑定通过生成专门用于绑定端口的功能键的方式暴露给特别允许的模块管理器 。然后模块管理器可以使用自定义规则集控制模块可以绑定到哪些端口，和转移端口到仅已验证端口名称和模块的模块。路由模块可以扮演这个角色（请参阅 [ICS 26](../ics-026-routing-module) ）。

### 定义

`Identifier` ， `get` ， `set`和`delete`的定义与 [ICS 24](../ics-024-host-requirements) 中的相同。

*端口*是一种特殊的标识符，用于许可模块创建和对使用通道。

*模块*是主机状态机的子组件，独立于 IBC 处理程序。示例包括以太坊智能合约和  Cosmos SDK和 Substrate 的模块。 除了主机状态机可以使用对象功能或源身份验证来访问模块的许可端口的能力之外，IBC 规范不对模块功能进行任何假设。

### 所需属性

- 一个模块绑定到端口后，其他模块将无法使用该端口，直到该模块释放它
- 一个模块可以选择释放端口或将其转移到另一个模块
- 单个模块可以一次绑定到多个端口
- 分配端口时，先到先得，先绑定先服务，链可以在第一次启动时将已知模块绑定“保留”端口。

作为一个有用的比较，以下对 TCP 的类比大致准确：

IBC Concept | TCP/IP Concept | Differences
--- | --- | ---
IBC | TCP | Many, see the architecture documents describing IBC
Port (e.g. "bank") | Port (e.g. 80) | No low-number reserved ports, ports are strings
Module (e.g. "bank") | Application (e.g. Nginx) | Application-specific
Client | - | No direct analogy, a bit like L2 routing and a bit like TLS
Connection | - | No direct analogy, folded into connections in TCP
Channel | Connection | Any number of channels can be opened to or from a port simultaneously

## 技术指标

### 数据结构

The host state machine MUST support either object-capability reference or source authentication for modules.

In the former object-capability case, the IBC handler must have the ability to generate *object-capabilities*, unique, opaque references which can be passed to a module and will not be duplicable by other modules. Two examples are store keys as used in the Cosmos SDK ([reference](https://github.com/cosmos/cosmos-sdk/blob/97eac176a5d533838333f7212cbbd79beb0754bc/store/types/store.go#L275)) and object references as used in Agoric's Javascript runtime ([reference](https://github.com/Agoric/SwingSet)).

```typescript
type CapabilityKey object
```

```typescript
function newCapabilityPath(): CapabilityKey {
  // provided by host state machine, e.g. pointer address in Cosmos SDK
}
```

In the latter source authentication case, the IBC handler must have the ability to securely read the *source identifier* of the calling module, a unique string for each module in the host state machine, which cannot be altered by the module or faked by another module. An example is smart contract addresses as used by Ethereum ([reference](https://ethereum.github.io/yellowpaper/paper.pdf)).

```typescript
type SourceIdentifier string
```

```typescript
function callingModuleIdentifier(): SourceIdentifier {
  // provided by host state machine, e.g. contract address in Ethereum
}
```

`generate` and `authenticate` functions are then defined as follows.

In the former case, `generate` returns a new object-capability key, which must be returned by the outer-layer function, and `authenticate` requires that the outer-layer function take an extra argument `capability`, which is an object-capability key with uniqueness enforced by the host state machine. Outer-layer functions are any functions exposed by the IBC handler ([ICS 25](../ics-025-handler-interface)) or routing module ([ICS 26](../ics-026-routing-module)) to modules.

```
function generate(): CapabilityKey {
    return newCapabilityPath()
}
```

```
function authenticate(key: CapabilityKey): boolean {
    return capability === key
}
```

In the latter case, `generate` returns the calling module's identifier and `authenticate` merely checks it.

```typescript
function generate(): SourceIdentifier {
    return callingModuleIdentifier()
}
```

```typescript
function authenticate(id: SourceIdentifier): boolean {
    return callingModuleIdentifier() === id
}
```

#### 储存路径

`portPath` takes an `Identifier` and returns the store path under which the object-capability reference or owner module identifier associated with a port should be stored.

```typescript
function portPath(id: Identifier): Path {
    return "ports/{id}"
}
```

### 子协议

#### 标识符验证

Owner module identifier for ports are stored under a unique `Identifier` prefix. The validation function `validatePortIdentifier` MAY be provided.

```typescript
type validatePortIdentifier = (id: Identifier) => boolean
```

If not provided, the default `validatePortIdentifier` function will always return `true`.

#### 绑定到端口

The IBC handler MUST implement `bindPort`. `bindPort` binds to an unallocated port, failing if the port has already been allocated.

If the host state machine does not implement a special module manager to control port allocation, `bindPort` SHOULD be available to all modules. If it does, `bindPort` SHOULD only be callable by the module manager.

```typescript
function bindPort(id: Identifier) {
    abortTransactionUnless(validatePortIdentifier(id))
    abortTransactionUnless(privateStore.get(portPath(id)) === null)
    key = generate()
    privateStore.set(portPath(id), key)
    return key
}
```

#### 转让端口所有权

If the host state machine supports object-capabilities, no additional protocol is necessary, since the port reference is a bearer capability. If it does not, the IBC handler MAY implement the following `transferPort` function.

`transferPort` SHOULD be available to all modules.

```typescript
function transferPort(id: Identifier) {
    abortTransactionUnless(authenticate(privateStore.get(portPath(id))))
    key = generate()
    privateStore.set(portPath(id), key)
}
```

#### 释放端口

The IBC handler MUST implement the `releasePort` function, which allows a module to release a port such that other modules may then bind to it.

`releasePort` SHOULD be available to all modules.

> Warning: releasing a port will allow other modules to bind to that port and possibly intercept incoming channel opening handshakes. Modules should release ports only when doing so is safe.

```typescript
function releasePort(id: Identifier) {
    abortTransactionUnless(authenticate(privateStore.get(portPath(id))))
    privateStore.delete(portPath(id))
}
```

### 属性和不变量

- By default, port identifiers are first-come-first-serve: once a module has bound to a port, only that module can utilise the port until the module transfers or releases it. A module manager can implement custom logic which overrides this.

## 向后兼容性

不适用。

## 向前兼容性

Port binding is not a wire protocol, so interfaces can change independently on separate chains as long as the ownership semantics are unaffected.

## 示例实现

即将发布。

## 其他实现

即将发布。

## 历史

2019年6月29日-初稿

## 版权

本文中的所有内容均根据 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 获得许可。
