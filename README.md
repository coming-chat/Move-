# Move 语言中文白皮书
## 翻译自： [Move language whiter paper](https://github.com/coming-chat/Move-white-paper/blob/main/Move%202020-05-26.pdf)

## Move: 一种使用可编程**resources**的语言

Sam Blackshear, Evan Cheng, David L. Dill, Victor Gao, Ben Maurer, Todd Nowacki, Alistair Pott, Shaz Qadeer, Rain, Dario Russi, Stephane Sezer, Tim Zakian, Runtian Zhou * 合著

译者：GG@ComingChat

读者须知：本报告在协会发布白皮书 v2.0 之前发布，其中包括对 Libra 支付系统的多项关键更新。已删除过时的链接，但除此之外，本报告尚未修改以包含更新，应在该上下文中阅读。


## 摘要
我们为 Libra 区块链[1][2] 提供了一种安全且灵活的编程语言，它的名字叫做Move。Move也是一种可执行的字节码语言，用于实现自定义交易和智能合约。

Move 的关键特性是能够使用受线性逻辑启发的语义定义自定义**resources**类型 [3]：***resources***永远不能被复制或隐式丢弃，只能在程序存储位置之间移动。这些安全保证由 Move 的类型系统静态强制执行。虽然有这些特殊保护，但不会局限***resources***这一类型的功能，它可以像传统的类型一样，可以存储在数据结构中，也可以作为参数传递。

***First-class resources***是一个非常笼统的概念，程序员不仅可以使用它来实现安全的数字资产，还可以编写正确的业务逻辑来包装资产以及对数值不同权限的访问和操作。
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

### 2.2. 在一个开放的系统中进行数字资产编码

像 Move 这样的区块链编程语言的作用是决定如何表示转换和状态。为了支持丰富的金融基础设施，Libra 区块链的状态必须能够在给定的时间点对数字资产的所有者进行编码。此外，状态转换应该允许资产转移。
区块链编程语言的设计还必须考虑另一个考虑因素。与其他公有链一样，Libra 区块链是一个开放系统。 任何人都可以查看当前的区块链状态或向验证节点提交交易（即提议状态转换）。传统web2行业，用于管理数字资产的软件（例如银行软件）在具有特殊管理控制的封闭系统中运行。 在公有链中，所有参与者都是平等的。 参与者可以提出她喜欢的任何状态转换，但系统并不应该允许所有状态转换。 例如，Alice 可以自由提议转移 Bob 拥有的资产的状态转换。 状态转换函数必须能够识别这个状态转换是无效的并且拒绝它。

在开放软件系统中选择编码数字资产所有权的转换和状态表示具有挑战性。 特别是，实物资产有两个属性难以在数字资产中编码：
- 稀缺性：控制系统中的资产供应， 禁止复制现有资产，创建新资产为特权操作。
- 访问控制：系统中的参与者应该能够通过访问控制策略保护她的资产。

为了直观表现，我们将在一系列关于状态转换表示的 strawman 提案 实例中看到这些问题是如何出现的。我们将假设一个区块链跟踪称为 StrawCoin 的单一数字资产。区块链状态 G 被构造为一个键值存储，它将用户身份（用加密公钥表示）映射到对每个用户持有的 StrawCoin 进行编码的自然数值。提案由一个交易脚本组成，该脚本将使用给定的评估规则进行评估，生成一个更新以应用于全局状态。我们将写 G[𝐾] := 𝑛 来表示用值 𝑛 更新存储在全局区块链状态中键 𝐾 处的自然数。

每个提案的目标是设计一个系统，该系统具有足够的表达能力，允许 Alice 将 StrawCoin 发送给 Bob，但又受到足够的约束，以防止任何用户违反稀缺性或访问控制属性。 这些提案并没有试图解决重放攻击等安全问题 [6]，这些问题很重要，但与我们关于稀缺性和访问控制的讨论无关。

Scarcity. 最简单的提案是直接在交易脚本中对状态的更新进行编码：

   |交易脚本格式      |         评估规则 |
   |:-------:       | :------------:  |
   |<K,n>           |      G[K] := n  |
   
这种表示可以编码从 Alice 向 Bob 发送 StrawCoin。 但它有几个严重的问题。 一方面，这个提议并没有强制StrawCoin的Scarcity。 通过发送交易⟨Alice, 100⟩，Alice 可以“凭空”给自己尽可能多的 StrawCoin。 因此，Alice 发送给Bob的StrawCoin实际上毫无价值，因为Bob可以很容易地为自己创造这些代币。

稀缺性是有价值的实物资产的重要属性。 像黄金这样的稀有金属自然是稀缺的，但数字资产并不存在固有的实物稀缺性。 编码为某些字节序列（例如 G[Alice] → 10）的数字资产在物理上并不比另一个字节序列（例如 G[Alice] → 100）更难生成或复制。相反，评估规则必须以编程方式强制执行稀缺性。

让我们考虑考虑稀缺性的第二个提案：
 |交易脚本格式       |     评估规则 |
 | :---:           |    :-----:  | 
 |<Ka, n, Kb>      |  if G[Ka] >= n then G[Ka] := G[Ka] - n; G[Kb] := G[Kb] + n; |
 
在此方案下，交易脚本指定发送者 Alice 的公钥 𝐾𝑎 和接收者 Bob 的公钥 𝐾𝑏。 评估规则现在在执行任何更新之前检查存储在𝐾𝑎下的StrawCoin数量是否至少为𝑛。 如果检查成功，则评估规则从存储在发送者密钥下的StrawCoin中减去𝑛，并将𝑛 添加到存储在接收者密钥下的StrawCoin中。 在这种方案下，执行有效的交易脚本通过保存系统中的StrawCoin数量来强制稀缺性。 Alice 不能再凭空创造 StrawCoin——她只能从她的账户中扣除给予 Bob StrawCoin。

访问控制. 尽管第二个提案解决了稀缺性问题，但它仍然存在一个问题：Bob 可以发送花费属于 Alice 的 StrawCoin 的交易。 例如，评估规则中的任何内容都不会阻止 Bob 发送交易⟨Alice, 100, Bob⟩。 我们可以通过添加基于数字签名的访问控制机制来解决这个问题：

  | 交易所脚本格式| 评估规则 |
  |  :-----:    | :-----:  |
  | SKa(<Ka, n, Kb>) | if verify_sig(SKa(<Ka, n, Kb>)) && G[Ka] >= n then G[Ka] := G[Ka] - n ; G[Kb] := G[Kb] + n|
  
该方案要求 Alice 用她的私钥签署交易脚本。 我们写 𝑆𝐾(𝑚) 用于使用与公钥 𝐾 配对的私钥对消息 𝑚 进行签名。 评估规则使用 verify_sig 函数来检查与 Alice 的公钥 𝐾𝑎 的签名。 如果签名未验证，则不执行更新。 这条新规则通过使用数字签名的不可伪造性来防止 Alice 从她自己的账户以外的任何账户中借记 StrawCoin，从而解决了之前提案的问题。

顺便说一句，请注意在第一个strawman 提案中实际上不需要评估规则——提案的状态更新直接应用于键值存储。 但随着我们对提案的推进，执行更新的先决条件与更新本身之间出现了明显的分离。 评估规则通过评估脚本来决定是否执行更新以及执行什么更新。 这种分离是基本的，因为执行访问控制和稀缺策略不可避免地需要某种形式的评估——用户提出状态更改，并且必须执行计算以确定状态更改是否符合策略。 在一个开放的系统中，参与者不能被信任来执行链下策略并直接向状态提交更新（如在第一个提案中）。 相反，访问控制策略必须由评估规则在链上执行。

### 2.3. 已存在的区块链语言

StrawCoin 是一种玩具语言，但它试图捕捉比特币脚本 [7][8] 和以太坊虚拟机字节码 [9] 语言（尤其是后者）的精髓。 尽管这些语言比 StrawCoin 更复杂，但它们面临着许多相同的问题：

1. ***资产的间接代表***。 资产使用整数编码，但整数值与资产不同。 事实上，没有代表比特币/以太币/稻草币的类型或值！ 这使得编写使用资产的程序变得笨拙且容易出错。 诸如将资产传入/传出过程或将资产存储在数据结构中等模式需要特殊的语言支持。
2. ***稀缺性不可扩展***。 语言只代表一种稀缺资产。 此外，稀缺性保护直接硬编码在语言语义中。 希望创建自定义资产的程序员必须在没有语言支持的情况下小心地重新实现稀缺性。
3. ***访问控制不灵活***。 该模型实施的唯一访问控制策略是基于公钥的签名方案。 与稀缺性保护一样，访问控制策略深深嵌入语言语义中。 如何扩展语言以允许程序员定义自定义访问控制策略并不明显。

***比特币脚本。*** Bitcoin Script 有一个简单而优雅的设计，专注于表达用于花费比特币的自定义访问控制策略。 全局状态由一组未使用的交易输出 (UTXO) 组成。 比特币脚本程序提供满足其使用的旧 UTXO 的访问控制策略的输入（例如，数字签名），并为它创建的新 UTXO 指定自定义访问控制策略。 由于比特币脚本包含用于数字签名检查的强大指令（包括多重签名 [10] 支持），因此程序员可以对各种访问控制策略进行编码。

然而，Bitcoin Script 的表达能力从根本上是有限的。 程序员不能定义自定义数据类型（因此，自定义资产）或过程，并且语言不是图灵完备的。 合作方可以通过复杂的多交易协议 [11] 执行一些更丰富的计算，或者通过“彩色硬币”[12][13] 非正式地定义自定义资产。 然而，这些方案通过将复杂性推到语言之外来工作，因此不能实现真正的可扩展性。

***以太坊虚拟机字节码*** 以太坊是一个开创性的系统，它展示了如何将区块链系统用于支付以外的用途。 以太坊虚拟机 (EVM) 字节码程序员可以发布智能合约 [14]，与以太币等资产交互并使用图灵完备语言定义新资产。 EVM 支持比特币脚本不支持的许多特性，例如用户定义的过程、虚拟调用、循环和数据结构。

然而，EVM 的表现力为代价高昂的编程错误打开了大门。 与 StrawCoin 一样，以太币在语言中具有特殊地位，并且以强制稀缺性的方式实施。 但是自定义资产的实现者（例如，通过 ERC20 [15] 标准）不会继承这些保护（如 (2) 中所述）——他们必须小心不要引入允许复制、重用或资产丢失的错误。 由于 (1) 中描述的间接表示问题和 EVM 的高度动态行为相结合，这是具有挑战性的。 特别是，将 以太转移到智能合约涉及动态调度，这导致了一类新的错误，称为重入漏洞 [16]。 备受瞩目的漏洞利用，例如 DAO 攻击 [17] 和 Parity Wallet 黑客攻击 [18]，使攻击者能够窃取价值数百万美元甚至更多的加密货币。

## 3. Move 的设计目标

Libra 的使命是打造一个简单的全球货币和金融基础设施，为数十亿人提供支持 [1]。Move 语言旨在提供一个安全、可编程的基础，可以在此基础上构建这一愿景。Move 必须能够以精确、可理解和可验证的方式表达 Libra 货币和治理规则。 从长远来看，Move 必须能够对构成金融基础设施的各种资产和相应的业务逻辑进行编码。

为了满足这些要求，我们在设计 Move 时考虑了四个关键目标：first-class 资产、灵活性、安全性和可验证性。

### 3.1. First-Class Resources

区块链系统允许用户编写直接与数字资产交互的程序。 正如我们在 2.2 节中所讨论的，数字资产具有将它们与传统编程中使用的值（如布尔值、整数和字符串）区分开来的特殊特征。 使用资产进行编程的强大而优雅的方法需要保留这些特征的表示。

Move 的关键特性是能够使用受线性逻辑启发的语义定义自定义Resources类型 [3]：Resources永远不能被复制或隐式丢弃，只能在程序存储位置之间移动。 这些安全保证由 Move 的类型系统静态强制执行。 尽管有这些特殊保护，Resouces也可以作为普通的程序值——它们可以存储在数据结构中，作为参数传递给 过程（函数），等等。 First-Class Resources是一个非常笼统的概念，程序员不仅可以使用它来实现安全的数字资产，还可以编写正确的业务逻辑来包装资产和执行访问控制策略。
  
Libra 币本身是作为普通的 Move Resouces实现的，在语言中没有特殊状态。由于 Libra 代币代表由 Libra 储备管理的现实世界资产 [19]，Move 必须允许创建Resources（例如，当新的现实世界资产进入 Libra 储备时）、修改（例如，当数字资产更改所有权时） ) 和销毁（例如，当支持数字资产的实物资产被出售时）。Move程序员可以使用模块保护对这些关键操作的访问。 Move 模块类似于其他区块链语言中的智能合约。模块声明资源类型和过程，这些过程对创建、销毁和更新其声明的资源的规则进行编码。模块可以调用其他模块声明的过程并使用其他模块声明的类型。然而，模块强制执行强大的数据抽象——一个类型在其声明模块内部是透明的，而在其外部是不透明的。此外，对资源类型 T 的关键操作只能在定义 T 的模块内执行。

### 3.2. 灵活性

Move 通过交易脚本为 Libra 增加了灵活性。 每个 Libra 交易都包含一个交易脚本，该脚本实际上是交易的主要程序。 交易脚本是包含任意Move代码的单个过程，允许自定义交易。 一个脚本可以调用区块链中发布的模块的多个过程，并对结果进行本地计算。 这意味着脚本可以执行表达性的一次性行为（例如支付一组特定的收件人）或可重用行为（通过调用封装可重用逻辑的单个过程）。

Move模块通过安全但灵活的代码组合实现了不同类型的灵活性。 在高层次上，Move 中的模块/Resources/过程之间的关系类似于面向对象编程中的类/对象/方法之间的关系。 但是，有一些重要的区别——Move 模块可以声明多种Resource类型（或零Resource类型），而 Move 过程没有 self 或 this 值的概念。 Move模块最类似于 ML样式[20]模块的受限版本。

### 3.3. 安全

Move 必须拒绝不满足关键属性的程序，例如Resource安全、类型安全和内存安全。 我们如何选择一个可执行的表示，以确保在区块链上执行的每个程序都满足这些属性？ 两种可能的方法是：
(a) 使用带有检查这些属性的编译器的高级编程语言
(b) 使用低级无类型汇编并在运行时执行这些安全检查。

Move 采取了介于这两个极端之间的方法。 Move 的可执行格式是一种类型化的字节码，它比汇编高级，但比源语言低。 字节码由字节码验证器在链上检查Resource、类型和内存安全性，然后由字节码解释器直接执行。 这种选择允许 Move 提供通常与源语言相关的安全保证，但无需将源编译器添加到受信任的计算库或将编译成本添加到交易执行的关键路径中。

### 3.4. 可验证性

理想情况下，我们将通过链上字节码分析或运行时检查来检查 Move 程序的每个安全属性。不幸的是，这是不可行的。 我们必须仔细权衡安全保证的重要性和普遍性与计算成本和通过链上验证执行保证的增加的协议复杂性。

我们的方法是尽可能多地对关键安全属性执行轻量级的链上验证，但设计 Move 语言以支持先进的链下静态验证工具。 我们做出了一些设计决策，使 Move 比大多数通用语言更适合静态验证：
1. ***没有动态调用***。 每个过程的调用点都可以静态确定。 这使得验证工具可以轻松准确地推断过程调用的效果，而无需执行复杂的调用图构造分析。 
2. ***有限的可变性***。 Move每个值的变化都是通过引用发生的。 引用是必须在单个交易脚本范围内创建和销毁的临时值。 Move 的字节码验证器使用类似于 Rust 的“借用检查”方案来确保在任何时间点最多存在一个对值的可变引用。 此外，该语言确保全局存储始终是树而不是任意图。 这允许验证工具模块化推理写操作的影响。
3. 模块化。 Move模块强制执行数据抽象并本地化对resouces的关键操作。 模块启用的封装与 Move 类型系统强制执行的保护相结合，确保为模块类型建立的属性不会被模块外部的代码违反。我们希望这种设计能够通过孤立地查看模块而不考虑外部调用者来对重要的模块不变量进行详尽的功能验证。

静态验证工具可以利用 Move 的这些属性来准确有效地检查是否存在运行时故障（例如，整数溢出）和重要的程序特定功能正确性属性（例如，最终可以由参与者来声明锁定在支付渠道中的resources）。 我们将在第 7 节中分享有关我们功能验证计划的更多细节。

## 4. Move概览

我们通过简单的点对点支付所涉及的交易脚本和模块来介绍 Move 的基础知识。该模块是真实的Libra 代币实现的简化版本。 示例交易脚本演示了模块外的恶意或粗心程序员不能违反模块resources的关键安全不变量。 示例模块展示了如何实现利用强数据抽象来建立和维护这些不变量的资源。

本节中的代码片段是用 Move 中间代码（IR) 的变体编写的。Move IR 足够高级，可以编写人类可读的代码，但又足够低级，可以直接转换为 Move 字节码。我们在 IR 中展示代码是因为基于堆栈的 Move 字节码更难阅读，我们目前正在设计 Move 源语言（参见第 7 节）。我们注意到，在执行代码之前，Move 类型系统提供的所有安全保证都在字节码级别进行了检查。
