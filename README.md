# Uniswap V3 Book中文版
[English Version](https://github.com/Jeiwan/uniswapv3-book)

本书将引导读者如何开发一个完整的高级的去中心化应用。在本书中，我们将搭建一个去中心化交易所，[Uniswap V3](https://uniswap.org/)的克隆



## 为什么是Uniswap？
- 它的底层数学原理非常简单，`x * y = k`，然而却十分有效
- 它是一个在简单公式基础上搭建的较为复杂的去中心化应用
- 它是去中心化且经过时间检验的。学习这个在生产环境中运行了数年并且处理了价值几十亿美元交易的应用将会大大提高你的开发水平。


## 我们要做什么

![Front-end application screenshot](/screenshot.png)

我们将会搭建Uniswap V3的一个完整克隆。它 **不是完全的复制** 并且 **不能直接在生产环境中使用** ，因为在其中我们用自己的方式实现了很多功能因此 **一定** 包含一些bug。一定不要将其部署在主网！

尽管我们将主要关注其智能合约部分，我们仍然会顺带搭建一个前端应用界面。我（即原作者）非前端开发者因此前端开发水平有限，但在这里可以向你展示一个DEX将如何集成到一个前端应用中。



完整代码可以参考以下仓库:

https://github.com/Jeiwan/uniswapv3-code

你可以在这里阅读本书:

https://y1cunhui.github.io/uniswapV3-book-zh-cn/



## 本地部署

如何本地部署该书:
1. 安装 [Hugo](https://gohugo.io/).
2. Clone the repo:
  ```shell
  $ git clone https://github.com/y1cunhui/uniswapV3-book-zh-cn
  $ cd uniswapV3-book-zh-cn
  ```
3. 运行:
  ```shell
  $ hugo server -D
  ```
4. 访问 http://localhost:1313/ (或者在你命令行输出中展示的URL)

## TODO

1. [x] Introduction
2. [x] milestone_0
   1. [x] introduction-to-markets
   2. [x] constant-function-market-maker
   3. [x] uniswap-v3
   4. [x] dev-environment
3. [x] milestone_1
   1. [x] introduction
   2. [x] calculating-liquidity
   3. [x] providing-liquidity
   4. [x] first-swap
   5. [x] manager-contract
   6. [x] deployment
   7. [x] user-interface 
4. [x] milestone_2
   1. [x] introduction
   2. [x] output-amount-calculation
   3. [x] math-in-solidity
   4. [x] tick-bitmap-index
   5. [x] generalize-minting
   6. [x] generalize-swapping
   7. [x] quoter-contract
   8. [x] user-interface
5. [x] milestone_3
   1. [x] introduction
   2. [x] different-price-ranges
   3. [x] cross-tick swaps
   4. [x] slippage protection
   5. [x] liquidity protection
   6. [x] more-on-fixed point
   7. [x] flash loans
   8. [x] user interface
6. [x] milestone_4
   1. [x] introduction
   2. [x] factory-contract
   3. [x] swap-path
   4. [x] multi-pool-swap
   5. [x] user-interface
   6. [x] tick-rounding
7. [x] milestone_5
   1. [x] introduction
   2. [x] swap-fees
   3. [x] flash-loan-fees
   4. [x] protocol-fees
   5. [x] price-oracle
   6. [x] user-interface
8. [x] milestone_6
   1. [x] introduction
   2. [x] erc721-overview
   3. [x] nft-manager
   4. [x] ntf-renderer
9.  [ ] proofreading
   1.  [x] introduction
   2.  [x] milestone_1
   3.  [ ] milestone_2
   4.  [ ] milestone_3
   5.  [ ] milestone_4
   6.  [ ] milestone_5
   7.  [ ] milestone_6
