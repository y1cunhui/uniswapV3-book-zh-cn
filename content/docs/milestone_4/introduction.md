---
title: "简介"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 多池子交易

在实现了跨 tick 交易之后，我们已经十分接近真实的 Uniswap V3 交易了。我们的实现中一个非常重要的限制在于，仅仅允许在同一个池子里的交易——如果某一对 token 没有池子，我们就不能在这两个 token 之间进行交易。在 Uniswap 中并不是如此，因为它允许多池子交易。在这章中，我们将在我们的实现中添加多池子交易的功能。

计划如下：
1. 首先，我们将学习并实现工厂合约；
2. 接下来，我们将探究链式交易，或者叫做多池子交易如何工作，并实现 Path 库；
3. 接下来，我们会更新前端来支持多池子交易；
4. 我们将会实现一个基本的路由，来寻找到两个 token 之间的路径；
5. 在上述过程中，我们也会学到关于 tick 间隔的知识，一种优化交易的方式。

在完成本章后，我们的实现将能够处理多池子的交易，例如，通过不同的稳定币来进行 WBTC 和 WETH 的交易：WETH → USDC → USDT → WBTC。

让我们开始吧！

> 你可以在[这个 Github branch](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_4) 找到本章完整代码.

> 本章对已有的合约做出了大量改动。 [在这里可以看到相比于上一章所有的代码变动](https://github.com/Jeiwan/uniswapv3-code/compare/milestone_3...milestone_4)

> 如果你有任何问题，欢迎在[本章的 Github Discussion 中提问和交流](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-4-multi-pool-swaps)!

