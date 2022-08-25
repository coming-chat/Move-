# Move 语言中文白皮书
## 翻译自： [Move language whiter paper](https://github.com/coming-chat/Move-white-paper/blob/main/Move%202020-05-26.pdf)

## Move: 一种使用可编程**resources**的语言

Sam Blackshear, Evan Cheng, David L. Dill, Victor Gao, Ben Maurer, Todd Nowacki, Alistair Pott, Shaz Qadeer, Rain, Dario Russi, Stephane Sezer, Tim Zakian, Runtian Zhou * 合著

读者须知：本报告在协会发布白皮书 v2.0 之前发布，其中包括对 Libra 支付系统的多项关键更新。已删除过时的链接，但除此之外，本报告尚未修改以包含更新，应在该上下文中阅读。


## 摘要
我们为 Libra 区块链[1][2] 提供了一种安全且灵活的编程语言，它的名字叫做Move。Move也是一种可执行的字节码语言，用于实现自定义交易和智能合约。

Move 的关键特性是能够使用受线性逻辑启发的语义定义自定义**resources**类型 [3]：**resources**永远不能被复制或隐式丢弃，只能在程序存储位置之间移动。这些安全保证由 Move 的类型系统静态强制执行。虽然有这些特殊保护，但不会局限resource这一类型的功能，它可以像传统的类型一样，可以存储在数据结构中，也可以作为参数传递。

**First-class resources**是一个非常笼统的概念，程序员不仅可以使用它来实现安全的数字资产，还可以编写正确的业务逻辑来包装资产以及对数值不同权限的访问和操作。
Move 的安全性和表现力使我们能够在 Move 中实现 Libra 协议的重要部分，包括 Libra 币、交易处理和验证者管理。

## 1.介绍

互联网和移动宽带的出现连接了全球数十亿人，提供了获取知识、免费通信和广泛的低成本、更便捷的服务。
这种连通性也使更多人能够访问金融生态系统。然而，尽管取得了这些进展，但对于最需要金融服务的人来说，获得金融服务的机会仍然有限。

Libra的使命是改变这种状况 [1]。 在本文中，我们介绍了 Move，这是一种新的编程语言，用于在 Libra 协议中实现自定义交易逻辑和智能合约 [2]。为了介绍move，我们分以下例举的几个章节来说明：

1. 描述在区块链上表示数字资产的挑战（第 2 节）。
2. 解释我们的 Move 设计如何应对这些挑战（第 3 节）。
3. 对 Move 的关键特性和编程模型（第 4 节）给出一个面向示例的概述。
4. 深入研究语言和虚拟机设计的技术细节（第 5 节、第 6 节和附录 A）。
5. 最后总结我们在 Move 上取得的进展，描述我们的语言发展计划，并概述我们在 Libra 区块链上支持第三方 Move 代码的路线图（第 7 节）。

本文适用于两种不同的读者：
- 可能不熟悉区块链系统的编程语言研究人员。 我们鼓励这些读者从头到尾阅读这篇论文，但我们警告说，我们有时可能会在没有为不熟悉的读者提供足够背景的情况下提及区块链概念。 在深入研究本文之前阅读 [2] 会有所帮助，但这不是必需的。
- 可能不熟悉编程语言研究但有兴趣了解 Move 语言的区块链开发人员。 我们鼓励这些读者从第 3 节开始。我们警告第 5 节、第 6 节和附录 A 包含一些可能不熟悉的编程语言术语和形式化。

## 2. 在区块链上管理数字资产
我们将首先在抽象层面简要解释区块链，以帮助读者理解像 Move 这样的“区块链编程语言”所扮演的角色。 此讨论有意省略了区块链系统的许多重要细节，以便专注于从语言角度来看相关的功能。

### 2.1. 区块链 概要
区块链是一个复制的状态机 [4][5]。 系统中的复制器称为验证节点。 系统用户将交易发送给验证者。 每个验证节点都了解如何执行交易以将其内部状态机从当前状态转换为新状态。
验证节点利用他们对交易执行的共同理解来遵循共识协议来共同定义和维护复制状态。
- 验证节点从相同的初始状态开始。
- 验证节点选择执行相同的交易顺序。
- 执行交易产生确定性的状态转换。

验证阶节点执行以上步骤后，会就下一个状态达成一致。 重复应用此方案允许验证节点在继续就当前状态达成一致的同时处理交易。

请注意，共识协议和状态转换组件对彼此的实现细节不敏感。 只要共识协议确保交易之间的总顺序并且状态转换方案是确定性的，组件就可以和谐地交互。


