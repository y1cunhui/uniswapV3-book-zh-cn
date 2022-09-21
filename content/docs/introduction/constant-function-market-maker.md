---
title: "恒定函数做市商(CFMM)"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---
{{< katex display >}} {{</ katex >}}

# 恒定函数做市商 (Constant Function Market Makers)

> 本章节主要讲述了[Uniswap V2白皮书](https://uniswap.org/whitepaper.pdf)中的内容. 理解其中的数学原理能帮助你更好地构建像Uniswap这样的应用, 不过假设你没有理解本章全部内容也没有关系

正如我们在上一节中提到的那样，AMM的构建有许多不同的方法。我们将主要关注与构建一种特定的AMM：恒定函数做市商（有时也被称为恒定乘积做市商）。尽管名字听起来很复杂，但是它的核心数学原理只是一个非常简单的公式：


$$x * y = k$$

仅此而已，这就是AMM.

$x$ 和 $y$ 是池子合约所拥有的两种资产的数目。$k$ 是它们的乘积，我们暂时不考虑它的实际值等于多少。

> **为什么只有两种资产*x*和*y*？**
每个Uniswap的池子仅包含两种token。我们使用*x*和*y*来表示一个池子中的两种资产，其中*x*代表第一个token，*y*代表第二个token。两种token的顺序并不重要。

恒定函数做市商的原理是：**在每次交易后，*k*必须保持不变**。当用户进行交易，他们通常将一种类型的token放入池子（也即他们打算卖出的token），并且将另一种类型的token移出池子（也即打算购买的token）。这笔交易会改变池子中两种资产的数量，而上述原理表示，两种资产数目的**乘积**必须保持不变。我们之后还会在本书中看到许多次这个原理，这就是Uniswap的核心机制。

## 交易函数
现在我们知道了什么是池子以及交易的原理，接下来我们写一下交易发生时的公式：

$$(x + r\Delta x)(y - \Delta y) = k\$$

1. 一个池子包含一定数量的token 0 ($x$)和一定数量的token 1 ($y$)
2. 当我们用token 0购买token 1的时候，一些token 0被放入池子 ($\Delta x$)
3. 这个池子将给我们一定数量的token 1作为交换 ($\Delta y$)
4. 池子也会从我们付出的token 0中收取一定数量的手续费 ($r$)
5. 池子中token 0的数量发生了变化 ($x + r \Delta x$)， token 1的数量也发生了变化 ($y - \Delta y$)
6. 二者的乘积保持不变，仍然为 $k$

> 我们使用token 0和token 1这样的表述是因为代码中就是如此命名的。在现在，两个token的顺序并不关键

简单来说，我们给了池子一定数量的token 0，然后获得了一定数量的token 1。这个池子的工作就是给予我们正确数量的token 1，按照一个公平的价格。这会让我们得出以下结论：**池子决定了交易的价格**。


## 价格

池子里token的价格是如何计算的？

由于Uniswap不同的池子是不同的智能合约，**同一个池子里的两种token互为计价标准进行定价**。例如：在一个ETH/USDC的池子里，ETH的价格用USDC作为标定，而USDC的价格用ETH作为标定。假设一个ETH的价格是1000USDC，那么一个USDC的价格就是0.001ETH。每一个池子都是如此，不管token是否为稳定币（例如，ETH/BTC池）

在现实世界中，价格是根据[供求关系](https://www.investopedia.com/terms/l/law-of-supply-demand.asp)来决定的，对于AMM当然也是如此，现在，我们先不考虑需求方，只关注供应方。

池子中token的价格是由token的供应量决定的，也即**池子中拥有该token的资产数目**。token的价格也由此决定：



$$P_x = \frac{y}{x}, \quad P_y=\frac{x}{y}$$

其中 $P_x$ 和 $P_y$ 是一个token相对于另一个token的价格

这个价格被称作 *现货价格*， 它反映了当前的市场价。然而，交易实际成交的价格却并不是这个价格。现在我们再重新把需求方纳入考虑：

根据供求关系，**高需求使价格增长**，这也是我们应当在去中心化交易中满足的性质。我们希望当需求很高的时候价格会升高，并且我们能够用池子里的资产数量来衡量需求：你希望从池子中获取某个token的数量越多，价格变动就越剧烈。我们再重新考虑上面这个公式：


$$(x + r\Delta x)(y - \Delta y) = xy\$$

从这个公式中，我们能够推导出关于 $\Delta x$ 和 $\Delta y$ 的式子，这也意味着我们能够通过交易付出的token数目来计算出获得的token数目，反之亦然：


$$\Delta y = \frac{yr\Delta x}{x + r\Delta x}$$
$$\Delta x = \frac{x \Delta y}{r(y - \Delta y)}$$

事实上，这些公式就能够让我们重新计算价格。我们能够从 $\Delta y$ 公式中求出获得token数量（当我们希望卖出一定数量的token，即input amount为给定值），并且从 $\Delta x$ 的公式中求出需要提供的token数量（当我们希望购买一定数量的token，即output amount为给定值）。注意到，这里的公式是资产之间的关系，同时也把交易的数目(第一个公式中的 $\Delta x$ 和第二个公式中的 $\Delta y$)加入了计算。**这是同时考虑了供求双方的价格函数**。更进一步，我们甚至并不需要去计算价格！（因为我们直接计算出了交易的结果）


> 下面是从交易函数推导出上述价格函数的过程:
$$(x + r\Delta x)(y - \Delta y) = xy$$
$$y - \Delta y = \frac{xy}{x + r\Delta x}$$
$$-\Delta y = \frac{xy}{x + r\Delta x} - y$$
$$-\Delta y = \frac{xy - y({x + r\Delta x})}{x + r\Delta x}$$
$$-\Delta y = \frac{xy - xy - y r \Delta x}{x + r\Delta x}$$
$$-\Delta y = \frac{- y r \Delta x}{x + r\Delta x}$$
$$\Delta y = \frac{y r \Delta x}{x + r\Delta x}$$
以及:
$$(x + r\Delta x)(y - \Delta y) = xy$$
$$x + r\Delta x = \frac{xy}{y - \Delta y}$$
$$r\Delta x = \frac{xy}{y - \Delta y} - x$$
$$r\Delta x = \frac{xy - x(y - \Delta y)}{y - \Delta y}$$
$$r\Delta x = \frac{xy - xy + x \Delta y}{y - \Delta y}$$
$$r\Delta x = \frac{x \Delta y}{y - \Delta y}$$
$$\Delta x = \frac{x \Delta y}{r(y - \Delta y)}$$

## The Curve

The above calculations might seem too abstract and dry. Let's visualize the constant product function to better understand
how it works.

When plotted, the constant product function is a quadratic hyperbola:

![The shape of the constant product formula curve](/images/milestone_0/the_curve.png)

Where axes are the pool reserves. Every trade starts at the point on the curve that corresponds to the current ratio of
reserves. To calculate the output amount, we need to find a new point on the curve, which has the $x$ coordinate of $x+\Delta x$, i.e.
current reserve of token 0 + the amount we're selling. The change in $y$ is the amount of token 1 we'll get.

Let's look at a concrete example:

![Desmos chart example](/images/milestone_0/desmos.png)

1. The purple line is the curve, the axes are the reserves of a pool (notice that they're equal at the start price).
1. Start price is 1.
1. We're selling 200 of token 0. If we use only the start price, we expect to get 200 of token 1.
1. However, the execution price is 0.666, so we get only 133.333 of token 1!

This example is from [the Desmos chart](https://www.desmos.com/calculator/7wbvkts2jf) made by [Dan Robinson](https://twitter.com/danrobinson),
one of the creators of Uniswap. To build a better intuition of how it works, try making up different scenarios and
plotting them on the graph. Try different reserves, see how output amount changes when $\Delta x$ is small relative to $x$.

> As the legend goes, Uniswap was invented in Desmos.

I bet you're wondering why using such a curve? It might seem like it punishes you for trading big amounts. This is true,
and this is a desirable property! The law of supply and demand tells us that when demand is high (and supply is constant)
the price is also high. And when demand is low, the price is also lower. This is how markets work. And, magically,
the constant product function implements this mechanism! Demand is defined by the amount you want to buy, and supply is the
pool reserves. When you want to buy a big amount relative to pool reserves the price is higher than when you want to
buy a smaller amount. Such a simple formula guarantees such a powerful mechanism!

Even though Uniswap doesn't calculate trade prices, we can still see them on the curve. Surprisingly, there are multiple
prices when making a trade:

1. Before a trade, there's *a spot price*. It's equal to the relation of reserves, $y/x$ or $x/y$ depending on the
direction of the trade. This price is also *the slope of the tangent line* at the starting point.
1. After a trade, there's a new spot price, at a different point on the curve. And it's the slope of the tangent line at
this new point.
1. The actual price of the trade is the slope of the line connecting the two points!

**And that's the whole math of Uniswap! Phew!**

Well, this is the math of Uniswap V2, and we're studying Uniswap V3. So in the next part, we'll see how the mathematics
of Uniswap V3 is different.