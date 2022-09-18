---
title: "Build a WebSocket Server by Javascript"
date: "2015-08-03"
toc: false # Enable Table of Contents for specific page
categories:
  - "Blog"
  - "Project"
tags:
  - "websocket"
  - "javascript"
  - "node.js"
  - "programming"

author: "Rory"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
hideSummary: true
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/gongzili456/btree.fun/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

根据[RFC 6455](https://tools.ietf.org/html/rfc6455)文件对 WebSocket 的描述，我们可以了解到：WebSocket 是『在受控环境中运行不受信任代码的一个客户端到一个从该代码已经选择加入通信的远程主机间的全双工通信』。
<!--more-->

## What are WebSocket?

> The WebSocket Protocol enables two-way communication between a client running untrusted code in a controlled environment to a remote host that has opted-in to communications from that code.

## Why would I need WebSocket?

想象一下这个情景，你在网页上向你的好友发了一条消息，然后你就等待对方回应，但是你不知道对方什么时候才会给你回复，你可能会一直刷新页面知道你看到对方的回复，这真是太累了。这时你想如果消息能主动的推倒你的面前那该多好啊。但是 HTTP 协议是无状态的，每次传递消息都必须新建连接，所以 HTTP 无法实现这个需求。WebSocket 就是用来解决需要进行实时通信应用的，比如股票查询等。

## How do they work?

WebSocket 分两个阶段，握手和数据传输（handshake and the data transfer）。

### Handshake

握手就是建立 Client 到 Server 的互信关系，从而建立连接。 握手的过程是基于 HTTP 协议实现的，这要求 Client 和 Server 使用指定的 Header 来实现。

#### Client Header

```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
```

#### Server Header

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

这里需要主意的一点就是：Client 发起握手请求，将`Sec-WebSocket-Key`发给 Server，然后 Server 将其与 `258EAFA5-E914-47DA-95CA-C5AB0DC85B11` 拼起来（在本例子中将是 "s3pPLMBiTxaQ9kYGzzhZRbK+xOo=258EAFA5-E914-47DA-95CA-C5AB0DC85B11" ）。然后进行 `SHA-1 hash`，再将其结果进行 base64 编码。

## Data Frame

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+
```

- FIN: 1bit 指示这个消息是否是最否一个片段；
- RSV1, RSV2, RSV3: 每个 1 bit。默认为零，保留位，可自己实现定义；
- Opcode: 4 bits 定义 payload data 的解释，如果收到一个未知的操作码，接收端点必须 _将 WebSokcket 连接视为失败_。 定义了以下值。
  1. %x0 代表一个继续帧
  2. %x1 代表一个文本帧
  3. %x2 代表一个二进制帧
  4. %x3-7 保留用于未来的非控制帧
  5. %x8 代表连接关闭
  6. %x9 代表 ping
  7. %xA 代表 pong
  8. %xB-F 保留用于未来的控制帧
- Mask: 1 bit 定义 “负载数据”是否是经过掩码的。 如果设置为 1，一个掩码键出现在 masking-key，且这个是用于根据 5.3 节解掩码(unmask)“负载数据”。 从客户端发送到服务器的所有帧有这个位设置为 1。
- Payload length: 7 bits, 7+16 bits, 或者 7+64 bits： “负载数据”的长度，以字节为单位：如果是 0-125，这是负载长度。 如果是 126，之后的两字节解释为一个 16 位的无符号整数是负载长度。 如果是 127，之后的 8 字节解释为一个 64 位的无符号整数（最高有效位必须是 0）是负载长度。 多字节长度数量以网络字节顺序来表示。 注意，在所有情况下，最小数量的字节必须用于编码长度，例如，一个 124 字节长的字符串的长度不能被编码为序列 126，0，124。 负载长度是“扩展数据”长度+“应用数据”长度。 “扩展数据”长度可能是零，在这种情况下，负载长度是“应用数据”长度。
- Masking-key: 0 or 4 bytes： 客户端发送到服务器的所有帧通过一个包含在帧中的 32 位值来掩码。 如果 mask 位设置为 1，则该字段存在，如果 mask 位设置为 0，则该字段缺失。 详细信息请参见 5.3 节 客户端到服务器掩码。
- Payload data: (x+y) bytes： “负载数据”定义为“扩展数据”连接“应用数据”。
- Extension data: x bytes： “扩展数据”是 0 字节除非已经协商了一个扩展。 任何扩展必须指定“扩展数据”的长度，或长度是如何计算的，以及扩展如何使用必须在打开阶段握手期间协商。 如果存在，“扩展数据”包含在总负载长度中。
- Application data: y bytes： 任意的“应用数据”，占用“扩展数据”之后帧的剩余部分。 “应用数据”的长度等于负载长度减去“扩展数据”长度。

## Summary

代码详见[我的 github](https://github.com/gongzili456/websocket-example/tree/master)
