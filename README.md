# Uniswap V3 Development Book中文版 (🚧 NOT FINISHED YET 🚧)
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

https://uniswapv3book.com/



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

## 译者注
如果对文章翻译有意见和改进，欢迎提issue和PR（翻译结束前，暂不接受PR）

本书原作者的其他两个系列博客也是我非常推荐的，链接附在下面欢迎感兴趣者学习

1. https://jeiwan.net/posts/building-blockchain-in-go-part-1/
如何从零开始搭建一个区块链，使用golang语言

2. https://jeiwan.net/posts/programming-defi-uniswap-1/
如果从零开始搭建uniswap，包含从V1到V2（V3即本书），也是我的DeFi入门教程
推荐阅读本书前可以先阅读该系列。

## TODO

1. [ ] Introduction
2. [x] milestone_0
   1. [x] introduction-to-markets
   2. [x] constant-function-market-maker
   3. [x] uniswap-v3
   4. [x] dev-environment
3. [ ] milestone_1
   1. [x] introduction
   2. [x] calculating-liquidity
   3. [x] providing-liquidity
   4. [ ] first-swap
   5. [ ] manager-contract
   6. [ ] deployment
   7. [ ] user-interface 
4. [ ] milestone_2
5. [ ] milestone_3
6. [ ] milestone_4
7. [ ] milestone_5
8. [ ] milestone_6
9.  [ ] proofreading
