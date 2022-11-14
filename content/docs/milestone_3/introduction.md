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

# 跨tick交易

我们现在已经完成了 Uniswap V3 实现的很大一部分，并且已经很接近原版了！然而，我们的实现仅仅支持在同一个价格区间内的交易——这也是我们在这一个 milestone 中来改进的点。

在这个milestone中，我们会：
1. 更新 `mint` 函数，使得能够在不同的价格区间提供流动性；
2. 更新 `swap` 函数，使得在当前价格区间流动性不足时能够跨价格区间交易；
3. 学习如何在智能合约中计算流动性；
4. 在 `mint` 和 `swap` 函数中实现滑点控制；
5. 更新前端用户界面，使得能够在不同价格区间添加流动性；
6. 增加对于定点数运算的一些了解。

在这个 milestone 中，我们将彻底完成 swap 这个 Uniswap 中最核心的功能！

让我们开始吧。

> 本章的完整代码可以参考[这个 Github 分支](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_3)

> 本章也对于已有的合约做了许多修改，[你可以在这里看到在上一章基础上进行的改动](https://github.com/Jeiwan/uniswapv3-code/compare/milestone_2...milestone_3)

> 如果你关于本章有任何问题，欢迎[在本章的 Github Discussion](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-3-cross-tick-swaps)提问和交流！

