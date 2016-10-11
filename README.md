介绍
====

[![Go Foundation](https://img.shields.io/badge/go-foundation-green.svg)](http://golangfoundation.org)
[![Go Report Card](https://goreportcard.com/badge/github.com/funny/fastway)](https://goreportcard.com/report/github.com/funny/fastway)
[![Build Status](https://travis-ci.org/funny/fastway.svg?branch=master)](https://travis-ci.org/funny/fastway)
[![codecov](https://codecov.io/gh/funny/fastway/branch/master/graph/badge.svg)](https://codecov.io/gh/funny/fastway)
[![GoDoc](https://img.shields.io/badge/api-reference-blue.svg)](https://godoc.org/github.com/funny/fastway/go)

本网关是一个游戏用网关，它负责客户端和服务端之间的消息转发。

通过网关，我们预期达到以下目的：

+ 减少暴露在公网的服务器
+ 提升网络故障转移的效率
+ 复用客户端网络连接
+ 使服务端主动连接客户端成为可能
+ 不修改服务端的情况下引入断线重连机制和加密机制

说明
====

<p align="center"><img src="https://rawgit.com/funny/fastway/master/README.svg" alt="Gateway" /></p>
<p align="center"><b>图1 - 拓扑结构</b></p>

基本逻辑：

+ 每个系统允许部署任意个网关
+ 每个网关启动时守护对外和对内两个网络端口
+ 游戏服务端启动时主动连接到所有网关的对内端口，并告知网关自己的ID
+ 新的客户端连接产生时，随机连接到一个网关的对外端口
+ 客户端使用服务端ID来建立虚拟连接，每个客户端可以建立多个虚拟连接
+ 网关为每个虚拟连接分配一个ID，并告知服务端有一个新的虚拟连接产生
+ 客户端和服务端拿到虚拟连接ID之后即可使用此虚拟连接进行后续通讯
+ 通讯过程中通过在消息头部附加虚拟连接ID来告知网关此消息的去处
+ 服务端在已知客户端连接ID的情况下，可以主动连接客户端

设计意图和实现细节：

+ 允许部署任意个网关的目的是实现负载均和防单点故障
+ 网关可以开启reuseport特性，提升多核利用率
+ 由服务端主动连接网关目的是倒置依赖性，降低网关复杂度，网关服务注册和发现由用户自定义
+ 客户端具体如何连接到网关可以由用户自定义，可以是根据负载情况或地域分配等
+ 为防止恶意攻击，网关可以限制每个客户端虚拟连接数
+ 网关运用零拷贝和内存池来提升消息处理效率
+ 服务端主动连接客户端时，客户端连接ID可以是RPC获取或Redis存储，具体实现由用户自定义

命令行
=====

本网关提供了一个命令行程序用来对外提供服务。

命令行公共参数：

| 参数名 | 说明 | 默认值 |
| --- | --- | --- |
| ReusePort | 是否开启reuseport特性 | false |
| MaxPacketSize | 最大的消息包体积 | 512K |
| MemPoolType | [slab内存池类型 (sync、atom或chan)](https://github.com/funny/slab) | atom |
| MemPoolFactor | slab内存池的Chunk递增指数 | 2 |
| MemPoolMaxChunk | slab内存池中最大的Chunk大小 | 64K |
| MemPoolMinChunk | slab内存池中最小的Chunk大小 | 64B |
| MemPoolPageSize | slab内存池的每个Slab内存大小 | 1M |

客户端相关参数：

| 参数名 | 说明 | 默认值 |
| --- | --- | --- |
| ClientAddr | 网关暴露给客户端的地址 | ":0" |
| ClientMaxConn | 每个客户端可以创建的最大虚拟连接数 | 16 |
| ClientBufferSize | 每个客户端连接使用的 bufio.Reader 缓冲区大小 | 2K |
| ClientDeadline | 客户端连接不活跃超过此项设置的时长，连接将被关闭 | 30秒 |
| ClientSendChanSize | 每个客户端连接异步发送消息用的chan缓冲区大小 | 1000 |
| ClientSnetEnable | [是否为客户端开启snet协议](https://github.com/funny/snet) | false |
| ClientSnetEncrypt | 是否为客户端开启snet加密 | false |
| ClientSnetBuffer | 每个客户端物理连接对应的snet重连缓冲区大小 | 32k |
| ClientSnetInitTimeout | 客户端snet握手超时时间 | 10秒 |
| ClientSnetWaitTimeout | 客户端snet等待重连超时时间 | 60秒 |

服务端相关参数：

| 参数名 | 说明 | 默认值 |
| --- | --- | --- |
| ServerAddr | 网关暴露给服务端的地址 | ":0" |
| ServerAuthPassword | 用于验证服务端合法性的秘钥 | 空 |
| ServerAuthTimeout | 验证服务端连接时的最大IO等待时间 | 3秒 |
| ServerBufferSize | 每个服务端连接使用的 bufio.Reader 缓冲区大小 | 64K |
| ServerDeadline | 服务端物理连接不活跃超过此项设置的时长，连接将被关闭 | 30秒 |
| ServerSendChanSize | 每个服务端连接异步发送消息用的chan缓冲区大小 | 10万 |
| ServerSnetEnable | [是否为服务端开启snet协议](https://github.com/funny/snet) | false |
| ServerSnetEncrypt | 是否为服务端开启snet加密 | false |
| ServerSnetBuffer | 每个服务端物理连接对应的snet重连缓冲区大小 | 1M |
| ServerSnetInitTimeout | 服务端snet握手超时时间 | 10秒 |
| ServerSnetWaitTimeout | 服务端snet等待重连超时时间 | 60秒 |

API
===

本网关目前支持以下编程语言的调用：

+ [Go版](https://github.com/fast/fastway/tree/master/go)
+ [C#版](https://github.com/fast/fastway/tree/master/csharp)

通讯协议
=======

客户端和服务端均采用相同的协议格式与网关进行通讯，消息由4个字节包长度信息和变长的包体组成：

```
+--------+--------+
| Length | Packet |
+--------+--------+
  4 byte   Length
```

每个消息的包体可以再分解为4个字节的虚拟连接ID和变长的消息内容两部分：

```
+---------+-------------+
| Conn ID |   Message   |
+---------+-------------+
   4 byte    Length - 4
```

`Conn ID`是虚拟连接的唯一标识，网关通过识别`Conn ID`来转发消息。

当`Conn ID`为0时，`Message`的内容被网关进一步解析并作为指令进行处理。

网关指令固定以一个字节的指令类型字段开头，指令参数根据指令类型而定：

```
+--------+------------------+
|  Type  |       Args       |
+--------+------------------+
  1 byte    Length - 4 - 1
```

目前支持的指令如下：

| **Type** | **用途** | **Args** | **说明** |
| ---- | ---- | ---- | ---- |
| 0 | Dial | Remote ID | 创建虚拟连接 |
| 1 | Accept | Conn ID + Remote ID | 由网关发送给虚拟连接创建者，告知虚拟连接创建成功 |
| 2 | Connect | Conn ID + Remote ID | 由网关发送给虚拟连接的被连接方，告知有新的虚拟连接产生 |
| 3 | Refuse | Remote ID | 由网关发送给虚拟连接创建者，告知无法连接到远端 |
| 4 | Close | Conn ID | 客户端、服务端、网关都有可能发送次消息 |
| 5 | Ping | 无 | 客户端和服务端在idle timeout时间范围内通过此消息保活，网关收到后回发同样消息 |

特殊说明：

+ 协议允许客户端主动连接服务端，也允许服务端主动连接客户端。
+ 新建虚拟连接的时候，网关会把虚拟连接信息发送给两端。
+ 客户端连接服务端时：
	+ 网关回发的`Accept`指令中`Remote ID`为服务端ID
	+ 发送给服务端的`Connect`指令中`Remote ID`为客户端ID。
+ 服务端连接客户端时：
	+ 网关回发的`Accept`指令中`Remote ID`为客户端ID
	+ 发送给客户端的`Connect`指令中`Remote ID`为服务端ID。

握手协议
=======

服务端在连接网关时需要先进行握手来验证服务端的合法性。

握手过程如下：

0. 合法的服务端应持有正确的网关秘钥。
1. 网关在接受到新的服务端连接之后，向新连接发送一个`uint64`范围内的随机数作为挑战码。
2. 服务端收到挑战码后，拿出秘钥，计算 `MD5(挑战码 + 秘钥)`，得到验证码。
3. 服务端将验证码和自身节点ID一起发送给网关。
4. 网关收到消息后同样计算 `MD5(挑战码 + 秘钥)`，跟收到的验证码比对是否一致。
5. 验证码比对一致，网关将新连接登记为对应节点ID的连接。

握手下行数据格式：

```
+----------------+
| Challenge Code |
+----------------+
      8 byte
```

握手上行数据格式：

```
+-----------+-----------+
|    MD5    | Server ID |
+-----------+-----------+
   16 byte      4 byte
```

协议示例
=======

客户端请求网关创建虚拟连接到服务端：

```
+------------+-------------+---------+---------------+
| Length = 9 | Conn ID = 0 | CMD = 0 | Server ID = 1 |
+------------+-------------+---------+---------------+
	4 byte        4 byte      1 byte        4 byte
```

网关响应虚拟连接创建请求：

```
+-------------+-------------+---------+----------------+---------------+
| Length = 13 | Conn ID = 0 | CMD = 1 | Conn ID = 9527 | Server ID = 1 |
+-------------+-------------+---------+----------------+---------------+
	4 byte        4 byte       1 byte       4 byte           4 byte
```

网关告知服务端有新的虚拟连接：

```
+-------------+-------------+---------+----------------+---------------+
| Length = 13 | Conn ID = 0 | CMD = 2 | Conn ID = 9527 | Server ID = 1 |
+-------------+-------------+---------+----------------+---------------+
	4 byte        4 byte       1 byte       4 byte           4 byte
```

客户端通过虚拟连接发送"Hello"到服务端：

```
+------------+----------------+-------------------+
| Length = 9 | Conn ID = 9527 | Message = "Hello" |
+------------+----------------+-------------------+
	4 byte         4 byte            5 byte
```

客户端关闭虚拟连接：

```
+------------+-------------+---------+----------------+
| Length = 9 | Conn ID = 0 | CMD = 5 | Conn ID = 9527 |
+------------+-------------+---------+----------------+
	4 byte        4 byte      1 byte       4 byte
```

相关项目
=======

* [通讯协议代码生成](https://github.com/funny/fastbin)
* [slab算法的内存池](https://github.com/funny/slab)
* [支持断线重连和加密的流协议](https://github.com/funny/snet)

参与项目
=======

欢迎提交通过github的issues功能提交反馈或提问。

技术群：474995422