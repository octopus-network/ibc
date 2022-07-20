---
ics: '30'
title: IBC 中间件
stage: 草案
category: IBC/APP
requires: 4, 25, 26
kind: 实例化
author: Aditya Sripal <aditya@interchain.berlin>, Ethan Frey <ethan@confio.tech>
created: '2021-06-01'
modified: '2021-06-18'
---

## 概要

该标准文档规定了一个模块必须实现的接口和状态机逻辑，以便充当核心 IBC 和下层应用程序之间的中间件。 IBC 中间件可实现对应用程序功能的任意扩展，而无需更改应用程序或核心 IBC。

### 动机

IBC applications are designed to be self-contained modules that implement their own application-specific logic through a set of interfaces with the core IBC handlers. These core IBC handlers, in turn, are designed to enforce the correctness properties of IBC (transport, authentication, ordering) while delegating all application-specific handling to the IBC application modules. However, there are cases where some functionality may be desired by many applications, yet not appropriate to place in core IBC. The most prescient example of this, is the generalized fee payment protocol. Most applications will want to opt in to a protocol that incentivizes relayers to relay packets on their channel. However, some may not wish to enable this feature and yet others will want to implement their own custom fee handler.

如果没有中间件方案，开发人员必须选择是将此扩展置于每个相关应用的内部逻辑中或将这段应用逻辑放在核心 IBC 中。将它放在每个应用程序中是冗余的并且容易出错；将逻辑放置在核心 IBC 中需要所有应用程序都选用，并且违反了核心 IBC (TAO) 和应用程序之间的抽象隔离。随着扩展数量的增加，这两种情况都不可扩展，因为这都必然会使应用程序或核心 IBC 处理程序中的代码膨胀。

中间件允许开发人员将扩展定义为可以包裹基础应用程序的单独模块。因此，该中间件可以执行自己的自定义逻辑，并将数据传递给应用程序，以便它可以在不知道中间件存在的情况下运行其逻辑。这允许应用程序和中间件实现自己隔离的逻辑，同时仍然能够作为一个数据包流程的一个环节来运行。

### 定义

`Middleware` ：在数据包执行期间，位于核心 IBC 和下层 IBC 应用程序之间的自成一体的模块。核心 IBC 和下层应用程序之间的所有消息都必须流经中间件，中间件可以执行自己的自定义逻辑。 

`Underlying Application` ：下层应用程序是直接连接到相关中间件的应用程序。该下层应用程序自身可能是链接到基础应用程序的中间件。

`Base Application` ：基础应用程序是不包含任何中间件的 IBC 应用程序。它可以被 0 个或多个中间件嵌套，形成一个应用栈。

`Application Stack (or stack)` ：栈是连接到核心 IBC 的一整套应用程序逻辑（一个或多个中间件 + 基础应用程序）。栈可能只是一个基础应用程序，也可能是一系列嵌套了基础应用程序的中间件。

### 所需属性

- 中间件支持应用程序逻辑的任意扩展
- 中间件可以任意嵌套，以形成一个由应用扩展组成的链条
- 核心 IBC 无需更改
- 基本应用程序逻辑不需要改变

## Technical Specification

### 总体设计

为了实现 IBC 中间件的功能，模块必须实现 IBC 应用程序回调并将预处理数据传递给嵌套的应用程序。模块还必须实现`WriteAcknowledgement`和`SendPacket` 。它们将由终端应用程序调用，以便模块可以在将数据传递到核心 IBC 之前对信息进行后处理。

当嵌套应用程序时，模块必须确保它处于核心 IBC 与应用程序双向通信的中间位置。开发人员应该通过直接向 IBC 路由器（而不是任何嵌套应用程序）注册顶级模块来做到这一点。反过来，嵌套应用程序必须只能访问中间件的`WriteAcknowledgement`和`SendPacket` ，而不是直接访问核心 IBC 处理程序。

此外，中间件必须注意确保应用程序逻辑可以执行自己的版本协商，而不会受到嵌套中间件的干扰。为了做到这一点，中间件将使用自己的中间件版本字符串预先添加版本。在应用回调中，中间件必须对前缀进行自己的版本协商，然后在将数据交给嵌套应用的回调之前去掉前缀。该要求仅限于中间件期望在交易对手栈上的同一级别上具有兼容的交易对手中间件时的情况。只在通道的单侧执行的中间件，不得修改通道版本。

版本： `{middleware_version}:{app_version}`

每个应用程序栈都必须为核心 IBC 保留自己的唯一端口。因此，具有相同基础应用程序的两个栈必须绑定到不同的端口。

#### 接口

```typescript
// Middleware implements the ICS26 Module interface
interface Middleware extends ICS26Module {
    app: ICS26Module // middleware has acccess to an underlying application which may be wrapped by more middleware
    ics4Wrapper: ICS4Wrapper // middleware has access to ICS4Wrapper which may be core IBC Channel Handler or a higher-level middleware that wraps this middleware.
}
```

```typescript
// This is implemented by ICS4 and all middleware that are wrapping base application.
// The base application will call `sendPacket` or `writeAcknowledgement` of the middleware directly above them
// which will call the next middleware until it reaches the core IBC handler.
interface ICS4Wrapper {
    sendPacket(
      capability: CapabilityKey,
      sourcePort: Identifier,
      sourceChannel: Identifier,
      timeoutHeight: Height,
      timeoutTimestamp: uint64,
      data: bytes)
    writeAcknowledgement(packet: Packet, ack: Acknowledgement)
}
```

#### 握手回调

```typescript
function onChanOpenInit(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string) {
    middlewareVersion, appVersion = splitMiddlewareVersion(version)
    doCustomLogic()
    app.OnChanOpenInit(
        order,
        connectionHops,
        portIdentifier,
        channelIdentifier,
        counterpartyPortIdentifier,
        counterpartyChannelIdentifier,
        appVersion, // note we only pass app version here
    )
}

function OnChanOpenTry(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string,
  counterpartyVersion: string) {
      cpMiddlewareVersion, cpAppVersion = splitMiddlewareVersion(counterpartyVersion)
      middlewareVersion, appVersion = splitMiddlewareVersion(version)
      if !isCompatible(cpMiddlewareVersion, middlewareVersion) {
          return error
      }
      doCustomLogic()

      // call the underlying applications OnChanOpenTry callback
      app.OnChanOpenTry(
          order,
          connectionHops,
          portIdentifier,
          channelIdentifier,
          counterpartyPortIdentifier,
          counterpartyChannelIdentifier,
          cpAppVersion, // note we only pass counterparty app version here
          appVersion, // only pass app version
      )
}

function onChanOpenAck(
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  counterpartyVersion: string) {
      middlewareVersion, appVersion = splitMiddlewareVersion(counterpartyVersion)
      if !isCompatible(middlewareVersion) {
          return error
      }
      doCustomLogic()
      
      // call the underlying applications OnChanOpenTry callback
      app.OnChanOpenAck(portIdentifier, channelIdentifier, appVersion)
}

function OnChanOpenConfirm(
    portIdentifier: Identifier,
    channelIdentifier: Identifier) {
    doCustomLogic()

    app.OnChanOpenConfirm(portIdentifier, channelIdentifier)
}

function splitMiddlewareVersion(version: string): []string {
    splitVersions = split(version,  ":")
    middlewareVersion = version[0]
    appVersion = join(version[1:], ":")
    return []string{middlewareVersion, appVersion}
}
```

注意：不需要与远程栈上的交易对手中间件协商的中间件，将不会实现版本拆分和协商，而将简单地在回调上执行自己的自定义逻辑，并不依赖于交易对手也有类似行为。

#### 数据包回调

```typescript
function onRecvPacket(packet: Packet, relayer: string): bytes {
    doCustomLogic()

    app_acknowledgement = app.onRecvPacket(packet, relayer)

    // middleware may modify ack
    ack = doCustomLogic(app_acknowledgement)
   
    return marshal(ack)
}

function onAcknowledgePacket(packet: Packet, acknowledgement: bytes, relayer: string) {
    doCustomLogic()

    // middleware may modify ack
    app_ack = getAppAcknowledgement(acknowledgement)

    app.OnAcknowledgePacket(packet, app_ack, relayer)

    doCustomLogic()
}

function onTimeoutPacket(packet: Packet, relayer: string) {
    doCustomLogic()

    app.OnTimeoutPacket(packet, relayer)

    doCustomLogic()
}

function onTimeoutPacketClose(packet: Packet, relayer: string) {
    doCustomLogic()

    app.onTimeoutPacketClose(packet, relayer)

    doCustomLogic()
}
```

注意：中间件可以对 ICS-26 中定义的所有 IBC 模块回调的底层应用程序数据进行预处理和后处理。

#### ICS-4 Wrappers

```typescript
function writeAcknowledgement(
  packet: Packet,
  acknowledgement: bytes) {
    // middleware may modify acknowledgement
    ack_bytes = doCustomLogic(acknowledgement)

    return ics4.writeAcknowledgement(packet, ack_bytes)
}
```

```typescript
function sendPacket(
  capability: CapabilityKey,
  sourcePort: Identifier,
  sourceChannel: Identifier,
  timeoutHeight: Height,
  timeoutTimestamp: uint64,
  app_data: bytes) {
    // 中间件可以修改数据包
    data = doCustomLogic(app_data)

    return ics4.sendPacket(
      capability,
      sourcePort,
      sourceChannel,
      timeoutHeight,
      timeoutTimestamp,
      data)
}
```

### 用户交互

In the case where the middleware requires some user input in order to modify the outgoing packet messages from the underlying application, the middleware MUST get this information from the user before it receives the packet message from the underlying application. It must then do its own authentication of the user input, and ensure that the user input provided to the middleware is matched to the correct outgoing packet message. The middleware MAY accomplish this by requiring that the user input to middleware, and packet message to underlying application are sent atomically and ordered from outermost middleware to base application.

### 安全模型

As seen above, IBC middleware may arbitrarily modify any incoming or outgoing data from an underlying application. Thus, developers should not use any untrusted middleware in their application stacks.

## 向后兼容性

中间件设计方案是当前 IBC 已经启用的设计模式。该 ICS 旨在标准化对 IBC 中间件的特定设计模式。核心 IBC 或任何现有应用程序无需更改。

## 向前兼容性

不适用。

## 示例实现

即将到来。

## 其他实现

即将到来。

## 历史

2021 年 6 月 22 日 - 提交草案

## 版权

本文中的所有内容均根据 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 获得许可。
