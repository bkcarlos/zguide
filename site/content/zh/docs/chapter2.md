---
weight: 2
title: '2. 套接字和模式'
---

# 第2章 - 套接字和模式 {#sockets-and-patterns}

在第一章中，我们体验了 ZeroMQ，通过一些主要 ZeroMQ 模式的基本示例：请求-回复、发布-订阅和管道。在本章中，我们将深入了解并开始学习如何在真实程序中使用这些工具。

我们将涵盖：

* 如何创建和使用 ZeroMQ 套接字。
* 如何在套接字上发送和接收消息。
* 如何围绕 ZeroMQ 的异步 I/O 模型构建应用程序。
* 如何在一个线程中处理多个套接字。
* 如何正确处理致命和非致命错误。
* 如何处理中断信号，如 Ctrl-C。
* 如何干净地关闭 ZeroMQ 应用程序。
* 如何检查 ZeroMQ 应用程序的内存泄漏。
* 如何发送和接收多部分消息。
* 如何跨网络转发消息。
* 如何构建简单的消息队列代理。
* 如何使用 ZeroMQ 编写多线程应用程序。
* 如何使用 ZeroMQ 在线程之间发送信号。
* 如何使用 ZeroMQ 协调节点网络。
* 如何为发布-订阅创建和使用消息信封。
* 使用 HWM（高水位标记）防止内存溢出。

## 套接字 API {#The-Socket-API}

说实话，ZeroMQ 对你玩了一个偷梁换柱的把戏，我们对此不道歉。这是为了你好，这比伤害你更伤害我们。ZeroMQ 呈现了一个熟悉的基于套接字的 API，这需要我们付出巨大努力来隐藏一堆消息处理引擎。然而，结果将慢慢修复你关于如何设计和编写分布式软件的世界观。

套接字是网络编程的事实标准 API，也有助于防止你的眼睛掉到脸颊上。使 ZeroMQ 对开发人员特别有吸引力的一点是它使用套接字和消息，而不是其他任意的概念集合。向 Martin Sustrik 致敬，他做到了这一点。它将"面向消息的中间件"（一个保证会让整个房间都陷入昏迷状态的短语）变成了"超级辣味套接字！"，这让我们对披萨有了奇怪的渴望，并渴望了解更多。

就像一道最喜欢的菜一样，ZeroMQ 套接字很容易消化。套接字有四个部分的生命，就像 BSD 套接字一样：

* 创建和销毁套接字，它们一起形成套接字生命的因果循环（参见 `zmq_socket()`、`zmq_close()`）。

* 通过设置选项和必要时检查它们来配置套接字（参见 `zmq_setsockopt()`、`zmq_getsockopt()`）。

* 通过创建到套接字的 ZeroMQ 连接将套接字插入网络拓扑（参见 `zmq_bind()`、`zmq_connect()`）。

* 通过在套接字上写入和接收消息来使用套接字承载数据（参见 `zmq_msg_send()`、`zmq_msg_recv()`）。

请注意，套接字总是 void 指针，消息（我们很快就会讲到）是结构体。所以在 C 中你按原样传递套接字，但在所有处理消息的函数中传递消息的地址，如 `zmq_msg_send()` 和 `zmq_msg_recv()`。作为助记符，要意识到"在 ZeroMQ 中，所有你的套接字都属于我们"，但消息是你在代码中实际拥有的东西。

创建、销毁和配置套接字的工作方式与你对任何对象的期望一样。但请记住，ZeroMQ 是一个异步的、弹性的结构。这对我们如何将套接字插入网络拓扑以及之后如何使用套接字有一些影响。

### 将套接字插入拓扑 {#Plugging-Sockets-into-the-Topology}

要在两个节点之间创建连接，你在一个节点中使用 `zmq_bind()`，在另一个节点中使用 `zmq_connect()`。作为一般经验法则，执行 `zmq_bind()` 的节点是"服务器"，位于众所周知的网络地址上，而执行 `zmq_connect()` 的节点是"客户端"，具有未知或任意的网络地址。因此我们说我们"将套接字绑定到端点"和"将套接字连接到端点"，端点是那个众所周知的网络地址。

ZeroMQ 连接与经典的 TCP 连接有些不同。主要的显著差异是：

* 它们跨任意传输（`inproc`、`ipc`、`tcp`、`pgm` 或 `epgm`）。参见相关文档。

* 一个套接字可能有许多传出和传入连接。

* 没有 `zmq_accept()` 方法。当套接字绑定到端点时，它会自动开始接受连接。

* 网络连接本身在后台发生，如果网络连接中断（例如，如果对等体消失然后回来），ZeroMQ 将自动重新连接（PAIR 套接字除外）。

* 你的应用程序代码不能直接处理这些连接；它们被封装在套接字下。

许多架构遵循某种客户端/服务器模型，其中服务器是最静态的组件，客户端是最动态的组件，即它们来来去去最多。有时存在寻址问题：服务器对客户端可见，但不一定反之亦然。所以通常很明显哪个节点应该执行 `zmq_bind()`（服务器）和哪个应该执行 `zmq_connect()`（客户端）。这也取决于你使用的套接字类型，对于不寻常的网络架构有一些例外。我们稍后会看套接字类型。

现在，想象我们在启动服务器*之前*启动客户端。在传统网络中，我们得到一个大红色的失败标志。但 ZeroMQ 让我们任意启动和停止片段。一旦客户端节点执行 `zmq_connect()`，连接就存在，该节点可以开始向套接字写入消息。在某个阶段（希望在消息排队太多以至于开始被丢弃或客户端阻塞之前），服务器启动，执行 `zmq_bind()`，ZeroMQ 开始传递消息。

服务器节点可以绑定到许多端点（即协议和地址的组合），并且可以使用单个套接字来执行此操作。这意味着它将接受跨不同传输的连接：

```c
zmq_bind (socket, "tcp://*:5555");
zmq_bind (socket, "tcp://*:9999");
zmq_bind (socket, "inproc://somename");
```

对于大多数传输，你不能两次绑定到同一个端点，这与例如 UDP 不同。然而，`ipc` 传输确实允许一个进程绑定到已被第一个进程使用的端点。这旨在允许进程在崩溃后恢复。

虽然 ZeroMQ 试图对哪一边绑定和哪一边连接保持中立，但存在差异。我们稍后会更详细地看到这些。结果是你通常应该将"服务器"视为拓扑的静态部分，绑定到或多或少固定的端点，将"客户端"视为来来去去并连接到这些端点的动态部分。然后，围绕这个模型设计你的应用程序。这样"正常工作"的机会要好得多。

套接字有类型。套接字类型定义套接字的语义、其路由消息进出的策略、排队等。你可以将某些类型的套接字连接在一起，例如发布者套接字和订阅者套接字。套接字在"消息模式"中一起工作。我们稍后会更详细地看这个。

以这些不同方式连接套接字的能力给了 ZeroMQ 作为消息队列系统的基本力量。在此之上有层，例如代理，我们稍后会讲到。但本质上，使用 ZeroMQ，你通过像儿童建筑玩具一样将片段插在一起来定义你的网络架构。

### 发送和接收消息 {#Sending-and-Receiving-Messages}

要发送和接收消息，你使用 `zmq_msg_send()` 和 `zmq_msg_recv()` 方法。名称是常规的，但 ZeroMQ 的 I/O 模型与经典的 TCP 模型足够不同，你需要时间来理解它。

让我们看看在处理数据时 TCP 套接字和 ZeroMQ 套接字之间的主要差异：

* ZeroMQ 套接字承载消息，如 UDP，而不是 TCP 的字节流。ZeroMQ 消息是指定长度的二进制数据。我们稍后会讲到消息；它们的设计针对性能进行了优化，所以有点复杂。

* ZeroMQ 套接字在后台线程中执行 I/O。这意味着无论你的应用程序在忙什么，消息都会到达本地输入队列并从本地输出队列发送。

* ZeroMQ 套接字根据套接字类型内置了一对多的路由行为。

`zmq_msg_send()` 方法实际上并不将消息发送到套接字连接。它将消息排队，以便 I/O 线程可以异步发送它。除了某些异常情况外，它不会阻塞。所以当 `zmq_msg_send()` 返回到你的应用程序时，消息不一定已经发送。

### 单播传输 {#Unicast-Transports}

ZeroMQ 提供一组单播传输（`inproc`、`ipc` 和 `tcp`）和多播传输（`epgm`、`pgm`）。多播是一种高级技术，我们稍后会讲到。除非你知道你的扇出比率会使一对多单播变得不可能，否则甚至不要开始使用它。

对于大多数常见情况，使用 **`tcp`**，这是一个*断开连接的 TCP* 传输。它是弹性的、可移植的，并且对大多数情况来说足够快。我们称之为断开连接的，因为 ZeroMQ 的 `tcp` 传输不需要端点在你连接到它之前就存在。客户端和服务器可以随时连接和绑定，可以来去自由，这对应用程序仍然是透明的。

进程间 `ipc` 传输是断开连接的，就像 `tcp` 一样。它有一个限制：它还不能在 Windows 上工作。按照约定，我们使用带有 ".ipc" 扩展名的端点名称，以避免与其他文件名的潜在冲突。在 UNIX 系统上，如果你使用 `ipc` 端点，你需要用适当的权限创建它们，否则它们可能无法在不同用户 ID 下运行的进程之间共享。你还必须确保所有进程都可以访问这些文件，例如通过在相同的工作目录中运行。

线程间传输 **`inproc`** 是一个连接的信号传输。它比 `tcp` 或 `ipc` 快得多。与 `tcp` 和 `ipc` 相比，这种传输有一个特定的限制：**服务器必须在任何客户端发出连接之前发出绑定**。这是 ZeroMQ 未来版本可能会修复的问题，但目前这定义了你如何使用 `inproc` 套接字。我们创建并绑定一个套接字并启动子线程，子线程创建并连接其他套接字。

### ZeroMQ 不是中性载体 {#ZeroMQ-is-Not-a-Neutral-Carrier}

ZeroMQ 新手常问的一个问题（我自己也问过）是，"我如何在 ZeroMQ 中编写 XYZ 服务器？"例如，"我如何在 ZeroMQ 中编写 HTTP 服务器？"这暗示如果我们使用普通套接字来承载 HTTP 请求和响应，我们应该能够使用 ZeroMQ 套接字做同样的事情，只是更快更好。

答案曾经是"这不是它的工作方式"。ZeroMQ 不是中性载体：它对它使用的传输协议强加了一个帧结构。这种帧结构与现有协议不兼容，后者倾向于使用自己的帧结构。例如，比较 HTTP 请求和 ZeroMQ 请求，两者都通过 TCP/IP。

HTTP 请求使用 CR-LF 作为其最简单的帧分隔符，而 ZeroMQ 使用长度指定的帧。所以你可以使用 ZeroMQ 编写类似 HTTP 的协议，例如使用请求-回复套接字模式。但它不会是 HTTP。

然而，从 v3.3 开始，ZeroMQ 有一个叫做 `ZMQ_ROUTER_RAW` 的套接字选项，让你可以在没有 ZeroMQ 帧结构的情况下读写数据。你可以使用它来读写适当的 HTTP 请求和响应。Hardeep Singh 贡献了这个更改，以便他可以从他的 ZeroMQ 应用程序连接到 Telnet 服务器。在撰写本文时，这仍然有些实验性，但它显示了 ZeroMQ 如何持续发展以解决新问题。也许下一个补丁将是你的。

### I/O 线程 {#I-O-Threads}

我们说过 ZeroMQ 在后台线程中执行 I/O。一个 I/O 线程（用于所有套接字）对于除最极端的应用程序之外的所有应用程序都是足够的。当你创建一个新上下文时，它从一个 I/O 线程开始。一般的经验法则是允许每秒每千兆字节的输入或输出数据一个 I/O 线程。要增加 I/O 线程的数量，在创建任何套接字*之前*使用 `zmq_ctx_set()` 调用：

```c
int io_threads = 4;
void *context = zmq_ctx_new ();
zmq_ctx_set (context, ZMQ_IO_THREADS, io_threads);
assert (zmq_ctx_get (context, ZMQ_IO_THREADS) == io_threads);
```

我们已经看到一个套接字可以同时处理几十个，甚至数千个连接。这对你如何编写应用程序有根本性影响。传统的网络应用程序每个远程连接有一个进程或一个线程，该进程或线程处理一个套接字。ZeroMQ 让你将整个结构折叠成一个进程，然后根据需要分解它以进行扩展。

如果你仅将 ZeroMQ 用于线程间通信（即不执行外部套接字 I/O 的多线程应用程序），你可以将 I/O 线程设置为零。虽然这不是一个重大优化，更多的是一种好奇心。

## 消息模式 {#Messaging-Patterns}

在 ZeroMQ 套接字 API 的棕色纸包装下面是消息模式的世界。如果你有企业消息传递的背景，或者很了解 UDP，这些会有些熟悉。但对大多数 ZeroMQ 新手来说，它们是一个惊喜。我们太习惯于 TCP 范式，其中套接字一对一映射到另一个节点。

让我们简要回顾一下 ZeroMQ 为你做了什么。它快速高效地将数据块（消息）传递给节点。你可以将节点映射到线程、进程或节点。ZeroMQ 为你的应用程序提供了一个单一的套接字 API，无论实际传输是什么（如进程内、进程间、TCP 或多播）。它会自动重新连接到对等体（PAIR 套接字除外），因为它们来来去去。它根据需要在发送方和接收方都排队消息。它限制这些队列以防止进程内存不足。它处理套接字错误。它在后台线程中执行所有 I/O。它使用无锁技术在节点之间通信，所以永远不会有锁、等待、信号量或死锁。

但穿透这些，它根据称为*模式*的精确配方路由和排队消息。正是这些模式提供了 ZeroMQ 的智能。它们封装了我们来之不易的分发数据和工作的最佳方式的经验。ZeroMQ 的模式是硬编码的，但未来版本可能允许用户定义的模式。

ZeroMQ 模式由具有匹配类型的套接字对实现。换句话说，要理解 ZeroMQ 模式，你需要理解套接字类型以及它们如何协同工作。大多数情况下，这只需要学习；在这个级别上很少有明显的东西。

内置的核心 ZeroMQ 模式是：

* **请求-回复**，它将一组客户端连接到一组服务。这是远程过程调用和任务分发模式。

* **发布-订阅**，它将一组发布者连接到一组订阅者。这是数据分发模式。

* **管道**，它以扇出/扇入模式连接节点，可以有多个步骤和循环。这是并行任务分发和收集模式。

* **独占对**，它专门连接两个套接字。这是连接进程中两个线程的模式，不要与套接字的"正常"对混淆。

我们在第一章中看了前三个，我们将在本章稍后看到独占对模式。`zmq_socket()` 手册页对模式相当清楚——值得多次阅读，直到它开始有意义。这些是连接-绑定对的有效套接字组合（任一侧都可以绑定）：

* PUB 和 SUB
* REQ 和 REP
* REQ 和 ROUTER（小心，REQ 插入一个额外的空帧）
* DEALER 和 REP（小心，REP 假设一个空帧）
* DEALER 和 ROUTER
* DEALER 和 DEALER
* ROUTER 和 ROUTER
* PUSH 和 PULL
* PAIR 和 PAIR

你还会看到对 XPUB 和 XSUB 套接字的引用，我们稍后会讲到（它们就像 PUB 和 SUB 的原始版本）。任何其他组合将产生未记录和不可靠的结果，ZeroMQ 的未来版本如果你尝试它们可能会返回错误。当然，你可以并且将通过代码桥接其他套接字类型，即从一种套接字类型读取并写入另一种。

### 高级消息模式 {#High-Level-Messaging-Patterns}

这四个核心模式被烘焙到 ZeroMQ 中。它们是 ZeroMQ API 的一部分，在核心 C++ 库中实现，并保证在所有优秀的零售店中都可以找到。

在这些之上，我们添加*高级消息模式*。我们在 ZeroMQ 之上构建这些高级模式，并用我们用于应用程序的任何语言实现它们。它们不是核心库的一部分，不随 ZeroMQ 包一起提供，并作为 ZeroMQ 社区的一部分存在于它们自己的空间中。例如，我们在可靠请求-回复章节中探索的 Majordomo 模式，位于 ZeroMQ 组织的 GitHub Majordomo 项目中。

我们在这本书中要为你提供的东西之一是一套这样的高级模式，包括小的（如何合理地处理消息）和大的（如何制作可靠的发布-订阅架构）。

### 处理消息 {#Working-with-Messages}

`libzmq` 核心库实际上有两个 API 来发送和接收消息。我们已经看过和使用的 `zmq_send()` 和 `zmq_recv()` 方法是简单的一行程序。我们会经常使用这些，但 `zmq_recv()` 在处理任意消息大小方面很糟糕：它将消息截断为你提供的任何缓冲区大小。所以有第二个 API，它与 `zmq_msg_t` 结构一起工作，具有更丰富但更困难的 API：

* 初始化消息：`zmq_msg_init()`、`zmq_msg_init_size()`、`zmq_msg_init_data()`。
* 发送和接收消息：`zmq_msg_send()`、`zmq_msg_recv()`。
* 释放消息：`zmq_msg_close()`。
* 访问消息内容：`zmq_msg_data()`、`zmq_msg_size()`、`zmq_msg_more()`。
* 处理消息属性：`zmq_msg_get()`、`zmq_msg_set()`。
* 消息操作：`zmq_msg_copy()`、`zmq_msg_move()`。

在线路上，ZeroMQ 消息是任何大小从零开始到内存中适合的 blob。你使用协议缓冲区、msgpack、JSON 或你的应用程序需要说话的任何其他东西进行自己的序列化。明智的做法是选择可移植的数据表示，但你可以对权衡做出自己的决定。

在内存中，ZeroMQ 消息是 `zmq_msg_t` 结构（或类，取决于你的语言）。以下是在 C 中使用 ZeroMQ 消息的基本规则：

* 你创建并传递 `zmq_msg_t` 对象，而不是数据块。

* 要读取消息，你使用 `zmq_msg_init()` 创建一个空消息，然后将其传递给 `zmq_msg_recv()`。

* 要从新数据写入消息，你使用 `zmq_msg_init_size()` 创建消息，同时分配某个大小的数据块。然后使用 `memcpy` 填充该数据，并将消息传递给 `zmq_msg_send()`。

* 要释放（不是销毁）消息，你调用 `zmq_msg_close()`。这会删除一个引用，最终 ZeroMQ 会销毁消息。

* 要访问消息内容，你使用 `zmq_msg_data()`。要知道消息包含多少数据，使用 `zmq_msg_size()`。

* 除非你阅读了手册页并确切知道为什么需要这些，否则不要使用 `zmq_msg_move()`、`zmq_msg_copy()` 或 `zmq_msg_init_data()`。

* 在你将消息传递给 `zmq_msg_send()` 后，ØMQ 将清除消息，即将大小设置为零。你不能发送同一条消息两次，并且在发送后无法访问消息数据。

* 如果你使用 `zmq_send()` 和 `zmq_recv()`，这些规则不适用，你向它们传递字节数组，而不是消息结构。

如果你想多次发送同一条消息，并且它很大，创建第二条消息，使用 `zmq_msg_init()` 初始化它，然后使用 `zmq_msg_copy()` 创建第一条消息的副本。这不会复制数据，而是复制引用。然后你可以发送消息两次（或更多，如果你创建更多副本），消息只有在最后一个副本被发送或关闭时才会最终被销毁。

ZeroMQ 还支持*多部分*消息，让你作为单个线路消息发送或接收帧列表。这在真实应用程序中广泛使用，我们将在本章稍后和高级请求-回复章节中看到这一点。

帧（在 ZeroMQ 参考手册页中也称为"消息部分"）是 ZeroMQ 消息的基本线路格式。帧是长度指定的数据块。长度可以是零或更多。如果你做过任何 TCP 编程，你会欣赏为什么帧对问题"我现在应该从这个网络套接字读取多少数据？"是一个有用的答案。

有一个叫做 ZMTP 的线路级协议，定义了 ZeroMQ 如何在 TCP 连接上读写帧。如果你对这是如何工作的感兴趣，规范相当短。

最初，ZeroMQ 消息是一个帧，就像 UDP。我们后来用多部分消息扩展了这一点，这些消息很简单就是一系列帧，"more"位设置为一，然后是一个设置为零的帧。然后 ZeroMQ API 让你用"more"标志写入消息，当你读取消息时，它让你检查是否有"more"。

因此，在低级 ZeroMQ API 和参考手册中，消息与帧之间有一些模糊性。所以这里有一个有用的词汇：

* 消息可以是一个或多个部分。
* 这些部分也称为"帧"。
* 每个部分是一个 `zmq_msg_t` 对象。
* 在低级 API 中，你分别发送和接收每个部分。
* 更高级的 API 提供包装器来发送整个多部分消息。

关于消息的其他一些值得了解的事情：

* 你可以发送零长度消息，例如用于从一个线程向另一个线程发送信号。

* ZeroMQ 保证传递消息的所有部分（一个或多个），或者都不传递。

* ZeroMQ 不会立即发送消息（单部分或多部分），而是在某个不确定的稍后时间。因此多部分消息必须适合内存。

* 消息（单部分或多部分）必须适合内存。如果你想发送任意大小的文件，你应该将它们分解成片段，并将每个片段作为单独的单部分消息发送。*使用多部分数据不会减少内存消耗。*

* 在自动销毁对象的语言中，当作用域关闭时，你必须在完成接收消息后调用 `zmq_msg_close()`。在发送消息后不要调用此方法。

再重复一遍，还不要使用 `zmq_msg_init_data()`。这是一个零复制方法，保证会给你制造麻烦。在你开始担心削减微秒之前，有更重要的 ZeroMQ 知识需要学习。

这个丰富的 API 使用起来可能很麻烦。这些方法针对性能而不是简单性进行了优化。如果你开始使用这些，在你仔细阅读手册页之前，你几乎肯定会弄错它们。所以好的语言绑定的主要工作之一是将这个 API 包装在更易于使用的类中。

### 处理多个套接字 {#Handling-Multiple-Sockets}

到目前为止，在所有示例中，大多数示例的主循环都是：

1. 在套接字上等待消息。
2. 处理消息。
3. 重复。

如果我们想同时从多个端点读取怎么办？最简单的方法是将一个套接字连接到所有端点，让 ZeroMQ 为我们做扇入。如果远程端点在同一模式中，这是合法的，但将 PULL 套接字连接到 PUB 端点是错误的。

要实际同时从多个套接字读取，使用 `zmq_poll()`。更好的方法可能是将 `zmq_poll()` 包装在一个框架中，将其转换为一个好的事件驱动*反应器*，但这比我们想在这里涵盖的工作要多得多。

让我们从一个肮脏的黑客开始，部分是为了不正确地做它的乐趣，但主要是因为它让我向你展示如何进行非阻塞套接字读取。这是一个使用非阻塞读取从两个套接字读取的简单示例。这个相当混乱的程序既充当天气更新的订阅者，又充当并行任务的工作者：

这种方法的成本是第一条消息的一些额外延迟（循环末尾的睡眠，当没有等待消息要处理时）。这在亚毫秒延迟至关重要的应用程序中会是一个问题。此外，你需要检查 `nanosleep()` 或你使用的任何函数的文档，以确保它不会忙循环。

你可以通过首先从一个读取，然后从第二个读取，而不是像我们在这个示例中那样优先考虑它们来公平地对待套接字。

现在让我们看看使用 `zmq_poll()` 正确完成的同样无意义的小应用程序：

items 结构有这四个成员：

```c
typedef struct {
    void *socket;       //  要轮询的 ZeroMQ 套接字
    int fd;             //  或者，要轮询的本机文件句柄
    short events;       //  要轮询的事件
    short revents;      //  轮询后返回的事件
} zmq_pollitem_t;
```

### 多部分消息 {#Multipart-Messages}

ZeroMQ 让我们将一个消息组合成几个帧，给我们一个"多部分消息"。现实应用程序大量使用多部分消息，既用于用地址信息包装消息，也用于简单的序列化。我们稍后会看到回复信封。

我们现在要学习的只是如何在任何需要转发消息而不检查它们的应用程序（如代理）中盲目且安全地读写多部分消息。

当你处理多部分消息时，每个部分都是一个 `zmq_msg` 项。例如，如果你发送一个五部分的消息，你必须构造、发送和销毁五个 `zmq_msg` 项。你可以提前做这件事（并将 `zmq_msg` 项存储在数组或其他结构中），或者在发送时逐个发送。

以下是我们如何在多部分消息中发送帧（我们将每个帧接收到一个消息对象中）：

```c
zmq_msg_send (&message, socket, ZMQ_SNDMORE);
...
zmq_msg_send (&message, socket, ZMQ_SNDMORE);
...
zmq_msg_send (&message, socket, 0);
```

以下是我们如何接收和处理消息中的所有部分，无论是单部分还是多部分：

```c
while (1) {
    zmq_msg_t message;
    zmq_msg_init (&message);
    zmq_msg_recv (&message, socket, 0);
    //  处理消息帧
    ...
    zmq_msg_close (&message);
    if (!zmq_msg_more (&message))
        break;      //  最后一个消息帧
}
```

关于多部分消息需要了解的一些事情：

* 当你发送多部分消息时，第一部分（和所有后续部分）只有在你发送最后一部分时才会真正在线路上发送。
* 如果你使用 `zmq_poll()`，当你接收到消息的第一部分时，其余部分也已经到达。
* 你将接收消息的所有部分，或者一个都不接收。
* 消息的每个部分都是一个单独的 `zmq_msg` 项。
* 无论你是否检查 more 属性，你都将接收消息的所有部分。
* 在发送时，ZeroMQ 在内存中排队消息帧，直到接收到最后一个，然后全部发送。
* 除了关闭套接字外，没有办法取消部分发送的消息。

### 中介和代理 {#Intermediaries-and-Proxies}

ZeroMQ 旨在实现分散智能，但这并不意味着你的网络中间是空的。它充满了消息感知的基础设施，我们经常用 ZeroMQ 构建这种基础设施。ZeroMQ 管道可以从微小的管道到成熟的面向服务的代理。消息传递行业称此为*中介*，意思是中间的东西处理双方。在 ZeroMQ 中，根据上下文，我们称这些为代理、队列、转发器、设备或代理。

这种模式在现实世界中极其常见，这就是为什么我们的社会和经济充满了中介，他们除了降低更大网络的复杂性和扩展成本外没有其他真正功能。现实世界的中介通常被称为批发商、分销商、经理等。

### 动态发现问题 {#The-Dynamic-Discovery-Problem}

当你设计更大的分布式架构时，你会遇到的问题之一是发现。也就是说，各部分如何了解彼此？如果各部分来来去去，这尤其困难，所以我们称之为"动态发现问题"。

动态发现有几种解决方案。最简单的是通过硬编码（或配置）网络架构来完全避免它，以便手动完成发现。也就是说，当你添加新部分时，你重新配置网络以了解它。

在实践中，这导致越来越脆弱和笨拙的架构。假设你有一个发布者和一百个订阅者。你通过在每个订阅者中配置发布者端点来将每个订阅者连接到发布者。这很容易。订阅者是动态的；发布者是静态的。现在假设你添加更多发布者。突然，它不再那么容易了。如果你继续将每个订阅者连接到每个发布者，避免动态发现的成本会越来越高。

有很多答案，但最简单的答案是添加一个中介；即网络中的一个静态点，所有其他节点都连接到它。在经典消息传递中，这是消息代理的工作。ZeroMQ 本身没有消息代理，但它让我们很容易构建中介。

你可能想知道，如果所有网络最终都变得足够大以需要中介，为什么我们不为所有应用程序简单地放置一个消息代理？对于初学者来说，这是一个公平的妥协。只需始终使用星形拓扑，忘记性能，事情通常会工作。然而，消息代理是贪婪的东西；在他们作为中心中介的角色中，他们变得太复杂、太有状态，最终成为问题。

最好将中介视为简单的无状态消息交换机。一个好的类比是 HTTP 代理；它在那里，但没有任何特殊作用。添加发布-订阅代理解决了我们示例中的动态发现问题。我们将代理设置在网络的"中间"。代理打开一个 XSUB 套接字、一个 XPUB 套接字，并将每个绑定到众所周知的 IP 地址和端口。然后，所有其他进程连接到代理，而不是彼此连接。添加更多订阅者或发布者变得微不足道。

我们需要 XPUB 和 XSUB 套接字，因为 ZeroMQ 从订阅者到发布者进行订阅转发。XSUB 和 XPUB 与 SUB 和 PUB 完全相同，除了它们将订阅公开为特殊消息。代理必须将这些订阅消息从订阅者端转发到发布者端，通过从 XPUB 套接字读取它们并将它们写入 XSUB 套接字。这是 XSUB 和 XPUB 的主要用例。

### 共享队列（DEALER 和 ROUTER 套接字）{#Shared-Queue-DEALER-and-ROUTER-sockets}

在 Hello World 客户端/服务器应用程序中，我们有一个与一个服务对话的客户端。但是，在实际情况下，我们通常需要允许多个服务以及多个客户端。这让我们可以扩展服务的功能（许多线程或进程或节点而不是只有一个）。唯一的约束是服务必须是无状态的，所有状态都在请求中或在某些共享存储（如数据库）中。

有两种方法将多个客户端连接到多个服务器。蛮力方法是将每个客户端套接字连接到多个服务端点。一个客户端套接字可以连接到多个服务套接字，然后 REQ 套接字将在这些服务之间分发请求。假设你将客户端套接字连接到三个服务端点：A、B 和 C。客户端发出请求 R1、R2、R3、R4。R1 和 R4 去服务 A，R2 去 B，R3 去服务 C。

这种设计让你便宜地添加更多客户端。你也可以添加更多服务。每个客户端将其请求分发到服务。但每个客户端都必须知道服务拓扑。如果你有 100 个客户端，然后你决定添加三个更多的服务，你需要重新配置和重启 100 个客户端，以便客户端知道三个新服务。

这显然不是我们想在凌晨 3 点做的事情，当我们的超级计算集群资源不足，我们迫切需要添加几百个新服务节点时。太多的静态部分就像液体混凝土：知识是分布式的，你拥有的静态部分越多，改变拓扑的努力就越大。我们想要的是坐在客户端和服务之间的东西，集中所有拓扑知识。理想情况下，我们应该能够随时添加和删除服务或客户端，而不触及拓扑的任何其他部分。

所以我们将编写一个小的消息排队代理，给我们这种灵活性。代理绑定到两个端点，客户端的前端和服务的后端。然后它使用 `zmq_poll()` 来监控这两个套接字的活动，当它有一些活动时，它在其两个套接字之间传输消息。它实际上并不明确管理任何队列——ZeroMQ 在每个套接字上自动执行此操作。

当你使用 REQ 与 REP 对话时，你得到一个严格同步的请求-回复对话。客户端发送请求。服务读取请求并发送回复。然后客户端读取回复。如果客户端或服务尝试做其他任何事情（例如，连续发送两个请求而不等待响应），他们将得到错误。

但我们的代理必须是非阻塞的。显然，我们可以使用 `zmq_poll()` 等待任一套接字上的活动，但我们不能使用 REP 和 REQ。

幸运的是，有两个名为 DEALER 和 ROUTER 的套接字让你可以进行非阻塞请求-响应。你将在高级请求-回复章节中看到 DEALER 和 ROUTER 套接字如何让你构建各种异步请求-回复流。现在，我们只是要看看 DEALER 和 ROUTER 如何让我们通过中介（即我们的小代理）扩展 REQ-REP。

在这个简单的扩展请求-回复模式中，REQ 与 ROUTER 对话，DEALER 与 REP 对话。在 DEALER 和 ROUTER 之间，我们必须有代码（如我们的代理）从一个套接字拉取消息并将它们推送到另一个套接字。

请求-回复代理绑定到两个端点，一个供客户端连接（前端套接字），一个供工作者连接（后端）。要测试这个代理，你需要更改你的工作者，使它们连接到后端套接字。以下是一个显示我的意思的客户端：

```c
//  请求-回复客户端
//  连接 REQ 套接字到 tcp://localhost:5559
//  发送 "Hello" 到服务器，期待回复 "World"
#include "zhelpers.h"

int main (void)
{
    void *context = zmq_ctx_new ();
    void *requester = zmq_socket (context, ZMQ_REQ);
    zmq_connect (requester, "tcp://localhost:5559");

    int request_nbr;
    for (request_nbr = 0; request_nbr != 10; request_nbr++) {
        char buffer [10];
        printf ("发送 Hello %d...\n", request_nbr);
        zmq_send (requester, "Hello", 5, 0);
        zmq_recv (requester, buffer, 10, 0);
        printf ("接收到 World %d\n", request_nbr);
    }
    zmq_close (requester);
    zmq_ctx_destroy (context);
    return 0;
}
```

以下是工作者：

```c
//  请求-回复工作者
//  连接 REP 套接字到 tcp://localhost:5560
//  期待 "Hello" 从客户端，回复 "World"
#include "zhelpers.h"

int main (void)
{
    void *context = zmq_ctx_new ();
    void *responder = zmq_socket (context, ZMQ_REP);
    zmq_connect (responder, "tcp://localhost:5560");

    while (1) {
        char buffer [10];
        zmq_recv (responder, buffer, 10, 0);
        printf ("接收到 Hello\n");
        sleep (1);          //  做一些 '工作'
        zmq_send (responder, "World", 5, 0);
    }
    zmq_close (responder);
    zmq_ctx_destroy (context);
    return 0;
}
```

以下是代理，它正确处理多部分消息：

```c
//  简单请求-回复代理
#include "zhelpers.h"

int main (void)
{
    //  准备我们的上下文和套接字
    void *context = zmq_ctx_new ();
    void *frontend = zmq_socket (context, ZMQ_ROUTER);
    void *backend  = zmq_socket (context, ZMQ_DEALER);
    zmq_bind (frontend, "tcp://*:5559");
    zmq_bind (backend,  "tcp://*:5560");

    //  初始化轮询集
    zmq_pollitem_t items [] = {
        { frontend, 0, ZMQ_POLLIN, 0 },
        { backend,  0, ZMQ_POLLIN, 0 }
    };
    //  在套接字之间切换消息
    while (1) {
        zmq_msg_t message;
        int more;               //  多部分检测

        zmq_poll (items, 2, -1);
        if (items [0].revents & ZMQ_POLLIN) {
            while (1) {
                //  处理来自前端的所有消息部分
                zmq_msg_init (&message);
                zmq_msg_recv (&message, frontend, 0);
                more = zmq_msg_more (&message);
                zmq_msg_send (&message, backend, more? ZMQ_SNDMORE: 0);
                zmq_msg_close (&message);
                if (!more)
                    break;      //  最后一个消息部分
            }
        }
        if (items [1].revents & ZMQ_POLLIN) {
            while (1) {
                //  处理来自后端的所有消息部分
                zmq_msg_init (&message);
                zmq_msg_recv (&message, backend, 0);
                more = zmq_msg_more (&message);
                zmq_msg_send (&message, frontend, more? ZMQ_SNDMORE: 0);
                zmq_msg_close (&message);
                if (!more)
                    break;      //  最后一个消息部分
            }
        }
    }
    //  我们永远不会到达这里，但如果我们这样做了，这是很好的做法：
    zmq_close (frontend);
    zmq_close (backend);
    zmq_ctx_destroy (context);
    return 0;
}
```

使用请求-回复代理使你的客户端/服务器架构更容易扩展，因为客户端看不到工作者，工作者也看不到客户端。唯一的静态节点是中间的代理。

### ZeroMQ 的内置代理函数 {#ZeroMQ-Built-In-Proxy-Function}

事实证明，前一节中 `rrbroker` 的核心循环非常有用且可重用。它让我们可以非常轻松地构建发布-订阅转发器和共享队列以及其他小型中介。ZeroMQ 将此包装在一个方法中，`zmq_proxy()`：

```c
zmq_proxy (frontend, backend, capture);
```

两个（或三个套接字，如果我们想捕获数据）必须正确连接、绑定和配置。当我们调用 `zmq_proxy()` 方法时，它就像启动 `rrbroker` 的主循环一样。让我们重写请求-回复代理来调用 `zmq_proxy()`，并将其重新标记为听起来昂贵的"消息队列"（人们为做得更少的代码收取房价）：

```c
//  消息队列代理
//  与请求-回复代理相同，但使用代理队列设备
#include "zhelpers.h"

int main (void)
{
    void *context = zmq_ctx_new ();

    //  面向客户端的套接字
    void *frontend = zmq_socket (context, ZMQ_ROUTER);
    int rc = zmq_bind (frontend, "tcp://*:5559");
    assert (rc == 0);

    //  面向服务的套接字
    void *backend = zmq_socket (context, ZMQ_DEALER);
    rc = zmq_bind (backend, "tcp://*:5560");
    assert (rc == 0);

    //  启动代理，直到用户中断程序
    //  由于不会有任何控制流返回，我们不会执行任何清理
    zmq_proxy (frontend, backend, NULL);

    //  从未执行
    zmq_close (frontend);
    zmq_close (backend);
    zmq_ctx_destroy (context);
    return 0;
}
```

如果你像大多数 ZeroMQ 用户一样，在这个阶段你的大脑开始思考，"如果我将随机套接字类型插入代理，我能做什么邪恶的事情？"简短的答案是：试试并弄清楚发生了什么。在实践中，你通常会坚持使用 ROUTER/DEALER、XSUB/XPUB 或 PULL/PUSH。

### 传输桥接 {#Transport-Bridging}

ZeroMQ 用户的一个常见请求是，"我如何将我的 ZeroMQ 网络与技术 X 连接？"其中 X 是一些其他网络或消息传递技术。

简单的答案是构建一个*桥*。桥是一个小应用程序，在一个套接字上说一种协议，并在另一个套接字上转换到/从第二种协议。如果你愿意，可以说是协议解释器。ZeroMQ 中的常见桥接问题是桥接两个传输或网络。

作为示例，我们将编写一个小代理，它位于发布者和一组订阅者之间，桥接两个网络。前端套接字（XSUB）面向天气服务器所在的内部网络，后端（XPUB）面向外部网络上的订阅者。它在前端套接字上订阅天气服务，并在后端套接字上重新发布其数据。

```c
//  天气更新代理
//  将 PUB-SUB 从一个网络桥接到另一个网络
#include "zhelpers.h"

int main (void)
{
    void *context = zmq_ctx_new ();

    //  这是我们接收消息的地方
    void *frontend = zmq_socket (context, ZMQ_XSUB);
    int rc = zmq_connect (frontend, "tcp://192.168.1.210:5556");
    assert (rc == 0);

    //  这是我们发送消息的地方
    void *backend = zmq_socket (context, ZMQ_XPUB);
    rc = zmq_bind (backend, "tcp://*:8100");
    assert (rc == 0);

    //  运行代理，直到用户中断程序
    zmq_proxy (frontend, backend, NULL);

    zmq_close (frontend);
    zmq_close (backend);
    zmq_ctx_destroy (context);
    return 0;
}
```

它看起来与之前的代理示例非常相似，但关键部分是前端和后端套接字在两个不同的网络上。我们可以使用这个模型，例如，将多播网络（`pgm` 传输）连接到 `tcp` 发布者。

## 处理错误和 ETERM {#Handling-Errors-and-ETERM}

ZeroMQ 的错误处理哲学是快速失败和恢复能力的混合。我们认为，进程应该尽可能容易受到内部错误的影响，并尽可能对外部攻击和错误具有鲁棒性。打个比方，活细胞在检测到单个内部错误时会自毁，但它会尽一切可能抵制来自外部的攻击。

断言，它们在 ZeroMQ 代码中随处可见，对鲁棒代码绝对重要；它们只是必须在细胞壁的正确一侧。应该有这样的壁。如果不清楚故障是内部的还是外部的，那是要修复的设计缺陷。在 C/C++ 中，断言立即停止应用程序并显示错误。在其他语言中，你可能会得到异常或停止。

当 ZeroMQ 检测到外部故障时，它向调用代码返回错误。在一些罕见情况下，如果没有明显的错误恢复策略，它会默默地丢弃消息。

在我们到目前为止看到的大多数 C 示例中，没有错误处理。**真正的代码应该对每个 ZeroMQ 调用进行错误处理**。如果你使用 C 以外的语言绑定，绑定可能会为你处理错误。在 C 中，你确实需要自己做这件事。有一些简单的规则，从 POSIX 约定开始：

* 创建对象的方法在失败时返回 NULL。
* 处理数据的方法可能返回处理的字节数，或在错误或失败时返回 -1。
* 其他方法在成功时返回 0，在错误或失败时返回 -1。
* 错误代码在 `errno` 或 `zmq_errno()` 中提供。
* 用于记录的描述性错误文本由 `zmq_strerror()` 提供。

例如：

```c
void *context = zmq_ctx_new ();
assert (context);
void *socket = zmq_socket (context, ZMQ_REP);
assert (socket);
int rc = zmq_bind (socket, "tcp://*:5555");
if (rc == -1) {
    printf ("E: bind failed: %s\n", strerror (errno));
    return -1;
}
```

你应该处理为非致命的两个主要异常情况：

* 当你的代码使用 `ZMQ_DONTWAIT` 选项接收消息且没有等待数据时，ZeroMQ 将返回 -1 并将 `errno` 设置为 `EAGAIN`。

* 当一个线程调用 `zmq_ctx_destroy()`，而其他线程仍在进行阻塞工作时，`zmq_ctx_destroy()` 调用关闭上下文，所有阻塞调用以 -1 退出，`errno` 设置为 `ETERM`。

在 C/C++ 中，断言可以在优化代码中完全删除，所以不要犯将整个 ZeroMQ 调用包装在 `assert()` 中的错误。它看起来整洁；然后优化器删除所有断言和你想要进行的调用，你的应用程序以令人印象深刻的方式中断。

让我们看看如何干净地关闭进程。我们将采用上一节的并行管道示例。如果我们在后台启动了很多工作者，我们现在想在批处理完成时杀死它们。让我们通过向工作者发送杀死消息来做到这一点。最好的地方是汇聚器，因为它真正知道何时批处理完成。

我们如何将汇聚器连接到工作者？PUSH/PULL 套接字是单向的。我们可以切换到另一种套接字类型，或者我们可以混合多个套接字流。让我们尝试后者：使用发布-订阅模型向工作者发送杀死消息：

* 汇聚器在新端点上创建 PUB 套接字。
* 工作者将其输入套接字连接到此端点。
* 当汇聚器检测到批处理结束时，它向其 PUB 套接字发送杀死消息。
* 当工作者检测到此杀死消息时，它退出。

汇聚器中不需要太多新代码：

```c
void *controller = zmq_socket (context, ZMQ_PUB);
zmq_bind (controller, "tcp://*:5559");
...
//  向工作者发送杀死信号
s_send (controller, "KILL");
```

以下是工作进程，它管理两个套接字（一个获取任务的 PULL 套接字和一个获取控制命令的 SUB 套接字），使用我们之前看到的 `zmq_poll()` 技术：

```c
//  带有杀死信号的任务工作者
//  连接 PULL 套接字到 tcp://localhost:5557
//  收集来自通风器的工作负载
//  连接 PUSH 套接字到 tcp://localhost:5558
//  向汇聚器发送结果
//  连接 SUB 套接字到 tcp://localhost:5559
//  收集来自汇聚器的控制命令

#include "zhelpers.h"

int main (void)
{
    void *context = zmq_ctx_new ();

    //  接收工作的套接字
    void *receiver = zmq_socket (context, ZMQ_PULL);
    zmq_connect (receiver, "tcp://localhost:5557");

    //  发送消息到汇聚器的套接字
    void *sender = zmq_socket (context, ZMQ_PUSH);
    zmq_connect (sender, "tcp://localhost:5558");

    //  接收控制消息的套接字
    void *controller = zmq_socket (context, ZMQ_SUB);
    zmq_connect (controller, "tcp://localhost:5559");
    zmq_setsockopt (controller, ZMQ_SUBSCRIBE, "", 0);

    //  处理来自两个套接字的消息
    zmq_pollitem_t items [] = {
        { receiver, 0, ZMQ_POLLIN, 0 },
        { controller, 0, ZMQ_POLLIN, 0 }
    };

    //  处理消息直到收到 KILL
    while (1) {
        zmq_poll (items, 2, -1);
        if (items [0].revents & ZMQ_POLLIN) {
            char *string = s_recv (receiver);
            free (string);

            //  做工作
            s_sleep (atoi (string));

            //  发送结果到汇聚器
            s_send (sender, "");
        }
        //  任何等待的控制器命令都作为中断
        if (items [1].revents & ZMQ_POLLIN) {
            char *string = s_recv (controller);
            if (strcmp (string, "KILL") == 0) {
                free (string);
                break;              //  退出循环
            }
            free (string);
        }
    }
    zmq_close (receiver);
    zmq_close (sender);
    zmq_close (controller);
    zmq_ctx_destroy (context);
    return 0;
}
```

以下是修改后的汇聚器应用程序。当它完成收集结果时，它向所有工作者广播杀死消息：

```c
//  带有杀死信号的任务汇聚器
//  绑定 PULL 套接字到 tcp://localhost:5558
//  收集来自工作者的结果并计时
//  绑定 PUB 套接字到 tcp://localhost:5559
//  向工作者发送杀死信号

#include "zhelpers.h"

int main (void)
{
    void *context = zmq_ctx_new ();

    //  接收消息的套接字
    void *receiver = zmq_socket (context, ZMQ_PULL);
    zmq_bind (receiver, "tcp://*:5558");

    //  向工作者发送杀死信号的套接字
    void *controller = zmq_socket (context, ZMQ_PUB);
    zmq_bind (controller, "tcp://*:5559");

    //  等待开始批处理
    char *string = s_recv (receiver);
    free (string);

    //  启动我们的时钟现在
    int64_t start_time = s_clock ();

    //  处理 100 个确认
    int task_nbr;
    for (task_nbr = 0; task_nbr < 100; task_nbr++) {
        char *string = s_recv (receiver);
        free (string);
        if ((task_nbr / 10) * 10 == task_nbr)
            printf (":");
        else
            printf (".");
        fflush (stdout);
    }
    //  计算并报告执行时间的持续时间
    printf ("总执行时间：%d 毫秒\n",
        (int) (s_clock () - start_time));

    //  向工作者发送杀死信号
    s_send (controller, "KILL");

    //  完成关闭的房屋清洁
    s_sleep (1);              //  给 0MQ 时间传递

    zmq_close (receiver);
    zmq_close (controller);
    zmq_ctx_destroy (context);
    return 0;
}
```

## 处理中断信号 {#Handling-Interrupt-Signals}

现实应用程序需要在使用 Ctrl-C 或其他信号（如 `SIGTERM`）中断时干净地关闭。默认情况下，这些只是杀死进程，意味着消息不会被刷新，文件不会干净地关闭，等等。

以下是我们如何在各种语言中处理信号：

```c
//  展示了如何处理 Ctrl-C
#include "zhelpers.h"

static int s_interrupted = 0;
static void s_signal_handler (int signal_value)
{
    s_interrupted = 1;
}

static void s_catch_signals (void)
{
    struct sigaction action;
    action.sa_handler = s_signal_handler;
    action.sa_flags = 0;
    sigemptyset (&action.sa_mask);
    sigaction (SIGINT, &action, NULL);
    sigaction (SIGTERM, &action, NULL);
}

int main (void)
{
    void *context = zmq_ctx_new ();
    void *socket = zmq_socket (context, ZMQ_REP);
    zmq_bind (socket, "tcp://*:5555");

    s_catch_signals ();
    while (1) {
        //  阻塞直到收到消息
        zmq_msg_t message;
        zmq_msg_init (&message);
        int rc = zmq_msg_recv (&message, socket, 0);
        if (rc == -1 && errno == EINTR) {
            printf ("W: 接收时中断\n");
            break;
        }
        assert (rc == 0);
        zmq_msg_close (&message);

        if (s_interrupted) {
            printf ("W: 中断接收到，正在杀死服务器...\n");
            break;
        }
    }
    zmq_close (socket);
    zmq_ctx_destroy (context);
    return 0;
}
```

程序提供 `s_catch_signals()`，它捕获 Ctrl-C（`SIGINT`）和 `SIGTERM`。当任一信号到达时，`s_signal_handler()` 处理程序将一个字节写入主函数启动代码中创建的自管道。多亏了你的信号处理程序，你的应用程序不会自动死亡。相反，你有机会清理并优雅地退出。你现在必须明确检查中断并正确处理它。通过检查自管道是否包含任何数据（从 `interrupt.c` 复制）在你代码的主循环中来做到这一点。中断将影响 ZeroMQ 调用如下：

* 如果你的代码在阻塞调用中阻塞（发送消息、接收消息或轮询），那么当信号到达时，调用将返回 `EINTR`。
* 像 `s_recv()` 这样的包装器在被中断时返回 NULL。

所以检查 `EINTR` 返回代码、NULL 返回和/或自管道的队列中是否有任何数据。

如果你调用 `s_catch_signals()` 而不测试自管道的中断，那么你的应用程序将对 Ctrl-C 和 `SIGTERM` 免疫，这可能有用，但通常不是。

## 检测内存泄漏 {#Detecting-Memory-Leaks}

任何长时间运行的应用程序都必须正确管理内存，否则最终会用完所有可用内存并崩溃。如果你使用自动为你处理这个的语言，恭喜。如果你用 C 或 C++ 或任何其他你负责内存管理的语言编程，这里有一个关于使用 valgrind 的简短教程，它除了其他事情外还会报告你的程序中的任何泄漏。

* 要安装 valgrind，例如，在 Ubuntu 或 Debian 上，发出此命令：

```bash
sudo apt-get install valgrind
```

* 默认情况下，ZeroMQ 会导致 valgrind 大量抱怨。要删除这些警告，创建一个名为 `vg.supp` 的文件，包含：

```
{
   <socketcall_sendto>
   Memcheck:Param
   socketcall.sendto(msg)
   fun:send
   ...
}
{
   <socketcall_sendto>
   Memcheck:Param
   socketcall.send(msg)
   fun:send
   ...
}
```

* 修复你的应用程序在 Ctrl-C 后干净退出。对于任何自己退出的应用程序，这不是必需的，但对于长时间运行的应用程序，这是必不可少的，否则 valgrind 会抱怨所有当前分配的内存。

* 如果 `-DDEBUG` 不是你的默认设置，使用它构建你的应用程序。这确保 valgrind 可以准确告诉你内存在哪里泄漏。

* 最后，这样运行 valgrind：

```bash
valgrind --tool=memcheck --leak-check=full --suppressions=vg.supp someprog
```

在修复它报告的任何错误后，你应该得到令人愉快的消息：

```
==30536== ERROR SUMMARY: 0 errors from 0 contexts...
```

## 使用 ZeroMQ 进行多线程 {#Multithreading-with-ZeroMQ}

ZeroMQ 可能是编写多线程（MT）应用程序的最好方式。虽然如果你习惯了传统套接字，ZeroMQ 套接字需要一些重新调整，但 ZeroMQ 多线程将采用你所知道的关于编写 MT 应用程序的一切，将其扔到花园的一堆中，在上面倒汽油，然后点燃它。很少有书值得燃烧，但大多数关于并发编程的书都值得。

要制作完全完美的 MT 程序（我是字面意思），**我们不需要互斥锁、锁或任何其他形式的线程间通信，除了通过 ZeroMQ 套接字发送的消息**。

通过"完美的 MT 程序"，我指的是易于编写和理解的代码，它在任何编程语言中使用相同的设计方法工作，在任何操作系统上，并且在任意数量的 CPU 上扩展，零等待状态且没有收益递减点。

如果你花了多年学习让你的 MT 代码工作的技巧，更不用说快速，使用锁和信号量和临界区，当你意识到这一切都是白费时，你会感到厌恶。如果我们从 30 多年的并发编程中学到了什么，那就是：*只是不要共享状态*。这就像两个醉汉试图分享一瓶啤酒。不管他们是不是好朋友都无关紧要。迟早，他们会打起来。你向桌子添加的醉汉越多，他们越是为了啤酒而互相争斗。绝大多数 MT 应用程序看起来像醉酒的酒吧斗殴。

当你编写经典的共享状态 MT 代码时需要对抗的奇怪问题列表如果不直接转化为压力和风险会很滑稽，因为看似正常工作的代码在压力下突然失败。一家在错误代码方面拥有世界一流经验的大公司发布了其"多线程代码中的 11 个可能问题"列表，涵盖了被遗忘的同步、不正确的粒度、读写撕裂、无锁重排序、锁护送、两步舞和优先级反转。

是的，我们数了七个问题，不是十一个。但这不是重点。重点是，你真的希望运行电网或股票市场的代码在繁忙的周四下午 3 点开始出现两步锁护送吗？谁在乎这些术语实际意味着什么？这不是让我们对编程感兴趣的东西，用越来越复杂的黑客对抗越来越复杂的副作用。

一些广泛使用的模型，尽管是整个行业的基础，但从根本上是有问题的，共享状态并发就是其中之一。想要无限扩展的代码就像互联网那样做，通过发送消息和共享除了对破碎编程模型的共同蔑视外什么都不共享。

你应该遵循一些规则来用 ZeroMQ 编写快乐的多线程代码：

* 将数据私有隔离在其线程内，永远不要在多个线程中共享数据。唯一的例外是 ZeroMQ 上下文，它们是线程安全的。

* 远离经典的并发机制，如互斥锁、临界区、信号量等。这些在 ZeroMQ 应用程序中是反模式。

* 在进程开始时创建一个 ZeroMQ 上下文，并将其传递给你想要通过 `inproc` 套接字连接的所有线程。

* 使用*附加*线程在应用程序内创建结构，并使用 PAIR 套接字通过 `inproc` 将这些连接到其父线程。模式是：绑定父套接字，然后创建连接其套接字的子线程。

* 使用*分离*线程模拟独立任务，使用它们自己的上下文。通过 `tcp` 连接这些。稍后你可以将这些移动到独立进程而不显著更改代码。

* 线程之间的所有交互都作为 ZeroMQ 消息发生，你可以或多或少正式地定义这些消息。

* 不要在线程之间共享 ZeroMQ 套接字。ZeroMQ 套接字不是线程安全的。技术上可以将套接字从一个线程迁移到另一个线程，但这需要技能。唯一远程合理的在线程之间共享套接字的地方是在需要在套接字上进行垃圾收集等魔法的语言绑定中。

如果你需要在应用程序中启动多个代理，例如，你会想要在各自的线程中运行每个代理。很容易犯在一个线程中创建代理前端和后端套接字，然后将套接字传递给另一个线程中的代理的错误。这起初可能看起来工作，但在实际使用中会随机失败。记住：*除了在创建套接字的线程中，不要使用或关闭套接字*。

如果你遵循这些规则，你可以很容易地构建优雅的多线程应用程序，并根据需要稍后将线程分离到单独的进程中。应用程序逻辑可以位于线程、进程或节点中：无论你的规模需要什么。

ZeroMQ 使用本机 OS 线程而不是虚拟"绿色"线程。优点是你不需要学习任何新的线程 API，ZeroMQ 线程干净地映射到你的操作系统。你可以使用标准工具，如 Intel 的 ThreadChecker 来查看你的应用程序在做什么。缺点是本机线程 API 并不总是可移植的，如果你有大量线程（数千个），一些操作系统会感到压力。

让我们看看这在实践中是如何工作的。我们将把我们的旧 Hello World 服务器变成更有能力的东西。原始服务器在单个线程中运行。如果每个请求的工作很少，那很好：一个 ØMQ 线程可以在 CPU 核心上全速运行，没有等待，做大量工作。但现实的服务器必须为每个请求做重要的工作。当 10,000 个客户端同时击中服务器时，单个核心可能不够。所以现实的服务器会启动多个工作线程。然后它尽可能快地接受请求并将这些分发给其工作线程。工作线程磨练工作并最终将其回复发送回。

当然，你可以使用代理和外部工作进程来完成所有这些，但启动一个吞掉十六个核心的进程通常比十六个进程每个吞掉一个核心更容易。此外，将工作者作为线程运行将减少网络跳跃、延迟和网络流量。

Hello World 服务的 MT 版本基本上将代理和工作者折叠到单个进程中：

```c
//  多线程 Hello World 服务器
#include "zhelpers.h"
#include <pthread.h>

static void *
worker_routine (void *context) {
    //  套接字与工作者通信
    void *responder = zmq_socket (context, ZMQ_REP);
    zmq_connect (responder, "inproc://workers");

    while (1) {
        char *string = s_recv (responder);
        printf ("接收到请求：[%s]\n", string);
        free (string);
        //  做一些 '工作'
        sleep (1);
        //  发送回复给客户端
        s_send (responder, "World");
    }
    zmq_close (responder);
    return NULL;
}

int main (void)
{
    void *context = zmq_ctx_new ();

    //  套接字与客户端通信
    void *clients = zmq_socket (context, ZMQ_ROUTER);
    zmq_bind (clients, "tcp://*:5555");

    //  套接字与工作者通信
    void *workers = zmq_socket (context, ZMQ_DEALER);
    zmq_bind (workers, "inproc://workers");

    //  启动工作者线程池
    int thread_nbr;
    for (thread_nbr = 0; thread_nbr < 5; thread_nbr++) {
        pthread_t worker;
        pthread_create (&worker, NULL, worker_routine, context);
    }
    //  连接工作队列到客户端套接字
    zmq_proxy (clients, workers, NULL);

    //  我们永远不会到达这里，但干净对练习有好处
    zmq_close (clients);
    zmq_close (workers);
    zmq_ctx_destroy (context);
    return 0;
}
```

所有代码现在对你来说应该是可识别的。它如何工作：

* 服务器启动一组工作线程。每个工作线程创建一个 REP 套接字，然后在此套接字上处理请求。工作线程就像单线程服务器。唯一的区别是传输（`inproc` 而不是 `tcp`）和绑定-连接方向。

* 服务器创建一个 ROUTER 套接字与客户端对话，并将其绑定到其外部接口（通过 `tcp`）。

* 服务器创建一个 DEALER 套接字与工作者对话，并将其绑定到其内部接口（通过 `inproc`）。

* 服务器启动一个连接两个套接字的代理。代理公平地从所有客户端拉取传入请求，并将这些分发给工作者。它还将回复路由回其原点。

注意创建线程在大多数编程语言中不是可移植的。POSIX 库是 pthreads，但在 Windows 上你必须使用不同的 API。在我们的示例中，`pthread_create` 调用启动一个运行我们定义的 `worker_routine` 函数的新线程。我们将在高级请求-回复章节中看到如何将其包装在可移植 API 中。

这里"工作"只是一秒钟的暂停。我们可以在工作者中做任何事情，包括与其他节点对话。这是 MT 服务器在 ØMQ 套接字和节点方面的样子。注意请求-回复链是 `REQ-ROUTER-queue-DEALER-REP`。

## 线程之间的信号（PAIR 套接字）{#Signaling-Between-Threads-PAIR-Sockets}

当你开始使用 ZeroMQ 制作多线程应用程序时，你会遇到如何协调线程的问题。虽然你可能会想插入"sleep"语句，或使用多线程技术如信号量或互斥锁，**你应该使用的唯一机制是 ZeroMQ 消息**。记住醉汉和啤酒瓶的故事。

让我们制作三个线程，当它们准备好时相互发信号。在这个示例中，我们在 `inproc` 传输上使用 PAIR 套接字：

```c
//  多线程中继
#include "zhelpers.h"
#include <pthread.h>

static void *
step1 (void *context) {
    //  连接到步骤 2 并告诉它我们准备好了
    void *xmitter = zmq_socket (context, ZMQ_PAIR);
    zmq_connect (xmitter, "inproc://step2");
    printf ("步骤 1 准备好了，发信号给步骤 2\n");
    s_send (xmitter, "READY");
    zmq_close (xmitter);
    return NULL;
}

static void *
step2 (void *context) {
    //  绑定 inproc 套接字用于步骤 1
    void *receiver = zmq_socket (context, ZMQ_PAIR);
    zmq_bind (receiver, "inproc://step2");
    pthread_t thread;
    pthread_create (&thread, NULL, step1, context);

    //  等待信号并传递给步骤 3
    char *string = s_recv (receiver);
    free (string);
    zmq_close (receiver);

    //  连接到步骤 3 并告诉它我们准备好了
    void *xmitter = zmq_socket (context, ZMQ_PAIR);
    zmq_connect (xmitter, "inproc://step3");
    printf ("步骤 2 准备好了，发信号给步骤 3\n");
    s_send (xmitter, "READY");
    zmq_close (xmitter);
    return NULL;
}

int main (void)
{
    void *context = zmq_ctx_new ();

    //  绑定 inproc 套接字用于步骤 2
    void *receiver = zmq_socket (context, ZMQ_PAIR);
    zmq_bind (receiver, "inproc://step3");
    pthread_t thread;
    pthread_create (&thread, NULL, step2, context);

    //  等待信号
    char *string = s_recv (receiver);
    free (string);
    zmq_close (receiver);

    printf ("测试成功！\n");
    zmq_ctx_destroy (context);
    return 0;
}
```

这是使用 ZeroMQ 进行多线程的经典模式：

1. 两个线程通过 `inproc` 进行通信，使用共享上下文。
2. 父线程创建一个套接字，将其绑定到 `inproc://` 端点，*然后*启动子线程，将上下文传递给它。
3. 子线程创建第二个套接字，将其连接到该 `inproc://` 端点，*然后*向父线程发信号表示它已准备好。

注意使用此模式的多线程代码不能扩展到进程。如果你使用 `inproc` 和套接字对，你正在构建紧密绑定的应用程序，即线程在结构上相互依赖的应用程序。在延迟真正重要时这样做。另一种设计模式是松散绑定的应用程序，其中线程有自己的上下文并通过 `ipc` 或 `tcp` 进行通信。你可以轻松地将松散绑定的线程分解为单独的进程。

这是我们第一次显示使用 PAIR 套接字的示例。为什么使用 PAIR？其他套接字组合可能看起来工作，但它们都有可能干扰信号的副作用：

* 你可以对发送方使用 PUSH，对接收方使用 PULL。这看起来简单且会工作，但记住 PUSH 会将消息分发给所有可用的接收方。如果你意外启动两个接收方（例如，你已经有一个运行，你启动第二个），你会"丢失"一半的信号。PAIR 的优点是拒绝超过一个连接；对是*排他的*。

* 你可以对发送方使用 DEALER，对接收方使用 ROUTER。然而，ROUTER 将你的消息包装在"信封"中，意味着你的零大小信号变成多部分消息。如果你不关心数据并将任何东西视为有效信号，并且如果你不从套接字读取超过一次，那不重要。但是，如果你决定发送真实数据，你会突然发现 ROUTER 为你提供"错误"的消息。DEALER 也分发传出消息，给出与 PUSH 相同的风险。

* 你可以对发送方使用 PUB，对接收方使用 SUB。这将正确传递你的消息，完全按照你发送它们的方式，PUB 不像 PUSH 或 DEALER 那样分发。但是，你需要使用空订阅配置订阅者，这很烦人。

由于这些原因，PAIR 是线程对之间协调的最佳选择。

## 节点协调 {#Node-Coordination}

当你想要协调网络上的一组节点时，PAIR 套接字不再工作得很好。这是线程和节点策略不同的少数几个领域之一。主要是，节点来来去去，而线程通常是静态的。如果远程节点离开并回来，PAIR 套接字不会自动重新连接。

线程和节点之间的第二个重要区别是你通常有固定数量的线程，但节点数量更可变。让我们采用我们之前的一个场景（天气服务器和客户端）并使用节点协调来确保订阅者在启动时不会丢失数据。

应用程序将这样工作：

* 发布者提前知道它期望多少订阅者。这只是它从某个地方获得的魔术数字。

* 发布者启动并等待所有订阅者连接。这是节点协调部分。每个订阅者订阅，然后通过另一个套接字告诉发布者它已准备好。

* 当发布者连接了所有订阅者时，它开始发布数据。

在这种情况下，我们将使用 REQ-REP 套接字流来同步订阅者和发布者。这里是发布者：

```c
//  同步发布者
//  绑定 PUB socket 到 tcp://*:5561
//  绑定 REP socket 到 tcp://*:5562

#include "zhelpers.h"

#define SUBSCRIBERS_EXPECTED  10

int main (void)
{
    void *context = zmq_ctx_new ();

    //  用于发布更新的套接字
    void *publisher = zmq_socket (context, ZMQ_PUB);
    int sndhwm = 1100000;
    zmq_setsockopt (publisher, ZMQ_SNDHWM, &sndhwm, sizeof (int));
    zmq_bind (publisher, "tcp://*:5561");

    //  用于接收同步信号的套接字
    void *syncservice = zmq_socket (context, ZMQ_REP);
    zmq_bind (syncservice, "tcp://*:5562");

    //  从订阅者获取同步
    printf ("等待订阅者...\n");
    int subscribers = 0;
    while (subscribers < SUBSCRIBERS_EXPECTED) {
        //  - 等待同步请求
        char *string = s_recv (syncservice);
        free (string);
        //  - 发送同步回复
        s_send (syncservice, "");
        subscribers++;
    }
    //  现在广播恰好 1M 更新，然后结束
    printf ("广播消息\n");
    int update_nbr;
    for (update_nbr = 0; update_nbr < 1000000; update_nbr++)
        s_send (publisher, "Rhubarb");

    s_send (publisher, "END");

    zmq_close (publisher);
    zmq_close (syncservice);
    zmq_ctx_destroy (context);
    return 0;
}
```

这里是订阅者：

```c
//  同步订阅者
#include "zhelpers.h"

int main (void)
{
    void *context = zmq_ctx_new ();

    //  第一，连接我们的订阅者套接字
    void *subscriber = zmq_socket (context, ZMQ_SUB);
    zmq_connect (subscriber, "tcp://localhost:5561");
    zmq_setsockopt (subscriber, ZMQ_SUBSCRIBE, "", 0);

    //  0MQ 很快但不是瞬时的。如果你启动订阅者线程
    //  并立即开始发布，订阅者会错过你发送的任何东西。
    //  等待 SUB 套接字建立连接。
    sleep (1);

    //  第二，通过另一个套接字与发布者同步
    void *syncclient = zmq_socket (context, ZMQ_REQ);
    zmq_connect (syncclient, "tcp://localhost:5562");

    //  - 发送同步请求
    s_send (syncclient, "");

    //  - 等待同步回复
    char *string = s_recv (syncclient);
    free (string);

    //  第三，获取我们的更新并报告接收了多少
    int update_nbr = 0;
    while (1) {
        char *string = s_recv (subscriber);
        if (strcmp (string, "END") == 0) {
            free (string);
            break;
        }
        free (string);
        update_nbr++;
    }
    printf ("接收到 %d 更新\n", update_nbr);

    zmq_close (subscriber);
    zmq_close (syncclient);
    zmq_ctx_destroy (context);
    return 0;
}
```

这个 Bash shell 脚本将启动十个订阅者，然后启动发布者：

```bash
echo "启动订阅者..."
for ((a=0; a<10; a++)); do
    syncsub &
done
echo "启动发布者..."
syncpub
```

这给我们这个令人满意的输出：

```
启动订阅者...
启动发布者...
接收到 1000000 更新
接收到 1000000 更新
...
接收到 1000000 更新
接收到 1000000 更新
```

我们不能假设 SUB 连接将在 REQ/REP 对话完成时完成。如果你使用除 `inproc` 之外的任何传输，不能保证出站连接将以任何顺序完成。所以，示例在订阅和发送 REQ/REP 同步之间进行了一秒钟的暴力睡眠。

更鲁棒的模型可能是：

* 发布者打开 PUB 套接字并开始发送"Hello"消息（不是数据）。
* 订阅者连接 SUB 套接字，当它们接收到 Hello 消息时，它们通过 REQ/REP 套接字对告诉发布者。
* 当发布者有了所有必要的确认时，它开始发送真实数据。

## 零拷贝 {#Zero-Copy}

ZeroMQ 的消息 API 让你直接从应用程序缓冲区发送和接收消息，而无需复制数据。我们称之为*零拷贝*，它可以提高某些应用程序的性能。

你应该考虑在特定情况下使用零拷贝，即你以高频率发送大块内存（数千字节）。对于短消息，或较低的消息速率，使用零拷贝会使你的代码更混乱和更复杂，而没有可测量的好处。像所有优化一样，当你知道它有帮助时使用这个，并在之前和之后*测量*。

要进行零拷贝，你使用 `zmq_msg_init_data()` 创建一个消息，该消息引用已经用 `malloc()` 或其他分配器分配的数据块，然后将其传递给 `zmq_msg_send()`。当你创建消息时，你还传递一个 ZeroMQ 在完成发送消息时将调用来释放数据块的函数。这是最简单的示例，假设 `buffer` 是在堆上分配的 1,000 字节块：

```c
void my_free (void *data, void *hint) {
    free (data);
}
//  从缓冲区发送消息，我们分配它，ZeroMQ 将为我们释放它
zmq_msg_t message;
zmq_msg_init_data (&message, buffer, 1000, my_free, NULL);
zmq_msg_send (&message, socket, 0);
```

注意你在发送消息后不调用 `zmq_msg_close()`——`libzmq` 在实际完成发送消息时会自动执行此操作。

在接收时没有办法进行零拷贝：ZeroMQ 向你提供一个你可以根据需要存储的缓冲区，但它不会直接将数据写入应用程序缓冲区。

在写入时，ZeroMQ 的多部分消息与零拷贝很好地协作。在传统消息传递中，你需要将不同的缓冲区整理到一个可以发送的缓冲区中。这意味着复制数据。使用 ZeroMQ，你可以将来自不同来源的多个缓冲区作为单独的消息帧发送。将每个字段作为长度分隔的帧发送。对应用程序来说，它看起来像一系列发送和接收调用。但在内部，多个部分通过单个系统调用写入网络并读回，所以非常高效。

## 发布-订阅消息信封 {#Pub-Sub-Message-Envelopes}

在发布-订阅模式中，我们可以将键分离到一个单独的消息帧中，我们称之为*信封*。如果你想使用发布-订阅信封，自己制作它们。这是可选的，在之前的发布-订阅示例中我们没有这样做。使用发布-订阅信封对简单情况是多一点工作，但对真实情况更干净，其中键和数据自然是分离的东西。

订阅进行前缀匹配。也就是说，它们寻找"以 XYZ 开头的所有消息"。明显的问题是：如何分隔键和数据，以便前缀匹配不会意外匹配数据。最好的答案是使用信封，因为匹配不会跨越帧边界。这里是发布-订阅信封在代码中的最小示例。这个发布者发送两种类型的消息，A 和 B。

信封保存消息类型：

```c
//  发布-订阅信封发布者
//  注意数据我们发送是 "A" 和 "B"
#include "zhelpers.h"

int main (void)
{
    void *context = zmq_ctx_new ();
    void *publisher = zmq_socket (context, ZMQ_PUB);
    zmq_bind (publisher, "tcp://*:5563");

    while (1) {
        //  写两条消息，每条有信封和内容
        s_sendmore (publisher, "A");
        s_send (publisher, "我们不想看到这个");
        s_sendmore (publisher, "B");
        s_send (publisher, "我们想看到这个");
        sleep (1);
    }
    //  我们永远不会到达这里，但如果我们这样做了，这是很好的做法：
    zmq_close (publisher);
    zmq_ctx_destroy (context);
    return 0;
}
```

订阅者只想要 B 类型的消息：

```c
//  发布-订阅信封订阅者
#include "zhelpers.h"

int main (void)
{
    void *context = zmq_ctx_new ();
    void *subscriber = zmq_socket (context, ZMQ_SUB);
    zmq_connect (subscriber, "tcp://localhost:5563");
    zmq_setsockopt (subscriber, ZMQ_SUBSCRIBE, "B", 1);

    while (1) {
        //  读取信封与消息
        char *address = s_recv (subscriber);
        char *contents = s_recv (subscriber);
        printf ("[%s] %s\n", address, contents);
        free (address);
        free (contents);
    }
    zmq_close (subscriber);
    zmq_ctx_destroy (context);
    return 0;
}
```

当你运行这两个程序时，订阅者应该向你显示：

```
[B] 我们想看到这个
[B] 我们想看到这个
[B] 我们想看到这个
...
```

这个示例显示订阅过滤器拒绝或接受整个多部分消息（键加数据）。你永远不会得到多部分消息的一部分。如果你订阅多个发布者，并且你想知道它们的地址，以便你可以通过另一个套接字向它们发送数据（这是典型用例），创建一个三部分消息。

## 高水位标记 {#High-Water-Marks}

当你可以从进程到进程快速发送消息时，你很快会发现内存是宝贵的资源，并且可以轻易地被填满。进程中某个地方的几秒钟延迟可能会变成积压，除非你理解问题并采取预防措施，否则会炸毁服务器。

问题是这样的：想象你有进程 A 以高频率向进程 B 发送消息，B 正在处理它们。突然 B 变得非常忙（垃圾收集、CPU 过载等），在短时间内无法处理消息。对于一些重垃圾收集，可能是几秒钟，或者如果有更严重的问题，可能会更长。进程 A 仍在疯狂尝试发送的消息会发生什么？一些会坐在 B 的网络缓冲区中。一些会坐在以太网线本身上。一些会坐在 A 的网络缓冲区中。其余的会在 A 的内存中积累，就像 A 后面的应用程序发送它们一样快。如果你不采取一些预防措施，A 可能很容易耗尽内存并崩溃。

这是消息代理的一致、经典问题。让它更痛苦的是，表面上这是 B 的错，而 B 通常是 A 无法控制的用户编写的应用程序。

答案是什么？一个是将问题传递给上游。A 从其他地方获取消息。所以告诉那个进程，"停止！"等等。这被称为*流控制*。听起来合理，但如果你发送 Twitter 提要怎么办？你告诉整个世界在 B 整理自己时停止推特吗？

流控制在某些情况下工作，但在其他情况下不工作。传输层不能告诉应用层"停止"，就像地铁系统不能告诉大企业"请让你的员工再在工作中待半小时。我太忙了"。消息传递的答案是对缓冲区大小设置限制，然后当我们达到这些限制时，采取一些明智的行动。在某些情况下（虽然不是地铁系统），答案是丢弃消息。在其他情况下，最好的策略是等待。

ZeroMQ 使用 HWM（高水位标记）的概念来定义其内部管道的容量。套接字的每个连接都有自己的管道，以及用于发送和/或接收的 HWM，取决于套接字类型。一些套接字（PUB、PUSH）只有发送缓冲区。一些（SUB、PULL、REQ、REP）只有接收缓冲区。一些（DEALER、ROUTER、PAIR）既有发送又有接收缓冲区。

在 ZeroMQ v2.x 中，HWM 默认是无限的。这很容易，但对高容量发布者通常是致命的。在 ZeroMQ v3.x 中，它默认设置为 1,000，这更合理。如果你仍在使用 ZeroMQ v2.x，你应该始终在套接字上设置 HWM，无论是 1,000 来匹配 ZeroMQ v3.x 还是考虑你的消息大小和预期订阅者性能的另一个数字。

当你的套接字达到其 HWM 时，它将根据套接字类型阻塞或丢弃数据。PUB 和 ROUTER 套接字在达到其 HWM 时会丢弃数据，而其他套接字类型会阻塞。在 `inproc` 传输上，发送方和接收方共享相同的缓冲区，所以真正的 HWM 是两侧设置的 HWM 的总和。

最后，HWM 不是精确的；虽然你可能默认得到*多达* 1,000 条消息，但由于 `libzmq` 实现其队列的方式，真正的缓冲区大小可能要低得多（少至一半）。

## 消息丢失问题解决器 {#Missing-Message-Problem-Solver}

当你使用 ZeroMQ 构建应用程序时，你会不止一次遇到这个问题：丢失你期望接收的消息。我们整理了一个图表，逐步介绍了最常见的原因。

这是图形所说的总结：

* 在 SUB 套接字上，使用 `zmq_setsockopt()` 和 `ZMQ_SUBSCRIBE` 设置订阅，否则你不会得到消息。因为你通过前缀订阅消息，如果你订阅 ""（空订阅），你会得到所有内容。

* 如果你在 PUB 套接字开始发送数据*后*启动 SUB 套接字（即建立到 PUB 套接字的连接），你会丢失它在建立连接之前发布的任何内容。如果这是问题，设置你的架构，使 SUB 套接字先启动，然后 PUB 套接字开始发布。

* 即使你同步 SUB 和 PUB 套接字，你仍可能丢失消息。这是因为内部队列直到实际创建连接时才创建。如果你可以切换绑定/连接方向，使 SUB 套接字绑定，PUB 套接字连接，你可能会发现它更符合你的期望。

* 如果你使用 REP 和 REQ 套接字，并且你不坚持同步发送/接收/发送/接收顺序，ZeroMQ 会报告错误，你可能忽略。然后，看起来你在丢失消息。如果你使用 REQ 或 REP，坚持发送/接收顺序，并且在真实代码中，始终检查 ZeroMQ 调用的错误。

* 如果你使用 PUSH 套接字，你会发现第一个连接的 PULL 套接字会获得不公平的消息份额。准确的消息轮转只有当所有 PULL 套接字成功连接时才会发生，这可能需要几毫秒。作为 PUSH/PULL 的替代，对于较低的数据速率，考虑使用 ROUTER/DEALER 和负载平衡模式。

* 如果你在线程间共享套接字，不要。这会导致随机奇怪和崩溃。

* 如果你使用 `inproc`，确保两个套接字在同一个上下文中。否则连接方实际上会失败。另外，先绑定，然后连接。`inproc` 不是像 `tcp` 那样的断开连接传输。

* 如果你使用 ROUTER 套接字，通过发送格式错误的身份帧（或忘记发送身份帧）意外丢失消息是非常容易的。一般来说，在 ROUTER 套接字上设置 `ZMQ_ROUTER_MANDATORY` 选项是个好主意，但也要检查每个发送调用的返回代码。

* 最后，如果你真的不能弄清楚出了什么问题，制作一个重现问题的*最小*测试用例，并向 ZeroMQ 社区寻求帮助。
