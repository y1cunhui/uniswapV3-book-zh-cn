# Uniswap V3 Book中文版

中文/[English](https://github.com/Jeiwan/uniswapv3-book)

本书将引导读者如何开发一个完整的高级的去中心化应用。在本书中，我们将搭建一个去中心化交易所，[Uniswap V3](https://uniswap.org/) 的克隆



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

## 贡献

欢迎对原始仓库和本翻译仓库贡献，提 issue 或直接提 PR 均可。