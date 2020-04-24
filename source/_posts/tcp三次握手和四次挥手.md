---
title: tcp三次握手和四次挥手
date: 2020-04-24 15:13:22
---

> [文章链接](https://github.com/defpis/all-about-http-that-you-should-know/blob/master/docs/TCP%E4%B8%89%E6%AC%A1%E6%8F%A1%E6%89%8B%E5%92%8C%E5%9B%9B%E6%AC%A1%E6%8C%A5%E6%89%8B.md)

<!-- more -->

## 创建一个简单的服务

```typescript
import http from 'http';

const server = http.createServer(
  (req: http.IncomingMessage, res: http.ServerResponse) => {
    console.log(`${req.method} ${req.url}`);
    res.write('Hello World!');
    res.end();
  },
);

const host = 'localhost';
const port = 4000;
console.log(`Start server on ${host}:${port}`);
server.listen(port, host);
```

启动服务，并监视文件改变以重启服务

```bash
➜ npm run start:dev

> create-custom-operators-in-rxjs@1.0.0 start:dev /Users/defpis/Workspace/all-about-http-that-you-should-know
> tsnd --respawn --transpileOnly ./src

Using ts-node version 8.8.1, typescript version 3.8.3
Start server on localhost:4000
```

使用命令测试访问

```bash
➜ curl http://localhost:4000
Hello World!·
```

## 使用 Wireshark 抓包

打开 Wireshark，在过滤器中输入 `host 127.0.0.1 and port 4000`，选择 `Loopback: Io0`（本地环回接口）以监测流量。

使用 curl 命令发送请求，然后查看 Wireshark 面板，可以看到完整的网络连接与断开过程

![Wireshark 面板](https://i.loli.net/2020/04/07/Ck8v531O7VFwigT.jpg)

点击一个连接，可以查看连接的详细信息，从上到下分为

- frame：链路层封装
- null：无
- ip：网络层封装
- tcp：传输层封装

点开 tcp 传输层封包信息

```text
Transmission Control Protocol, Src Port: 51428, Dst Port: 4000, Seq: 0, Len: 0
    Source Port: 51428
    Destination Port: 4000
    [Stream index: 1]
    [TCP Segment Len: 0]
    Sequence number: 0    (relative sequence number)
    Sequence number (raw): 2901135694
    [Next sequence number: 1    (relative sequence number)]
    Acknowledgment number: 0
    Acknowledgment number (raw): 0
    1011 .... = Header Length: 44 bytes (11)
    Flags: 0x002 (SYN)
    Window size value: 65535
    [Calculated window size: 65535]
    Checksum: 0xfe34 [unverified]
    [Checksum Status: Unverified]
    Urgent pointer: 0
    Options: (24 bytes), Maximum segment size, No-Operation (NOP), Window scale, No-Operation (NOP), No-Operation (NOP), Timestamps, SACK permitted, End of Option List (EOL)
    [Timestamps]
```

首先关注 `Flags`，把它展开可以得到

```text
Flags: 0x002 (SYN)
    000. .... .... = Reserved: Not set
    ...0 .... .... = Nonce: Not set
    .... 0... .... = Congestion Window Reduced (CWR): Not set
    .... .0.. .... = ECN-Echo: Not set
    .... ..0. .... = Urgent: Not set
    .... ...0 .... = Acknowledgment: Not set
    .... .... 0... = Push: Not set
    .... .... .0.. = Reset: Not set
    .... .... ..1. = Syn: Set
    .... .... ...0 = Fin: Not set
    [TCP Flags: ··········S·]
```

显而易见，像 SYN、ACK 或 FIN 这些符号其实都是二进制的标志位

| 符号       | 十六进制 | 二进制          | 阶段                               |
| ---------- | -------- | --------------- | ---------------------------------- |
| [SYN]      | 0x002    | 000,000,000,010 | 第一次握手                         |
| [SYN, ACK] | 0x012    | 000,000,010,010 | 第二次握手                         |
| [ACK]      | 0x010    | 000,000,010,000 | 第三次握手，第二次挥手，第四次挥手 |
| [FIN, ACK] | 0x011    | 000,000,010,001 | 第一次挥手，第三次挥手             |

除此之外，tcp 封包中还包含用于验证的字段

| 字段        | 全称                  | 作用                                                   |
| ----------- | --------------------- | ------------------------------------------------------ |
| Seq/tcp.seq | Sequence number       | 表示曾经发送过数据的字节数+1，0 表示之前没有发送过数据 |
| Ack/tcp.ack | Acknowledgment number | 表示期待下次对方发送过来的 Seq 的值                    |

理解清楚这些概念就可以开始分析 tcp 连接建立的过程——三次握手。

## 三次握手

![三次握手](https://i.loli.net/2020/04/07/JKv3wlpc5zeCWPY.png)

客户端：服务器你好，我需要建立连接（第一次握手请求）

服务端：收到（第一次握手响应），正在准备，请确认连接（第二次握手请求）

客户端：收到（第二次握手响应），确认连接（第三次握手请求，服务端收到响应）

## 四次挥手

![四次挥手](https://i.loli.net/2020/04/07/m4O7U3CZnQpIksM.png)

客户端：服务器你好，我需要断开连接（第一次挥手请求）

服务端：收到（第一次挥手响应），正在准备（第二次挥手请求，客户端收到响应）

服务端：已准备好断开连接，请确认断开（第三次挥手请求）

客户端：收到（第三次挥手响应），确认断开（第四次挥手请求，服务端收到响应）
