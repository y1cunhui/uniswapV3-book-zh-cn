---
title: "简介"
weight: 0
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Uniswap V3 Book 中文版

中文/[English](https://uniswapv3book.com/)

<p align="center">
<img src="/images/book.jpg" alt="Uniswap V3 Development Book cover" width="480"/>
</p>

欢迎来到 DeFi（去中心化金融）和 AMM（自动做市商）的世界！这本书将会引领你走进这个神秘和有趣的世界！在这里，我们将会共同搭建最有趣、最重要的应用之一，堪称当今 DeFi 基石的项目——**Uniswap V3**！

这本书将会引领你开发一个这样的去中心化的应用，包括以下内容：
- 智能合约开发（使用 [Solidity](https://docs.soliditylang.org/en/latest/index.html)）；
- 合约测试与部署（使用 [Foundry](https://github.com/foundry-rs/foundry) 中的 Forge 和 Anvil 等工具）；
- DEX 的设计和数学原理；
- DEX 的前端应用开发（使用 [React](https://reactjs.org/) 和 [MetaMask](https://metamask.io/)）。

### 这本书不适合完全的初学者

我希望你是一个有经验的开发者，至少有任何一门编程语言的编程经验。如果你了解这本书中主要使用的编程语言 [Solidity](https://docs.soliditylang.org/en/v0.8.17/introduction-to-smart-contracts.html)，那这本书对你来说会更轻松一些。不过如果你不了解也问题不大：在这本书中，我们将逐步学习很多关于 Solidity 和 EVM（以太坊虚拟机）的内容。

### 但是，这本书适合区块链初学者

如果你只是听说过区块链，但还没有找到一个好的机会来深入理解它，那么这本书很适合你！你可以从中学习如何在区块链上开发（在本书中只涉及以太坊），区块链如何工作，如何编写和部署智能合约，如何在你的电脑上进行运行和测试，等等。

让我们开始吧！

## 参考链接：

1. 本书链接：https://y1cunhui.github.io/uniswapV3-book-zh-cn/ 
2. 本书的 Github 链接：https://github.com/y1cunhui/uniswapV3-book-zh-cn  
3. 所有的代码可以在这里找到：https://github.com/Jeiwan/uniswapv3-code  
4. 如果你认为你可以帮助 Uniswap，可以了解一下它们的 [grant 项目](https://www.notion.so/unigrants/Welcome-to-UNI-Grants-6e3e84967a984a5fb127ae749649ddc9) 
5. 如果你对于 DeFi 和区块链感兴趣，欢迎关注[原作者的 Twitter](https://twitter.com/jeiwan7) 以及[我的 Twitter](https://twitter.com/yicunhui2)
6. 原版书链接：https://uniswapv3book.com/ 
7. 原版 Github 链接：https://github.com/Jeiwan/uniswapv3-code 


## 问题或讨论？

每个章节都有对应的 [GitHub Discussion](https://github.com/Jeiwan/uniswapv3-book/discussions)。如果你在本书中遇到任何问题，欢迎在这里进行提问！

中文版目前不开放 Discussion，请去原版书仓库下提问。如有相关需求，欢迎提 issue。

---

## 完全的初学者如何入门？

这本书对于已经了解一些恒定乘积做市商和 Uniswap 的读者来说会容易一些。如果你对于 DEX 是彻底的新手，那我推荐你通过下面这些链接进行学习：（这同时也是译者的 DEX 入门课程）
1. 阅读原作者的 Uniswap V1 系列。它涵盖了最基础版本的 Uniswap，并且其中的代码也会非常简单。如果你有一些 Solidity 经验，你可以跳过其中的代码部分，因为这部分代码太基础了并且 Uniswap V2 的实现更好。
    1. [Programming DeFi: Uniswap. Part 1](https://jeiwan.net/posts/programming-defi-uniswap-1/)
    2. [Programming DeFi: Uniswap. Part 2](https://jeiwan.net/posts/programming-defi-uniswap-2/)
    3. [Programming DeFi: Uniswap. Part 3](https://jeiwan.net/posts/programming-defi-uniswap-3/)
2. 阅读原作者的 Uniswap V2 系列。我不会在其中过多深入探讨数学和相关概念，因为它们都在 V1 系列中讨论过了。但是非常值得去熟悉一下 V2 的代码——这会对于你用不同的方式思考智能合约编程有很大帮助（与我们通常写程序的方式不同）。
    1. [Programming DeFi: Uniswap V2. Part 1](https://jeiwan.net/posts/programming-defi-uniswapv2-1/)
    2. [Programming DeFi: Uniswap V2. Part 2](https://jeiwan.net/posts/programming-defi-uniswapv2-2/)
    3. [Programming DeFi: Uniswap V2. Part 3](https://jeiwan.net/posts/programming-defi-uniswapv2-3/)
    4. [Programming DeFi: Uniswap V2. Part 4](https://jeiwan.net/posts/programming-defi-uniswapv2-4/)

如果数学对你来说有点困难，可以考虑参加可汗学院的 [Algebra 1](https://www.khanacademy.org/math/algebra) 和 [Algebra 2](https://www.khanacademy.org/math/algebra2) 课程。Uniswap 中的数学并不难，但是也需要基本的代数变换知识。（译者注：国内高中数学水平足够）

## Uniswap Grants 项目

![Uniswap Foundation logo](/images/uf_logo.png)

在写作本书过程中，我收到了来自 [Uniswap Foundation](https://uniswapfoundation.mirror.xyz/) 的 grant。没有 grant 我可能不会有足够的动力和耐心来如此深入和详细地了解 Uniswap 并完成本书，这个 grant 也让这本书完全开源并且免费。你可以在[这里获取更多关于 Uniswap Grants 项目的信息](https://www.unigrants.org/) （并或许去申请！）。


