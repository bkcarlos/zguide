---
weight: 3
title: '3. 高级请求-回复模式'
---

# 第3章 - 高级请求-回复模式 {#advanced-request-reply}

在第2章中，我们通过开发一系列小应用程序学习了使用 ZeroMQ 的基础知识，每次都探索 ZeroMQ 的新方面。我们将在本章中继续这种方法，探索构建在 ZeroMQ 核心请求-回复模式之上的高级模式。

我们将涵盖：

* 请求-回复机制如何工作
* 如何组合 REQ、REP、DEALER 和 ROUTER 套接字
* ROUTER 套接字如何详细工作
* 负载均衡模式
* 构建简单的负载均衡消息代理
* 为 ZeroMQ 设计高级 API
* 构建异步请求-回复服务器
* 详细的代理间路由示例

## 请求-回复机制 {#The-Request-Reply-Mechanisms}

我们已经简要地看了多部分消息。现在让我们看一个主要用例，即*回复消息信封*。信封是一种在不触及数据本身的情况下安全地将数据与地址打包的方法。通过将回复地址分离到信封中，我们使得编写通用中介（如 API 和代理）成为可能，这些中介可以创建、读取和删除地址，无论消息负载或结构如何。

在请求-回复模式中，信封保存回复的返回地址。这是 ZeroMQ 网络在无状态情况下如何创建往返请求-回复对话的方式。

当你使用 REQ 和 REP 套接字时，你甚至看不到信封；这些套接字自动处理它们。但对于大多数有趣的请求-回复模式，你需要理解信封，特别是 ROUTER 套接字。我们将逐步解决这个问题。

### 简单回复信封 {#The-Simple-Reply-Envelope}

请求-回复交换由一个*请求*消息和一个最终*回复*消息组成。在简单的请求-回复模式中，每个请求有一个回复。在更高级的模式中，请求和回复可以异步流动。但是，回复信封总是以相同的方式工作。

ZeroMQ 回复信封正式由零个或多个回复地址组成，然后是空帧（信封分隔符），然后是消息体（零个或多个帧）。信封由多个套接字在链中协同工作创建。我们将分解这个。

我们将从通过 REQ 套接字发送"Hello"开始。REQ 套接字创建最简单的回复信封，它没有地址，只有一个空分隔符帧和包含"Hello"字符串的消息帧。这是一个两帧消息。

REP 套接字做匹配的工作：它剥离信封，直到并包括分隔符帧，保存整个信封，并将"Hello"字符串传递给应用程序。因此我们原来的 Hello World 示例在内部使用了请求-回复信封，但应用程序从未看到它们。

如果你监视在 `hwclient` 和 `hwserver` 之间流动的网络数据，这是你会看到的：每个请求和每个回复实际上都是两帧，一个空帧然后是主体。对于简单的 REQ-REP 对话，这似乎没有太大意义。但是当我们探索 ROUTER 和 DEALER 如何处理信封时，你会看到原因。

### 扩展回复信封 {#The-Extended-Reply-Envelope}

现在让我们用中间的 ROUTER-DEALER 代理扩展 REQ-REP 对，看看这如何影响回复信封。这是我们在第2章中已经看到的*扩展请求-回复模式*。实际上，我们可以插入任意数量的代理步骤。机制是相同的。

代理执行此操作，用伪代码：

```
准备上下文、前端和后端套接字
while true:
    在两个套接字上轮询
    if 前端有输入:
        从前端读取所有帧
        发送到后端
    if 后端有输入:
        从后端读取所有帧
        发送到前端
```

ROUTER 套接字与其他套接字不同，它跟踪它拥有的每个连接，并告诉调用者这些连接。它告诉调用者的方式是在接收到的每条消息前面放置连接*身份*。身份，有时称为*地址*，只是一个二进制字符串，除了"这是连接的唯一句柄"之外没有意义。然后，当你通过 ROUTER 套接字发送消息时，你首先发送身份帧。

`zmq_socket()` 手册页这样描述它：

> 当接收消息时，ZMQ_ROUTER 套接字应在将消息传递给应用程序之前，在消息前面添加包含原始对等体身份的消息部分。接收到的消息在所有连接的对等体之间公平排队。当发送消息时，ZMQ_ROUTER 套接字应删除消息的第一部分，并使用它来确定消息应路由到的对等体的身份。

作为历史注释，ZeroMQ v2.2 和更早版本使用 UUID 作为身份。ZeroMQ v3.0 和更高版本默认生成 5 字节身份（0 + 随机 32 位整数）。对网络性能有一些影响，但只有当你使用多个代理跳时，这很少见。更改主要是为了通过删除对 UUID 库的依赖来简化构建 `libzmq`。

身份是一个难以理解的概念，但如果你想成为 ZeroMQ 专家，这是必不可少的。ROUTER 套接字为它处理的每个连接*发明*一个随机身份。如果有三个 REQ 套接字连接到 ROUTER 套接字，它将发明三个随机身份，每个 REQ 套接字一个。

所以如果我们继续我们的工作示例，假设 REQ 套接字有 3 字节身份 `ABC`。在内部，这意味着 ROUTER 套接字保持一个哈希表，它可以在其中搜索 `ABC` 并找到 REQ 套接字的 TCP 连接。

当我们从 ROUTER 套接字接收消息时，我们得到三帧。

代理循环的核心是"从一个套接字读取，写入另一个"，所以我们字面上将这三帧发送到 DEALER 套接字。如果你现在嗅探网络流量，你会看到这三帧从 DEALER 套接字飞到 REP 套接字。REP 套接字像之前一样，剥离整个信封包括新的回复地址，并再次将"Hello"传递给调用者。

顺便说一下，REP 套接字一次只能处理一个请求-回复交换，这就是为什么如果你尝试读取多个请求或发送多个回复而不坚持严格的接收-发送循环，它会给出错误。

你现在应该能够可视化返回路径。当 `hwserver` 发送"World"回来时，REP 套接字用它保存的信封包装它，并通过线路向 DEALER 套接字发送三帧回复消息。

现在 DEALER 读取这三帧，并通过 ROUTER 套接字发送所有三帧。ROUTER 获取消息的第一帧，即 `ABC` 身份，并查找此连接。如果找到，它然后将接下来的两帧泵送到线路上。

REQ 套接字拾取此消息，并检查第一帧是否为空分隔符，确实如此。REQ 套接字丢弃该帧并将"World"传递给调用应用程序，调用应用程序将其打印出来，让第一次看 ZeroMQ 的年轻我们感到惊讶。

### 这有什么好处？{#What-This-Good-For}

说实话，严格请求-回复或扩展请求-回复的用例有些有限。一方面，没有简单的方法从常见故障中恢复，如服务器因错误的应用程序代码而崩溃。我们将在可靠请求-回复章节中更多地看到这一点。但是一旦你掌握了这四个套接字处理信封的方式，以及它们如何相互交谈，你可以做非常有用的事情。我们看到了 ROUTER 如何使用回复信封来决定将回复路由回哪个客户端 REQ 套接字。现在让我们用另一种方式表达这一点：

* 每次 ROUTER 给你一条消息时，它告诉你这来自哪个对等体，作为身份。
* 你可以使用这个与哈希表（以身份为键）来跟踪新对等体的到达。
* 如果你在消息的第一帧前面加上身份，ROUTER 将异步地将消息路由到连接到它的任何对等体。

ROUTER 套接字不关心整个信封。它们不知道空分隔符的任何信息。它们只关心让它们弄清楚将消息发送到哪个连接的那一个身份帧。

### 请求-回复套接字回顾 {#Recap-of-Request-Reply-Sockets}

让我们回顾一下：

* REQ 套接字向网络发送，在消息数据前面有一个空分隔符帧。REQ 套接字是同步的。REQ 套接字总是发送一个请求，然后等待一个回复。REQ 套接字一次与一个对等体交谈。如果你将 REQ 套接字连接到多个对等体，请求分发到每个对等体，回复预期从每个对等体轮流一次。

* REP 套接字读取并保存所有身份帧，直到并包括空分隔符，然后将以下帧或帧传递给调用者。REP 套接字是同步的，一次与一个对等体交谈。如果你将 REP 套接字连接到多个对等体，请求以公平方式从对等体读取，回复总是发送到发出最后请求的同一对等体。

* DEALER 套接字对回复信封视而不见，将其作为任何多部分消息处理。DEALER 套接字是异步的，就像 PUSH 和 PULL 组合一样。它们在所有连接之间分发发送的消息，并从所有连接公平排队接收的消息。

* ROUTER 套接字像 DEALER 一样对回复信封视而不见。它为其连接创建身份，并将这些身份作为任何接收消息的第一帧传递给调用者。相反，当调用者发送消息时，它使用第一个消息帧作为身份来查找要发送到的连接。ROUTER 是异步的。

## 请求-回复组合 {#Request-Reply-Combinations}

我们有四个请求-回复套接字，每个都有特定的行为。我们已经看到它们如何在简单和扩展请求-回复模式中连接。但这些套接字是你可以用来解决许多问题的构建块。

这些是合法的组合：

* REQ 到 REP
* DEALER 到 REP
* REQ 到 ROUTER
* DEALER 到 ROUTER
* DEALER 到 DEALER
* ROUTER 到 ROUTER

这些组合是无效的（我将解释为什么）：

* REQ 到 REQ
* REQ 到 DEALER
* REP 到 REP
* REP 到 ROUTER

这里有一些记住语义的技巧。DEALER 像异步 REQ 套接字，ROUTER 像异步 REP 套接字。在我们使用 REQ 套接字的地方，我们可以使用 DEALER；我们只是必须自己读写信封。在我们使用 REP 套接字的地方，我们可以放一个 ROUTER；我们只需要自己管理身份。

将 REQ 和 DEALER 套接字视为"客户端"，将 REP 和 ROUTER 套接字视为"服务器"。大多数情况下，你会想要绑定 REP 和 ROUTER 套接字，并将 REQ 和 DEALER 套接字连接到它们。这并不总是这么简单，但这是一个干净且难忘的起点。

### REQ 到 REP 组合 {#The-REQ-to-REP-Combination}

我们已经涵盖了与 REP 服务器交谈的 REQ 客户端，但让我们看一个方面：REQ 客户端*必须*启动消息流。REP 服务器不能与还没有首先发送请求的 REQ 客户端交谈。技术上，这甚至是不可能的，如果你尝试，API 也会返回 `EFSM` 错误。

### DEALER 到 REP 组合 {#The-DEALER-to-REP-Combination}

现在，让我们用 DEALER 替换 REQ 客户端。这给我们一个异步客户端，可以与多个 REP 服务器交谈。如果我们使用 DEALER 重写"Hello World"客户端，我们能够发送任意数量的"Hello"请求而无需等待回复。

当我们使用 DEALER 与 REP 套接字交谈时，我们*必须*准确地模拟 REQ 套接字本来会发送的信封，否则 REP 套接字会将消息作为无效丢弃。所以，要发送消息，我们：

* 发送设置了 MORE 标志的空消息帧；然后
* 发送消息体。

当我们接收消息时，我们：

* 接收第一帧，如果它不为空，丢弃整个消息；
* 接收下一帧并将其传递给应用程序。

### REQ 到 ROUTER 组合 {#The-REQ-to-ROUTER-Combination}

以同样的方式，我们可以用 DEALER 替换 REQ，我们可以用 ROUTER 替换 REP。这给我们一个异步服务器，可以同时与多个 REQ 客户端交谈。如果我们使用 ROUTER 重写"Hello World"服务器，我们能够并行处理任意数量的"Hello"请求。我们在第2章的 `mtserver` 示例中看到了这一点。

我们可以以两种不同的方式使用 ROUTER：

* 作为在前端和后端套接字之间切换消息的代理。
* 作为读取消息并对其进行操作的应用程序。

在第一种情况下，ROUTER 只是读取所有帧，包括人工身份帧，并盲目地传递它们。在第二种情况下，ROUTER *必须*知道它被发送的回复信封的格式。由于另一个对等体是 REQ 套接字，ROUTER 获得身份帧、空帧，然后是数据帧。

### DEALER 到 ROUTER 组合 {#The-DEALER-to-ROUTER-Combination}

现在我们可以将 REQ 和 REP 都换成 DEALER 和 ROUTER，得到最强大的套接字组合，即 DEALER 与 ROUTER 交谈。它给我们异步客户端与异步服务器交谈，双方都完全控制消息格式。

因为 DEALER 和 ROUTER 都可以处理任意消息格式，如果你希望安全地使用这些，你必须成为一点协议设计师。至少你必须决定是否希望模拟 REQ/REP 回复信封。这取决于你是否实际需要发送回复。

### DEALER 到 DEALER 组合 {#The-DEALER-to-DEALER-Combination}

你可以用 ROUTER 替换 REP，但如果 DEALER 与一个且仅一个对等体交谈，你也可以用 DEALER 替换 REP。

当你用 DEALER 替换 REP 时，你的工作者突然可以完全异步，发送任意数量的回复。成本是你必须自己管理回复信封，并且正确处理，否则什么都不会工作。我们稍后会看到一个工作示例。现在只是说 DEALER 到 DEALER 是最难正确处理的模式之一，幸运的是我们很少需要它。

### ROUTER 到 ROUTER 组合 {#The-ROUTER-to-ROUTER-Combination}

这对 N 到 N 连接来说听起来完美，但它是最难使用的组合。在你熟练掌握 ZeroMQ 之前，你应该避免它。我们将在可靠请求-回复章节的自由职业者模式中看到一个示例，以及在移动部件章节中对等工作的替代 DEALER 到 ROUTER 设计。

### 无效组合 {#Invalid-Combinations}

大多数情况下，尝试将客户端连接到客户端，或服务器连接到服务器是一个坏主意，不会工作。但是，与其给出一般的模糊警告，我将详细解释：

* REQ 到 REQ：双方都想通过向对方发送消息开始，这只有在你计时，使两个对等体同时交换消息时才能工作。甚至想想都让我头疼。

* REQ 到 DEALER：理论上你可以这样做，但如果你添加第二个 REQ，它会中断，因为 DEALER 无法向原始对等体发送回复。因此 REQ 套接字会混乱，和/或返回意图给另一个客户端的消息。

* REP 到 REP：双方都会等待对方发送第一条消息。

* REP 到 ROUTER：理论上 ROUTER 套接字可以启动对话并发送格式正确的请求，如果它知道 REP 套接字已连接*并且*它知道该连接的身份。这很混乱，比 DEALER 到 ROUTER 没有任何优势。

有效与无效分解的共同点是 ZeroMQ 套接字连接总是偏向于绑定到端点的一个对等体，以及连接到那的另一个对等体。此外，哪一边绑定哪一边连接不是任意的，而是遵循自然模式。我们期望"在那里"的一边绑定：它将是服务器、代理、发布者、收集器。"来来去去"的一边连接：它将是客户端和工作者。记住这一点将帮助你设计更好的 ZeroMQ 架构。

## 探索 ROUTER 套接字 {#Exploring-ROUTER-Sockets}

让我们更仔细地看看 ROUTER 套接字。我们已经看到它们如何通过将单个消息路由到特定连接来工作。我将更详细地解释我们如何识别那些连接，以及当 ROUTER 套接字无法发送消息时会做什么。

### 身份和地址 {#Identities-and-Addresses}

ZeroMQ 中的*身份*概念专门指 ROUTER 套接字以及它们如何识别它们与其他套接字的连接。更广泛地说，身份用作回复信封中的地址。在大多数情况下，身份是任意的且对 ROUTER 套接字本地：它是哈希表中的查找键。独立地，对等体可以有物理地址（如"tcp://192.168.55.117:5670"的网络端点）或逻辑地址（UUID 或电子邮件地址或其他唯一键）。

使用 ROUTER 套接字与特定对等体交谈的应用程序可以将逻辑地址转换为身份，如果它已构建必要的哈希表。因为 ROUTER 套接字只有在对等体发送消息时才宣布连接的身份（到特定对等体），你实际上只能回复消息，而不能自发地与对等体交谈。

即使你翻转规则并使 ROUTER 连接到对等体而不是等待对等体连接到 ROUTER，这也是正确的。但是你可以强制 ROUTER 套接字使用逻辑地址代替其身份。`zmq_setsockopt` 参考页称此为*设置套接字身份*。它的工作原理如下：

* 对等应用程序在绑定或连接*之前*设置其对等套接字（DEALER 或 REQ）的 `ZMQ_IDENTITY` 选项。
* 通常对等体然后连接到已绑定的 ROUTER 套接字。但 ROUTER 也可以连接到对等体。
* 在连接时，对等套接字告诉路由器套接字，"请为此连接使用此身份"。
* 如果对等套接字没有说，路由器为连接生成其通常的任意随机身份。
* ROUTER 套接字现在为来自该对等体的任何消息向应用程序提供此逻辑地址作为前缀身份帧。
* ROUTER 也期望逻辑地址作为任何传出消息的前缀身份帧。

这是两个对等体连接到 ROUTER 套接字的简单示例，其中一个强加逻辑地址"PEER2"：

```c
//  演示身份如何用于 ROUTER 套接字
#include "zhelpers.h"

int main (void)
{
    void *context = zmq_ctx_new ();

    void *sink = zmq_socket (context, ZMQ_ROUTER);
    zmq_bind (sink, "inproc://example");

    //  第一个身份说"没有什么特别的"
    void *anonymous = zmq_socket (context, ZMQ_REQ);
    zmq_connect (anonymous, "inproc://example");
    s_send (anonymous, "ROUTER 使用生成的 5 字节身份");
    s_dump (sink);

    //  设置打印标志（自由形式字符串）
    void *identified = zmq_socket (context, ZMQ_REQ);
    zmq_setsockopt (identified, ZMQ_IDENTITY, "PEER2", 5);
    zmq_connect (identified, "inproc://example");
    s_send (identified, "ROUTER 使用 REQ 的套接字身份");
    s_dump (sink);

    zmq_close (sink);
    zmq_close (anonymous);
    zmq_close (identified);
    zmq_ctx_destroy (context);
    return 0;
}
```

程序打印的内容：

```
----------------------------------------
[005] 006B8B4567
[000]
[039] ROUTER 使用生成的 5 字节身份
----------------------------------------
[005] PEER2
[000]
[038] ROUTER 使用 REQ 的套接字身份
```

### ROUTER 错误处理 {#ROUTER-Error-Handling}

ROUTER 套接字确实有一种相当残酷的方式处理它们无法发送到任何地方的消息：它们默默地丢弃它们。这是在工作代码中有意义的态度，但它使调试变得困难。"将身份作为第一帧发送"的方法足够棘手，我们在学习时经常搞错，当我们搞砸时 ROUTER 的石头般沉默不是很有建设性。

从 ZeroMQ v3.2 开始，有一个套接字选项你可以设置来捕获此错误：`ZMQ_ROUTER_MANDATORY`。在 ROUTER 套接字上设置它，然后当你在发送调用中提供不可路由的身份时，套接字将发出 `EHOSTUNREACH` 错误信号。

## 负载均衡模式 {#The-Load-Balancing-Pattern}

现在让我们看一些代码。我们将看到如何将 ROUTER 套接字连接到 REQ 套接字，然后连接到 DEALER 套接字。这两个示例遵循相同的逻辑，即*负载均衡*模式。这种模式是我们第一次接触使用 ROUTER 套接字进行故意路由，而不是简单地充当回复通道。

负载均衡模式非常常见，我们将在本书中多次看到它。它解决了简单轮询路由（如 PUSH 和 DEALER 提供的）的主要问题，如果任务不是都大致花费相同的时间，轮询会变得低效。

这是邮局的类比。如果你每个柜台有一个队列，你有一些人买邮票（快速、简单的交易），一些人开新账户（非常慢的交易），那么你会发现买邮票的人不公平地被困在队列中。就像在邮局一样，如果你的消息架构不公平，人们会感到恼火。

邮局的解决方案是创建一个队列，这样即使一个或两个柜台被慢工作困住，其他柜台将继续为客户提供先到先服务的服务。

PUSH 和 DEALER 使用简单方法的一个原因是纯粹的性能。如果你到达任何美国主要机场，你会发现在移民局有长长的人群队列。边境巡逻官员会提前派人在每个柜台排队，而不是使用单一队列。让人们提前走五十码可以为每位乘客节省一两分钟。而且因为每次护照检查大致花费相同的时间，所以多少是公平的。这是 PUSH 和 DEALER 的策略：提前发送工作负载，以减少传输距离。

这是 ZeroMQ 的一个反复出现的主题：世界的问题是多样的，你可以从以正确的方式解决不同的问题中受益。机场不是邮局，一种尺寸实际上不适合任何人。

让我们回到工作者（DEALER 或 REQ）连接到代理（ROUTER）的场景。代理必须知道工作者何时准备就绪，并保持工作者列表，以便它可以每次采用*最近最少使用*的工作者。

解决方案实际上非常简单：工作者在启动时发送"准备就绪"消息，并在完成每项任务后发送。代理逐一读取这些消息。每次读取消息时，它来自最后使用的工作者。因为我们使用 ROUTER 套接字，我们得到一个身份，然后可以用来将任务发送回工作者。

这是对请求-回复的扭曲，因为任务与回复一起发送，任务的任何响应都作为新请求发送。以下代码示例应该使其更清楚。

### ROUTER 代理和 REQ 工作者 {#ROUTER-Broker-and-REQ-Workers}

这是使用 ROUTER 代理与一组 REQ 工作者交谈的负载均衡模式的示例：

```c
//  ROUTER-to-REQ 示例
//  代理与多个工作者进行负载均衡，每个工作者使用 REQ 套接字
#include "zhelpers.h"
#include <pthread.h>

#define NBR_WORKERS 10

static void *
worker_task (void *args)
{
    void *context = zmq_ctx_new ();
    void *worker = zmq_socket (context, ZMQ_REQ);

    //  设置随机身份使跟踪更容易
    char identity [10];
    sprintf (identity, "%04X-%04X", randof (0x10000), randof (0x10000));
    zmq_setsockopt (worker, ZMQ_IDENTITY, identity, strlen (identity));
    zmq_connect (worker, "ipc://rtreq.ipc");

    int total = 0;
    while (1) {
        //  告诉代理我们准备好工作
        s_send (worker, "Hi Boss");

        //  获取工作负载分配，直到完成
        char *workload = s_recv (worker);
        if (strcmp (workload, "Fired!") == 0) {
            free (workload);
            break;
        }
        total++;

        //  做一些随机工作
        s_sleep (randof (500) + 1);
        free (workload);
    }
    printf ("已完成：%d 任务\n", total);
    zmq_close (worker);
    zmq_ctx_destroy (context);
    return NULL;
}

//  在这里我们必须构建 REQ 友好的信封，
//  当我们直接使用 ROUTER 套接字读取 REQ 套接字时
int main (void)
{
    void *context = zmq_ctx_new ();
    void *broker = zmq_socket (context, ZMQ_ROUTER);
    zmq_bind (broker, "ipc://rtreq.ipc");

    int worker_nbr;
    for (worker_nbr = 0; worker_nbr < NBR_WORKERS; worker_nbr++) {
        pthread_t worker;
        pthread_create (&worker, NULL, worker_task, NULL);
    }

    //  运行 5 秒然后停止工作者
    int64_t end_time = s_clock () + 5000;
    int workers_fired = 0;
    while (1) {
        //  下一个准备好的工作者，如果有的话
        char *identity = s_recv (broker);
        s_recv (broker);     //  信封分隔符
        s_recv (broker);     //  响应从工作者
        s_sendmore (broker, identity);
        s_sendmore (broker, "");

        //  鼓励工作者直到时间结束
        if (s_clock () < end_time)
            s_send (broker, "Work harder!");
        else {
            s_send (broker, "Fired!");
            if (++workers_fired == NBR_WORKERS)
                break;
        }
        free (identity);
    }
    zmq_close (broker);
    zmq_ctx_destroy (context);
    return 0;
}
```

示例运行五秒钟，然后每个工作者打印它们处理了多少任务。如果路由工作，我们会期望公平的工作分配：

```
已完成：20 任务
已完成：18 任务
已完成：21 任务
已完成：23 任务
已完成：19 任务
已完成：21 任务
已完成：17 任务
已完成：17 任务
已完成：25 任务
已完成：19 任务
```

要在此示例中与工作者交谈，我们必须创建一个 REQ 友好的信封，由身份加空信封分隔符帧组成。

### ROUTER 代理和 DEALER 工作者 {#ROUTER-Broker-and-DEALER-Workers}

在任何你可以使用 REQ 的地方，你都可以使用 DEALER。有两个具体区别：

* REQ 套接字总是在任何数据帧之前发送空分隔符帧；DEALER 不会。
* REQ 套接字在接收回复之前只发送一条消息；DEALER 是完全异步的。

同步与异步行为对我们的示例没有影响，因为我们正在做严格的请求-回复。当我们处理从故障中恢复时，它更相关，我们将在可靠请求-回复章节中讲到。

现在让我们看看完全相同的示例，但用 DEALER 套接字替换 REQ 套接字：

```c
//  ROUTER-to-DEALER 示例
//  代理与多个工作者进行负载均衡，每个工作者使用 DEALER 套接字
#include "zhelpers.h"
#include <pthread.h>

#define NBR_WORKERS 10

static void *
worker_task (void *args)
{
    void *context = zmq_ctx_new ();
    void *worker = zmq_socket (context, ZMQ_DEALER);

    //  设置随机身份使跟踪更容易
    char identity [10];
    sprintf (identity, "%04X-%04X", randof (0x10000), randof (0x10000));
    zmq_setsockopt (worker, ZMQ_IDENTITY, identity, strlen (identity));
    zmq_connect (worker, "ipc://rtdealer.ipc");

    int total = 0;
    while (1) {
        //  告诉代理我们准备好工作
        s_sendmore (worker, "");
        s_send (worker, "Hi Boss");

        //  获取工作负载分配，直到完成
        char *empty = s_recv (worker);
        free (empty);
        char *workload = s_recv (worker);
        if (strcmp (workload, "Fired!") == 0) {
            free (workload);
            break;
        }
        total++;

        //  做一些随机工作
        s_sleep (randof (500) + 1);
        free (workload);
    }
    printf ("已完成：%d 任务\n", total);
    zmq_close (worker);
    zmq_ctx_destroy (context);
    return NULL;
}

//  当我们直接使用 ROUTER 套接字读取 DEALER 套接字时，
//  我们得到身份框架，空框架，然后数据框架。
int main (void)
{
    void *context = zmq_ctx_new ();
    void *broker = zmq_socket (context, ZMQ_ROUTER);
    zmq_bind (broker, "ipc://rtdealer.ipc");

    int worker_nbr;
    for (worker_nbr = 0; worker_nbr < NBR_WORKERS; worker_nbr++) {
        pthread_t worker;
        pthread_create (&worker, NULL, worker_task, NULL);
    }

    //  运行 5 秒然后停止工作者
    int64_t end_time = s_clock () + 5000;
    int workers_fired = 0;
    while (1) {
        //  下一个准备好的工作者，如果有的话
        char *identity = s_recv (broker);
        s_recv (broker);     //  信封分隔符
        s_recv (broker);     //  响应从工作者
        s_sendmore (broker, identity);
        s_sendmore (broker, "");

        //  鼓励工作者直到时间结束
        if (s_clock () < end_time)
            s_send (broker, "Work harder!");
        else {
            s_send (broker, "Fired!");
            if (++workers_fired == NBR_WORKERS)
                break;
        }
        free (identity);
    }
    zmq_close (broker);
    zmq_ctx_destroy (context);
    return 0;
}
```

代码几乎相同，除了工作者使用 DEALER 套接字，并在数据帧之前读写空帧。这是我想要与 REQ 工作者保持兼容性时使用的方法。

但是，记住空分隔符帧的原因：它是为了允许终止在 REP 套接字中的多跳扩展请求，REP 套接字使用该分隔符分离回复信封，以便它可以将数据帧传递给其应用程序。

如果我们永远不需要将消息传递给 REP 套接字，我们可以简单地在两边丢弃空分隔符帧，这使事情更简单。这通常是我用于纯 DEALER 到 ROUTER 协议的设计。

## 负载均衡消息代理 {#A-Load-Balancing-Message-Broker}

前面的示例是半完整的。它可以使用虚拟请求和回复管理一组工作者，但它没有办法与客户端交谈。如果我们添加第二个*前端* ROUTER 套接字来接受客户端请求，并将我们的示例转换为可以在前端和后端之间切换消息的代理，我们就得到了一个有用且可重用的小型负载均衡消息代理。

这个代理执行以下操作：

* 接受来自一组客户端的连接。
* 接受来自一组工作者的连接。
* 接收来自客户端的请求，并将这些请求保存在一个队列中。
* 使用负载均衡模式将这些请求发送给工作者。
* 接收来自工作者的回复。
* 将这些回复发送回原始请求客户端。

代理代码相当长，但值得理解：

```c
//  负载均衡代理
//  演示简单而优雅的负载均衡
#include "zhelpers.h"
#include <pthread.h>

#define NBR_CLIENTS 10
#define NBR_WORKERS 3

//  基本请求-回复客户端使用 REQ 套接字
static void *
client_task (void *args)
{
    void *context = zmq_ctx_new ();
    void *client = zmq_socket (context, ZMQ_REQ);

    //  设置随机身份使跟踪更容易
    char identity [10];
    sprintf (identity, "%04X-%04X", randof (0x10000), randof (0x10000));
    zmq_setsockopt (client, ZMQ_IDENTITY, identity, strlen (identity));
    zmq_connect (client, "ipc://frontend.ipc");

    //  发送请求，获取回复
    s_send (client, "HELLO");
    char *reply = s_recv (client);
    printf ("客户端：%s\n", reply);
    free (reply);
    zmq_close (client);
    zmq_ctx_destroy (context);
    return NULL;
}

//  工作者使用 REQ 套接字实现负载均衡模式
static void *
worker_task (void *args)
{
    void *context = zmq_ctx_new ();
    void *worker = zmq_socket (context, ZMQ_REQ);

    //  设置随机身份使跟踪更容易
    char identity [10];
    sprintf (identity, "%04X-%04X", randof (0x10000), randof (0x10000));
    zmq_setsockopt (worker, ZMQ_IDENTITY, identity, strlen (identity));
    zmq_connect (worker, "ipc://backend.ipc");

    //  告诉代理我们准备好工作
    s_send (worker, "READY");

    while (1) {
        //  读取并保存所有帧直到得到空帧
        //  在这个例子中总共有三帧
        //  地址 + 空 + 请求
        char *address = s_recv (worker);
        char *empty = s_recv (worker);
        assert (*empty == 0);
        free (empty);

        //  获取请求，发送回复
        char *request = s_recv (worker);
        printf ("工作者：%s\n", request);
        free (request);

        s_sendmore (worker, address);
        s_sendmore (worker, "");
        s_send     (worker, "OK");
        free (address);
    }
    zmq_close (worker);
    zmq_ctx_destroy (context);
    return NULL;
}

int main (void)
{
    //  准备我们的上下文和套接字
    void *context = zmq_ctx_new ();
    void *frontend = zmq_socket (context, ZMQ_ROUTER);
    void *backend  = zmq_socket (context, ZMQ_ROUTER);
    zmq_bind (frontend, "ipc://frontend.ipc");
    zmq_bind (backend,  "ipc://backend.ipc");

    int client_nbr;
    for (client_nbr = 0; client_nbr < NBR_CLIENTS; client_nbr++) {
        pthread_t client;
        pthread_create (&client, NULL, client_task, NULL);
    }
    int worker_nbr;
    for (worker_nbr = 0; worker_nbr < NBR_WORKERS; worker_nbr++) {
        pthread_t worker;
        pthread_create (&worker, NULL, worker_task, NULL);
    }

    //  这里是主要的负载均衡循环
    //  轮询两个套接字进行活动
    zmq_pollitem_t items [] = {
        { backend,  0, ZMQ_POLLIN, 0 },
        { frontend, 0, ZMQ_POLLIN, 0 }
    };
    //  工作者队列，每个条目是一个工作者身份
    int capacity = 0;
    zlist_t *workers = zlist_new ();

    while (1) {
        int rc = zmq_poll (items, capacity? 2: 1, -1);
        if (rc == -1)
            break;              //  中断

        //  处理来自后端的工作者活动
        if (items [0].revents & ZMQ_POLLIN) {
            //  使用工作者身份进行负载均衡
            char *worker_id = s_recv (backend);
            assert (capacity < NBR_WORKERS);
            capacity++;
            zlist_append (workers, worker_id);

            //  第二帧是空的
            char *empty = s_recv (backend);
            assert (empty [0] == 0);
            free (empty);

            //  第三帧是 READY 或者是客户端回复地址
            char *client_id = s_recv (backend);

            //  如果客户端回复，发送其余部分给客户端
            if (strcmp (client_id, "READY") != 0) {
                empty = s_recv (backend);
                assert (empty [0] == 0);
                free (empty);
                char *reply = s_recv (backend);
                s_sendmore (frontend, client_id);
                s_sendmore (frontend, "");
                s_send     (frontend, reply);
                free (reply);
                if (--client_nbr == 0)
                    break; //  N 个客户端，所以退出
            }
            free (client_id);
        }
        if (items [1].revents & ZMQ_POLLIN) {
            //  现在从客户端获取下一个请求
            //  我们将路由到下一个工作者
            char *client_id = s_recv (frontend);
            char *empty = s_recv (frontend);
            assert (empty [0] == 0);
            free (empty);
            char *request = s_recv (frontend);

            //  出队并丢弃下一个工作者地址
            char *worker_id = zlist_pop (workers);
            capacity--;
            s_sendmore (backend, worker_id);
            s_sendmore (backend, "");
            s_sendmore (backend, client_id);
            s_sendmore (backend, "");
            s_send     (backend, request);

            free (client_id);
            free (request);
            free (worker_id);
        }
    }
    zlist_destroy (&workers);
    zmq_close (frontend);
    zmq_close (backend);
    zmq_ctx_destroy (context);
    return 0;
}
```

这个程序的困难部分是（a）每个套接字读写的信封，以及（b）负载均衡算法。我们将依次处理这些，从消息信封格式开始。

让我们从客户端到工作者再回来走一遍完整的请求-回复链。在这个代码中，我们设置客户端和工作者套接字的身份，使跟踪消息帧更容易。在现实中，我们会让 ROUTER 套接字为连接发明身份。假设客户端的身份是"CLIENT"，工作者的身份是"WORKER"。客户端应用程序发送包含"Hello"的单帧。

因为 REQ 套接字添加其空分隔符帧，ROUTER 套接字添加其连接身份，代理从前端 ROUTER 套接字读取客户端地址、空分隔符帧和数据部分。

代理将此发送给工作者，前缀是所选工作者的地址，加上额外的空部分以保持另一端的 REQ 快乐。

这个复杂的信封栈首先被后端 ROUTER 套接字咀嚼，它删除第一帧。然后工作者中的 REQ 套接字删除空部分，并向工作者应用程序提供其余部分。

工作者必须保存信封（即直到并包括空消息帧的所有部分），然后它可以对数据部分做需要的事情。注意 REP 套接字会自动执行此操作，但我们使用 REQ-ROUTER 模式，以便我们可以获得适当的负载均衡。

在返回路径上，消息与它们进来时相同，即后端套接字给代理一个五部分的消息，代理向前端套接字发送一个三部分的消息，客户端获得一个一部分的消息。

现在让我们看看负载均衡算法。它要求客户端和工作者都使用 REQ 套接字，并且工作者正确存储和重放它们收到的消息上的信封。算法是：

* 创建一个轮询集，总是轮询后端，只有在有一个或多个工作者可用时才轮询前端。

* 使用无限超时轮询活动。

* 如果后端有活动，我们要么有"准备就绪"消息，要么有客户端的回复。在任一情况下，我们将工作者地址（第一部分）存储在我们的工作者队列中，如果其余是客户端回复，我们通过前端将其发送回该客户端。

* 如果前端有活动，我们采用客户端请求，弹出下一个工作者（这是最后使用的），并将请求发送到后端。这意味着发送工作者地址、空部分，然后是客户端请求的三部分。

你现在应该看到你可以基于工作者在其初始"准备就绪"消息中提供的信息，通过变化重用和扩展负载均衡算法。例如，工作者可能启动并做性能自测试，然后告诉代理它们有多快。代理然后可以选择最快的可用工作者而不是最旧的。

## ZeroMQ 的高级 API {#A-High-Level-API-for-ZeroMQ}

我们将把请求-回复推到栈上并打开一个不同的区域，即 ZeroMQ API 本身。这个绕道有一个原因：当我们编写更复杂的示例时，低级 ZeroMQ API 开始看起来越来越笨拙。看看我们负载均衡代理中工作者线程的核心：

```c
while (true) {
    //  获取一个地址帧和空分隔符
    char *address = s_recv (worker);
    char *empty = s_recv (worker);
    assert (*empty == 0);
    free (empty);

    //  获取请求，发送回复
    char *request = s_recv (worker);
    printf ("工作者：%s\n", request);
    free (request);

    s_sendmore (worker, address);
    s_sendmore (worker, "");
    s_send     (worker, "OK");
    free (address);
}
```

该代码甚至不可重用，因为它只能处理信封中的一个回复地址，它已经对 ZeroMQ API 做了一些包装。如果我们使用 `libzmq` 简单消息 API，这是我们必须编写的：

```c
while (true) {
    //  获取一个地址帧和空分隔符
    char address [255];
    int address_size = zmq_recv (worker, address, 255, 0);
    if (address_size == -1)
        break;

    char empty [1];
    int empty_size = zmq_recv (worker, empty, 1, 0);
    assert (empty_size <= 0);
    if (empty_size == -1)
        break;

    //  获取请求，发送回复
    char request [256];
    int request_size = zmq_recv (worker, request, 255, 0);
    if (request_size == -1)
        return NULL;
    request [request_size] = 0;
    printf ("工作者：%s\n", request);
    
    zmq_send (worker, address, address_size, ZMQ_SNDMORE);
    zmq_send (worker, empty, 0, ZMQ_SNDMORE);
    zmq_send (worker, "OK", 2, 0);
}
```

当代码太长而无法快速编写时，它也太长而无法理解。到目前为止，我坚持使用原生 API，因为作为 ZeroMQ 用户，我们需要深入了解它。但当它妨碍我们时，我们必须将其视为要解决的问题。

我们当然不能简单地更改 ZeroMQ API，这是一个记录的公共合同，数千人同意并依赖。相反，我们基于迄今为止的经验在顶部构建更高级的 API，最具体地说，我们从编写更复杂的请求-回复模式的经验。

我们想要的是一个 API，让我们在一次拍摄中接收和发送整个消息，包括带有任意数量回复地址的回复信封。一个让我们用绝对最少的代码行做我们想要的事情。

制作一个好的消息 API 相当困难。我们有术语问题：ZeroMQ 使用"消息"来描述多部分消息和单个消息帧。我们有期望问题：有时将消息内容视为可打印字符串数据是自然的，有时作为二进制 blob。我们有技术挑战，特别是如果我们想避免过多地复制数据。

制作一个好的 API 的挑战影响所有语言，尽管我的具体用例是 C。无论你使用什么语言，想想你如何为你的语言绑定做贡献，使其尽可能好（或更好），比我将要描述的 C 绑定。

### 高级 API 的功能 {#Features-of-a-Higher-Level-API}

我的解决方案是使用三个相当自然和明显的概念：*字符串*（已经是我们 `s_send` 和 `s_recv` 辅助程序的基础）、*帧*（消息帧）和*消息*（一个或多个帧的列表）。这是工作者代码，重写到使用这些概念的 API 上：

```c
while (true) {
    zmsg_t *msg = zmsg_recv (worker);
    zframe_reset (zmsg_last (msg), "OK", 2);
    zmsg_send (&msg, worker);
}
```

减少我们需要读写复杂消息的代码量很棒：结果易于阅读和理解。让我们继续这个过程，处理与 ZeroMQ 工作的其他方面。这是基于我迄今为止使用 ZeroMQ 的经验，我希望在高级 API 中拥有的愿望清单：

* *套接字的自动处理*。我发现必须手动关闭套接字，并且在某些（但不是全部）情况下必须显式定义延迟超时很麻烦。当我关闭上下文时自动关闭套接字会很棒。

* *可移植线程管理*。每个重要的 ZeroMQ 应用程序都使用线程，但 POSIX 线程不可移植。所以一个体面的高级 API 应该在可移植层下隐藏这一点。

* *从父线程到子线程的管道*。这是一个反复出现的问题：如何在父线程和子线程之间发信号。我们的 API 应该提供 ZeroMQ 消息管道（自动使用 PAIR 套接字和 `inproc`）。

* *可移植时钟*。即使获得毫秒分辨率的时间，或睡眠几毫秒，都不可移植。现实的 ZeroMQ 应用程序需要可移植时钟，所以我们的 API 应该提供它们。

* *替换 `zmq_poll()` 的反应器*。轮询循环很简单，但笨拙。写很多这些，我们最终一遍又一遍地做同样的工作：计算定时器，当套接字准备好时调用代码。带有套接字读取器和定时器的简单反应器将节省大量重复工作。

* *正确处理 Ctrl-C*。我们已经看到如何捕获中断。如果这在所有应用程序中发生会很有用。

### CZMQ 高级 API {#The-CZMQ-High-Level-API}

将这个愿望清单变成 C 语言的现实给我们 [CZMQ](http://czmq.zeromq.org/)，一个用于 C 的 ZeroMQ 语言绑定。这个高级绑定实际上是从示例的早期版本发展而来的。它将用于 ZeroMQ 的更好语义与一些可移植性层结合起来，并且（对 C 重要，但对其他语言较少）容器如哈希和列表。CZMQ 还使用一个优雅的对象模型，导致坦率地可爱的代码。

这是使用高级 API（C 情况下的 CZMQ）重写的负载均衡代理：

```c
//  使用高级 API 的负载均衡代理
#include "czmq.h"

int main (void)
{
    zctx_t *ctx = zctx_new ();
    void *frontend = zsocket_new (ctx, ZMQ_ROUTER);
    void *backend = zsocket_new (ctx, ZMQ_ROUTER);
    zsocket_bind (frontend, "ipc://frontend.ipc");
    zsocket_bind (backend, "ipc://backend.ipc");

    //  工作者队列，每个条目是一个工作者身份
    zlist_t *workers = zlist_new ();

    while (true) {
        zmq_pollitem_t items [] = {
            { backend,  0, ZMQ_POLLIN, 0 },
            { frontend, 0, ZMQ_POLLIN, 0 }
        };
        //  只有工作者可用时才轮询前端
        int rc = zmq_poll (items, zlist_size (workers)? 2: 1, -1);
        if (rc == -1)
            break;              //  中断

        //  处理来自后端的工作者活动
        if (items [0].revents & ZMQ_POLLIN) {
            zmsg_t *msg = zmsg_recv (backend);
            if (!msg)
                break;          //  中断
            zframe_t *identity = zmsg_unwrap (msg);
            zlist_append (workers, identity);

            //  如果这不是 READY，转发给客户端
            zframe_t *frame = zmsg_first (msg);
            if (memcmp (zframe_data (frame), "READY", 5) == 0)
                zmsg_destroy (&msg);
            else
                zmsg_send (&msg, frontend);
        }
        if (items [1].revents & ZMQ_POLLIN) {
            //  获取客户端请求，路由到第一个可用工作者
            zmsg_t *msg = zmsg_recv (frontend);
            if (msg) {
                zmsg_wrap (msg, (zframe_t *) zlist_pop (workers));
                zmsg_send (&msg, backend);
            }
        }
    }
    //  当我们完成时，清理正确
    while (zlist_size (workers)) {
        zframe_t *frame = (zframe_t *) zlist_pop (workers);
        zframe_destroy (&frame);
    }
    zlist_destroy (&workers);
    zctx_destroy (&ctx);
    return 0;
}
```

CZMQ 提供的一件事是干净的中断处理。这意味着 Ctrl-C 将导致任何阻塞 ZeroMQ 调用以返回代码 -1 和 errno 设置为 `EINTR` 退出。高级接收方法在这种情况下将返回 NULL。所以，你可以干净地退出这样的循环：

```c
while (true) {
    zstr_send (client, "Hello");
    char *reply = zstr_recv (client);
    if (!reply)
        break;              //  中断
    printf ("客户端：%s\n", reply);
    free (reply);
    sleep (1);
}
```

或者，如果你调用 `zmq_poll()`，测试返回代码：

```c
if (zmq_poll (items, 2, 1000 * 1000) == -1)
    break;              //  中断
```

前面的示例仍然使用 `zmq_poll()`。那么反应器呢？CZMQ `zloop` 反应器简单但功能强大。它让你：

* 在任何套接字上设置读取器，即每当套接字有输入时调用的代码。
* 取消套接字上的读取器。
* 设置在特定间隔一次或多次触发的定时器。
* 取消定时器。

`zloop` 当然在内部使用 `zmq_poll()`。每次你添加或删除读取器时，它重建其轮询集，它计算轮询超时以匹配下一个定时器。然后，它为每个需要注意的套接字和定时器调用读取器和定时器处理程序。

当我们使用反应器模式时，我们的代码翻转过来。主逻辑看起来像这样：

```c
zloop_t *reactor = zloop_new ();
zloop_reader (reactor, self->backend, s_handle_backend, self);
zloop_start (reactor);
zloop_destroy (&reactor);
```
* 接收工作者返回的回复。
* 将这些回复发送回原始请求的客户端。

代理的代码相当长，但值得理解。

这个程序最难的部分是（a）每个套接字读写的信封（envelope），以及（b）负载均衡算法。我们将依次讲解，从消息信封格式开始。

让我们逐步走一遍从客户端到工作者再返回的完整请求-回复链。在这段代码中，我们设置了客户端和工作者套接字的身份，以便更容易追踪消息帧。实际上，我们会让 ROUTER 套接字为连接生成身份。假设客户端的身份是 "CLIENT"，工作者的身份是 "WORKER"。客户端应用发送一个包含 "Hello" 的单帧消息。

由于 REQ 套接字会添加一个空分隔帧，ROUTER 套接字会添加连接身份，代理会从前端 ROUTER 套接字读取到客户端地址、空分隔帧和数据部分。

代理将此消息发送给工作者，前面加上所选工作者的地址，并额外加一个空部分以满足另一端的 REQ。

这个复杂的信封堆栈首先被后端 ROUTER 套接字处理，移除第一帧。然后工作者中的 REQ 套接字移除空部分，并将剩余部分提供给工作者应用。

工作者必须保存信封（即直到并包括空消息帧的所有部分），然后它可以对数据部分执行所需的操作。请注意，REP 套接字会自动执行此操作，但我们使用 REQ-ROUTER 模式，以便我们可以获得适当的负载均衡。

在返回路径上，消息与进入时相同，即后端套接字给代理一个五部分消息，代理向前端套接字发送一个三部分消息，客户端得到一个一部分消息。

现在让我们看看负载均衡算法。它要求客户端和工作者都使用 REQ 套接字，并且工作者正确存储和重放它们收到的消息上的信封。算法是：

* 创建一个轮询集，总是轮询后端，只有在有一个或多个工作者可用时才轮询前端。

* 以无限超时轮询活动。

* 如果后端有活动，我们要么有"准备就绪"消息，要么有客户端的回复。在任何一种情况下，我们将工作者地址（第一部分）存储在我们的工作者队列中，如果其余部分是客户端回复，我们通过前端将其发送回该客户端。

* 如果前端有活动，我们接受客户端请求，弹出下一个工作者（即最后使用的），并将请求发送到后端。这意味着发送工作者地址、空部分，然后是客户端请求的三个部分。

你现在应该看到，你可以基于工作者在其初始"准备就绪"消息中提供的信息，使用负载均衡算法的变体来重用和扩展。例如，工作者可能启动并进行性能自测，然后告诉代理他们有多快。然后代理可以选择最快的可用工作者，而不是最旧的。

## ZeroMQ 的高级 API {#A-High-Level-API-for-ZeroMQ}

我们将把请求-回复推到栈上并打开一个不同的区域，即 ZeroMQ API 本身。这个绕道有一个原因：当我们编写更复杂的示例时，低级 ZeroMQ API 开始看起来越来越笨拙。看看我们负载均衡代理中工作者线程的核心：

```c
while (true) {
    //  获取一个地址帧和空分隔符
    char *address = s_recv (worker);
    char *empty = s_recv (worker);
    assert (*empty == 0);
    free (empty);

    //  获取请求，发送回复
    char *request = s_recv (worker);
    printf ("工作者：%s\n", request);
    free (request);

    s_sendmore (worker, address);
    s_sendmore (worker, "");
    s_send     (worker, "OK");
    free (address);
}
```

该代码甚至不可重用，因为它只能处理信封中的一个回复地址，它已经对 ZeroMQ API 做了一些包装。如果我们使用 `libzmq` 简单消息 API，这是我们必须编写的：

```c
while (true) {
    //  获取一个地址帧和空分隔符
    char address [255];
    int address_size = zmq_recv (worker, address, 255, 0);
    if (address_size == -1)
        break;

    char empty [1];
    int empty_size = zmq_recv (worker, empty, 1, 0);
    assert (empty_size <= 0);
    if (empty_size == -1)
        break;

    //  获取请求，发送回复
    char request [256];
    int request_size = zmq_recv (worker, request, 255, 0);
    if (request_size == -1)
        return NULL;
    request [request_size] = 0;
    printf ("工作者：%s\n", request);
    
    zmq_send (worker, address, address_size, ZMQ_SNDMORE);
    zmq_send (worker, empty, 0, ZMQ_SNDMORE);
    zmq_send (worker, "OK", 2, 0);
}
```

当代码太长而无法快速编写时，它也太长而无法理解。到目前为止，我坚持使用原生 API，因为作为 ZeroMQ 用户，我们需要深入了解它。但当它妨碍我们时，我们必须将其视为要解决的问题。

我们当然不能简单地更改 ZeroMQ API，这是一个记录的公共合同，数千人同意并依赖。相反，我们基于迄今为止的经验在顶部构建更高级的 API，最具体地说，我们从编写更复杂的请求-回复模式的经验。

我们想要的是一个 API，让我们在一次拍摄中接收和发送整个消息，包括带有任意数量回复地址的回复信封。一个让我们用绝对最少的代码行做我们想要的事情。

制作一个好的消息 API 相当困难。我们有术语问题：ZeroMQ 使用"消息"来描述多部分消息和单个消息帧。我们有期望问题：有时将消息内容视为可打印字符串数据是自然的，有时作为二进制 blob。我们有技术挑战，特别是如果我们想避免过多地复制数据。

制作一个好的 API 的挑战影响所有语言，尽管我的具体用例是 C。无论你使用什么语言，想想你如何为你的语言绑定做贡献，使其尽可能好（或更好），比我将要描述的 C 绑定。

### 高级 API 的功能 {#Features-of-a-Higher-Level-API}

我的解决方案是使用三个相当自然和明显的概念：*字符串*（已经是我们 `s_send` 和 `s_recv` 辅助程序的基础）、*帧*（消息帧）和*消息*（一个或多个帧的列表）。这是工作者代码，重写到使用这些概念的 API 上：

```c
while (true) {
    zmsg_t *msg = zmsg_recv (worker);
    zframe_reset (zmsg_last (msg), "OK", 2);
    zmsg_send (&msg, worker);
}
```

减少我们需要读写复杂消息的代码量很棒：结果易于阅读和理解。让我们继续这个过程，处理与 ZeroMQ 工作的其他方面。这是基于我迄今为止使用 ZeroMQ 的经验，我希望在高级 API 中拥有的愿望清单：

* *套接字的自动处理*。我发现必须手动关闭套接字，并且在某些（但不是全部）情况下必须显式定义延迟超时很麻烦。当我关闭上下文时自动关闭套接字会很棒。

* *可移植线程管理*。每个重要的 ZeroMQ 应用程序都使用线程，但 POSIX 线程不可移植。所以一个体面的高级 API 应该在可移植层下隐藏这一点。

* *从父线程到子线程的管道*。这是一个反复出现的问题：如何在父线程和子线程之间发信号。我们的 API 应该提供 ZeroMQ 消息管道（自动使用 PAIR 套接字和 `inproc`）。

* *可移植时钟*。即使获得毫秒分辨率的时间，或睡眠几毫秒，都不可移植。现实的 ZeroMQ 应用程序需要可移植时钟，所以我们的 API 应该提供它们。

* *替换 `zmq_poll()` 的反应器*。轮询循环很简单，但笨拙。写很多这些，我们最终一遍又一遍地做同样的工作：计算定时器，当套接字准备好时调用代码。带有套接字读取器和定时器的简单反应器将节省大量重复工作。

* *正确处理 Ctrl-C*。我们已经看到如何捕获中断。如果这在所有应用程序中发生会很有用。

### CZMQ 高级 API {#The-CZMQ-High-Level-API}

将这个愿望清单变成 C 语言的现实给我们 [CZMQ](http://czmq.zeromq.org/)，一个用于 C 的 ZeroMQ 语言绑定。这个高级绑定实际上是从示例的早期版本发展而来的。它将用于 ZeroMQ 的更好语义与一些可移植性层结合起来，并且（对 C 重要，但对其他语言较少）容器如哈希和列表。CZMQ 还使用一个优雅的对象模型，导致坦率地可爱的代码。

这是使用高级 API（C 情况下的 CZMQ）重写的负载均衡代理：

```c
//  使用高级 API 的负载均衡代理
#include "czmq.h"

int main (void)
{
    zctx_t *ctx = zctx_new ();
    void *frontend = zsocket_new (ctx, ZMQ_ROUTER);
    void *backend = zsocket_new (ctx, ZMQ_ROUTER);
    zsocket_bind (frontend, "ipc://frontend.ipc");
    zsocket_bind (backend, "ipc://backend.ipc");

    //  工作者队列，每个条目是一个工作者身份
    zlist_t *workers = zlist_new ();

    while (true) {
        zmq_pollitem_t items [] = {
            { backend,  0, ZMQ_POLLIN, 0 },
            { frontend, 0, ZMQ_POLLIN, 0 }
        };
        //  只有工作者可用时才轮询前端
        int rc = zmq_poll (items, zlist_size (workers)? 2: 1, -1);
        if (rc == -1)
            break;              //  中断

        //  处理来自后端的工作者活动
        if (items [0].revents & ZMQ_POLLIN) {
            zmsg_t *msg = zmsg_recv (backend);
            if (!msg)
                break;          //  中断
            zframe_t *identity = zmsg_unwrap (msg);
            zlist_append (workers, identity);

            //  如果这不是 READY，转发给客户端
            zframe_t *frame = zmsg_first (msg);
            if (memcmp (zframe_data (frame), "READY", 5) == 0)
                zmsg_destroy (&msg);
            else
                zmsg_send (&msg, frontend);
        }
        if (items [1].revents & ZMQ_POLLIN) {
            //  获取客户端请求，路由到第一个可用工作者
            zmsg_t *msg = zmsg_recv (frontend);
            if (msg) {
                zmsg_wrap (msg, (zframe_t *) zlist_pop (workers));
                zmsg_send (&msg, backend);
            }
        }
    }
    //  当我们完成时，清理正确
    while (zlist_size (workers)) {
        zframe_t *frame = (zframe_t *) zlist_pop (workers);
        zframe_destroy (&frame);
    }
    zlist_destroy (&workers);
    zctx_destroy (&ctx);
    return 0;
}
```

CZMQ 提供的一件事是干净的中断处理。这意味着 Ctrl-C 将导致任何阻塞 ZeroMQ 调用以返回代码 -1 和 errno 设置为 `EINTR` 退出。高级接收方法在这种情况下将返回 NULL。所以，你可以干净地退出这样的循环：

```c
while (true) {
    zstr_send (client, "Hello");
    char *reply = zstr_recv (client);
    if (!reply)
        break;              //  中断
    printf ("客户端：%s\n", reply);
    free (reply);
    sleep (1);
}
```

或者，如果你调用 `zmq_poll()`，测试返回代码：

```c
if (zmq_poll (items, 2, 1000 * 1000) == -1)
    break;              //  中断
```

前面的示例仍然使用 `zmq_poll()`。那么反应器呢？CZMQ `zloop` 反应器简单但功能强大。它让你：

* 在任何套接字上设置读取器，即每当套接字有输入时调用的代码。
* 取消套接字上的读取器。
* 设置在特定间隔一次或多次触发的定时器。
* 取消定时器。

`zloop` 当然在内部使用 `zmq_poll()`。每次你添加或删除读取器时，它重建其轮询集，它计算轮询超时以匹配下一个定时器。然后，它为每个需要注意的套接字和定时器调用读取器和定时器处理程序。

当我们使用反应器模式时，我们的代码翻转过来。主逻辑看起来像这样：

```c
zloop_t *reactor = zloop_new ();
zloop_reader (reactor, self->backend, s_handle_backend, self);
zloop_start (reactor);
zloop_destroy (&reactor);
```

消息的实际处理位于专用函数或方法内部。你可能不喜欢这种风格 —— 这是品味问题。它确实有助于混合定时器和套接字活动。在本文的其余部分中，我们将在更简单的情况下使用 `zmq_poll()`，在更复杂的示例中使用 `zloop`。

这是再次重写的负载均衡代理，这次使用 `zloop`：

```c
//  使用 zloop 的负载均衡代理
#include "czmq.h"

//  反应器设计，高级 API，多线程
typedef struct {
    zctx_t *ctx;                //  我们的 ZeroMQ 上下文
    void *frontend;             //  面向客户端的套接字
    void *backend;              //  面向工作者的套接字
    zlist_t *workers;           //  可用工作者列表
} lbbroker_t;

//  在反应器中处理来自后端的输入
static int s_handle_backend (zloop_t *loop, zmq_pollitem_t *poller, void *arg)
{
    //  使用 arg 指针获取上下文
    lbbroker_t *self = (lbbroker_t *) arg;
    
    //  处理工作者活动
    zmsg_t *msg = zmsg_recv (self->backend);
    if (!msg)
        return -1;          //  中断

    //  使用工作者身份进行负载均衡
    zframe_t *identity = zmsg_unwrap (msg);
    zlist_append (self->workers, identity);

    //  如果不是 READY，转发给客户端
    zframe_t *frame = zmsg_first (msg);
    if (memcmp (zframe_data (frame), "READY", 5) == 0)
        zmsg_destroy (&msg);
    else
        zmsg_send (&msg, self->frontend);

    //  启动轮询前端如果有工作者可用
    if (zlist_size (self->workers) == 1) {
        zmq_pollitem_t poller = { self->frontend, 0, ZMQ_POLLIN };
        zloop_poller (loop, &poller, s_handle_frontend, self);
    }
    return 0;
}

//  在反应器中处理来自前端的输入
static int s_handle_frontend (zloop_t *loop, zmq_pollitem_t *poller, void *arg)
{
    //  使用 arg 指针获取上下文
    lbbroker_t *self = (lbbroker_t *) arg;

    //  获取客户端请求，路由到第一个可用工作者
    zmsg_t *msg = zmsg_recv (self->frontend);
    if (msg) {
        zmsg_wrap (msg, (zframe_t *) zlist_pop (self->workers));
        zmsg_send (&msg, self->backend);
    }

    //  如果没有更多工作者，停止轮询前端
    if (zlist_size (self->workers) == 0) {
        zmq_pollitem_t poller = { self->frontend, 0, ZMQ_POLLIN };
        zloop_poller_end (loop, &poller);
    }
    return 0;
}

int main (void)
{
    lbbroker_t *self = (lbbroker_t *) zmalloc (sizeof (lbbroker_t));
    self->ctx = zctx_new ();
    self->frontend = zsocket_new (self->ctx, ZMQ_ROUTER);
    self->backend = zsocket_new (self->ctx, ZMQ_ROUTER);
    zsocket_bind (self->frontend, "ipc://frontend.ipc");
    zsocket_bind (self->backend, "ipc://backend.ipc");

    self->workers = zlist_new ();

    //  从后端套接字开始轮询，只从那里
    zloop_t *reactor = zloop_new ();
    zmq_pollitem_t poller = { self->backend, 0, ZMQ_POLLIN };
    zloop_poller (reactor, &poller, s_handle_backend, self);
    zloop_start (reactor);
    zloop_destroy (&reactor);

    //  当我们完成时，清理正确
    while (zlist_size (self->workers)) {
        zframe_t *frame = (zframe_t *) zlist_pop (self->workers);
        zframe_destroy (&frame);
    }
    zlist_destroy (&self->workers);
    zctx_destroy (&self->ctx);
    free (self);
    return 0;
}
```

当你发送 Ctrl-C 时，让应用程序正确关闭可能很棘手。如果你使用 `zctx` 类，它会自动设置信号处理，但你的代码仍然必须配合。如果 `zmq_poll` 返回 -1 或如果任何 `zstr_recv`、`zframe_recv` 或 `zmsg_recv` 方法返回 NULL，你必须中断任何循环。如果你有嵌套循环，使外层条件于 `!zctx_interrupted` 是有用的。

如果你使用子线程，它们不会收到中断。要告诉它们关闭，你可以：

* 销毁上下文，如果它们共享相同的上下文，在这种情况下它们正在等待的任何阻塞调用将以 ETERM 结束。
* 向它们发送关闭消息，如果它们使用自己的上下文。为此你需要一些套接字管道。

## 异步客户端/服务器模式 {#The-Asynchronous-Client-Server-Pattern}

在 ROUTER 到 DEALER 示例中，我们看到了一个 1-to-N 用例，其中一个服务器异步地与多个工作者通话。我们可以将其颠倒过来得到一个非常有用的 N-to-1 架构，其中各种客户端与单个服务器通话，并且异步地执行此操作。

```
#----------#   #----------#
|  Client  |   |  Client  |
+----------+   +----------+
|  DEALER  |   |  DEALER  |
'----------'   '----------'
      ^              ^
      |              |
      '------+-------'
             |
             v
      .-------------.
      |   ROUTER    |
      +-------------+
      |   Server    |
      #-------------#
```

这是它的工作原理：

* 客户端连接到服务器并发送请求。
* 对于每个请求，服务器发送 0 个或更多回复。
* 客户端可以发送多个请求而不等待回复。
* 服务器可以发送多个回复而不等待新请求。

这是显示此工作原理的代码：

```c
//  异步客户端/服务器
//  在单个进程中运行一个服务器和多个工作者
//  注意如何在不等待回复的情况下发送和接收消息

#include "czmq.h"

//  这是我们的客户端任务
//  它连接到服务器并向其发送请求
static void *
client_task (void *args)
{
    zctx_t *ctx = zctx_new ();
    void *client = zsocket_new (ctx, ZMQ_DEALER);

    //  设置身份，以便我们可以跟踪响应
    char identity [10];
    sprintf (identity, "%04X-%04X", randof (0x10000), randof (0x10000));
    zmq_setsockopt (client, ZMQ_IDENTITY, identity, strlen (identity));
    zsocket_connect (client, "tcp://localhost:5570");

    zmq_pollitem_t items [] = { { client, 0, ZMQ_POLLIN, 0 } };
    int request_nbr = 0;
    while (true) {
        //  每 100 毫秒滴答一下，直到被中断
        int centitick = zmq_poll (items, 1, 100 * ZMQ_POLL_MSEC);
        if (centitick == -1)
            break;              //  中断

        if (items [0].revents & ZMQ_POLLIN) {
            char *reply = zstr_recv (client);
            if (!reply)
                break;              //  中断
            printf ("客户端：%s\n", reply);
            free (reply);
        }
        //  每秒发送一个随机请求
        if (++request_nbr % 100 == 0) {
            char request [10];
            sprintf (request, "%03d", randof (1000));
            zstr_send (client, request);
        }
    }
    zctx_destroy (&ctx);
    return NULL;
}

//  这是我们的服务器任务
//  它使用工作者线程池来处理请求
static void *
server_worker (void *args)
{
    zctx_t *ctx = zctx_new ();
    void *worker = zsocket_new (ctx, ZMQ_DEALER);
    zsocket_connect (worker, "inproc://backend");

    while (true) {
        //  DEALER 套接字给我们回复信封和内容
        zmsg_t *msg = zmsg_recv (worker);
        if (!msg)
            break;              //  中断

        //  发送 0-4 个回复
        int reply, replies = randof (5);
        for (reply = 0; reply < replies; reply++) {
            //  睡眠几毫秒
            zclock_sleep (randof (1000) + 1);
            zmsg_send (&msg, worker);
        }
        zmsg_destroy (&msg);
    }
    zctx_destroy (&ctx);
    return NULL;
}

//  服务器任务运行一个小型负载均衡代理
static void *
server_task (void *args)
{
    //  前端套接字与客户端通话，后端与工作者通话
    zctx_t *ctx = zctx_new ();
    void *frontend = zsocket_new (ctx, ZMQ_ROUTER);
    void *backend = zsocket_new (ctx, ZMQ_DEALER);
    zsocket_bind (frontend, "tcp://*:5570");
    zsocket_bind (backend, "inproc://backend");

    //  启动池中的工作者线程，精确数量不是关键
    int thread_nbr;
    for (thread_nbr = 0; thread_nbr < 5; thread_nbr++) {
        pthread_t worker;
        pthread_create (&worker, NULL, server_worker, NULL);
    }
    //  连接后端到前端 via 代理
    zmq_proxy (frontend, backend, NULL);

    zctx_destroy (&ctx);
    return NULL;
}

//  应用程序在单个进程中运行服务器和客户端线程
//  看服务器如何接收请求并发送回复

int main (void)
{
    pthread_t client1, client2, client3, server;
    pthread_create (&client1, NULL, client_task, NULL);
    pthread_create (&client2, NULL, client_task, NULL);
    pthread_create (&client3, NULL, client_task, NULL);
    pthread_create (&server, NULL, server_task, NULL);
    sleep (5);                  //  运行 5 秒然后退出
    zctx_interrupted = 1;
    return 0;
}
```

示例在一个进程中运行，使用多个线程模拟真实的多进程架构。当你运行示例时，你会看到三个客户端（每个具有随机 ID），打印出它们从服务器获得的回复。仔细看，你会看到每个客户端任务每个请求获得 0 个或更多回复。

对此代码的一些评论：

* 客户端每秒发送一个请求，并获得零个或更多回复。为了使用 `zmq_poll()` 工作，我们不能简单地以 1 秒超时轮询，否则我们将只在收到最后一个回复后一秒发送新请求。所以我们以高频率轮询（100 次，每次轮询 1/100 秒），这大致是准确的。

* 服务器使用工作者线程池，每个同步处理一个请求。它使用内部队列将这些连接到其前端套接字。它使用 `zmq_proxy()` 调用连接前端和后端套接字。

```
   #---------#   #---------#   #---------#
   | Client  |   | Client  |   | Client  |
   +---------+   +---------+   +---------+
   | DEALER  |   | DEALER  |   | DEALER  |
   '---------'   '---------'   '---------'
     connect       connect       connect
        |             |             |
        '-------------+-------------'
                      |
.-------------------- | --------------------.
:                     v                     :
:                   bind                    :
:               .-----------.               :
:               |  ROUTER   |               :
:               +-----------+               :
:               |  Server   |               :
:               +-----------+               :
:               |  DEALER   |               :
:               '-----------'               :
:                   bind                    :
:                     |                     :
:       .-------------+-------------.       :
:       |             |             |       :
:       v             v             v       :
:    connect       connect       connect    :
:  .---------.   .---------.   .---------.  :
:  | DEALER  |   | DEALER  |   | DEALER  |  :
:  +---------+   +---------+   +---------+  :
:  | Worker  |   | Worker  |   | Worker  |  :
:  #---------#   #---------#   #---------#  :
'-------------------------------------------'
```

注意我们在客户端和服务器之间进行 DEALER 到 ROUTER 对话，但在服务器主线程和工作者之间内部，我们进行 DEALER 到 DEALER。如果工作者严格同步，我们会使用 REP。但是，因为我们想发送多个回复，我们需要一个异步套接字。我们不想路由回复，它们总是去向发送给我们请求的单个服务器线程。

让我们考虑路由信封。客户端发送由单个帧组成的消息。服务器线程接收两帧消息（以客户端身份为前缀的原始消息）。我们将这两帧发送给工作者，它将其视为正常的回复信封，将其作为两帧消息返回给我们。然后我们使用第一帧作为身份将第二帧作为回复路由回客户端。

它看起来像这样：

```
     client          server       frontend       worker
   [ DEALER ]<---->[ ROUTER <----> DEALER <----> DEALER ]
             1 part         2 parts       2 parts
```

现在对于套接字：我们可以使用负载均衡 ROUTER 到 DEALER 模式与工作者通话，但这是额外的工作。在这种情况下，DEALER 到 DEALER 模式可能很好：权衡是每个请求的较低延迟，但不平衡工作分配的较高风险。在这种情况下，简单性获胜。

当你构建与客户端保持有状态对话的服务器时，你将遇到一个经典问题。如果服务器为每个客户端保持一些状态，并且客户端不断来去，最终它将耗尽资源。即使相同的客户端保持连接，如果你使用默认身份，每个连接都会看起来像一个新的。

我们在上面的示例中通过仅在很短的时间内保持状态（工作者处理请求所需的时间）然后丢弃状态来作弊。但这对于许多情况不实用。要在有状态异步服务器中正确管理客户端状态，你必须：

* 从客户端到服务器做心跳。在我们的示例中，我们每秒发送一个请求，可以可靠地用作心跳。

* 使用客户端身份（无论是生成的还是显式的）作为键存储状态。

* 检测停止的心跳。如果在，比如说，两秒内没有来自客户端的请求，服务器可以检测到这一点并销毁它为该客户端持有的任何状态。

## 工作示例：代理间路由 {#Worked-Example-Inter-Broker-Routing}

让我们把到目前为止看到的一切结合起来，将事情扩展到一个真实的应用程序。我们将逐步构建这个，经过几次迭代。我们最好的客户紧急呼叫我们，要求设计一个大型云计算设施。他对跨越许多数据中心的云有这样的愿景，每个数据中心都是客户端和工作者的集群，并且作为一个整体一起工作。因为我们足够聪明，知道实践总是胜过理论，我们建议使用 ZeroMQ 制作一个工作模拟。我们的客户渴望在老板改变主意之前锁定预算，并且在 Twitter 上读到了关于 ZeroMQ 的好东西，同意了。

### 建立细节 {#Establishing-the-Details}

几杯咖啡后，我们想跳入编写代码，但一个小声音告诉我们在对完全错误的问题制作轰动解决方案之前获得更多细节。"云在做什么样的工作？"，我们问。

客户解释：

* 工作者运行在各种硬件上，但它们都能够处理任何任务。每个集群有几百个工作者，总共多达十几个集群。

* 客户端为工作者创建任务。每个任务是独立的工作单元，客户端想要的只是找到可用的工作者，并尽快将任务发送给它。将有很多客户端，它们会任意来去。

* 真正的困难是能够随时添加和删除集群。集群可以立即离开或加入云，带来所有工作者和客户端。

* 如果在自己的集群中没有工作者，客户端的任务将去往云中其他可用的工作者。

* 客户端一次发送一个任务，等待回复。如果他们在 X 秒内没有得到答案，他们只会再次发送任务。这不是我们的关注点；客户端 API 已经这样做了。

* 工作者一次处理一个任务；它们是非常简单的野兽。如果它们崩溃，它们会被启动它们的任何脚本重新启动。

所以我们仔细检查以确保我们正确理解了这一点：

* "集群之间会有某种超级网络互连，对吧？"，我们问。客户说，"是的，当然，我们不是白痴。"

* "我们谈论的是什么样的体积？"，我们问。客户回答，"每个集群多达一千个客户端，每个每秒最多做十个请求。请求很小，回复也很小，每个不超过 1K 字节。"

所以我们做一点计算，看到这在普通 TCP 上工作得很好。2,500 个客户端 x 10/秒 x 1,000 字节 x 2 个方向 = 50MB/秒或 400Mb/秒，对于 1Gb 网络不是问题。

这是一个直接的问题，不需要异域硬件或协议，只需要一些聪明的路由算法和仔细的设计。我们从设计一个集群（一个数据中心）开始，然后我们弄清楚如何将集群连接在一起。

### 单个集群的架构 {#Architecture-of-a-Single-Cluster}

工作者和客户端是同步的。我们想使用负载均衡模式将任务路由给工作者。工作者都是相同的；我们的设施没有不同服务的概念。工作者是匿名的；客户端从不直接寻址它们。我们在这里不尝试提供保证交付、重试等。

出于我们已经检查的原因，客户端和工作者不会直接彼此交谈。这使得动态添加或删除节点变得不可能。所以我们的基本模型由我们之前看到的请求-回复消息代理组成。

```
#--------#  #--------#  #--------#
| Client |  | Client |  | Client |
+--------+  +--------+  +--------+
|  REQ   |  |  REQ   |  |  REQ   |
'---+----'  '---+----'  '---+----'
    |           |           |
    '-----------+-----------'
                |
          .-----+------.
          |   ROUTER   |
          +------------+
          |    Load    |
          |  balancer  |  Broker
          +------------+
          |   ROUTER   |
          '-----+------'
                |
    .-----------+-----------.
    |           |           |
.---+----.  .---+----.  .---+----.
|  REQ   |  |  REQ   |  |  REQ   |
+--------+  +--------+  +--------+
| Worker |  | Worker |  | Worker |
#--------#  #--------#  #--------#
```

### 扩展到多个集群 {#Scaling-to-Multiple-Clusters}

现在我们将其扩展到多个集群。每个集群都有一组客户端和工作者，以及一个将这些连接在一起的代理。

```
     Cluster 1          :          Cluster 2
                        :
.---.  .---.  .---.     :     .---.  .---.  .---.
| C |  | C |  | C |     :     | C |  | C |  | C |
'-+-'  '-+-'  '-+-'     :     '-+-'  '-+-'  #-+-'
  |      |      |       :       |      |      |
  |      |      |       :       |      |      |
#-+------+------+-#     :     #-+------+------+-#
|     Broker      |     :     |     Broker      |
#-+------+------+-#     :     #-+------+------+-#
  |      |      |       :       |      |      |
  |      |      |       :       |      |      |
.-+-.  .-+-.  .-+-.     :     .-+-.  .-+-.  .-+-.
| W |  | W |  | W |     :     | W |  | W |  | W |
'---'  '---'  '---'     :     '---'  '---'  '---'
```
