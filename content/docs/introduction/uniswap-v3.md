---
title: "Uniswap V3"
weight: 3
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---
{{< katex display >}} {{</ katex >}}

# Uniswap V3简介

> 本章节主要讲述了[Uniswap V3白皮书](https://uniswap.org/whitepaper-v3.pdf)中的内容。同样，假设你没有理解本章的所有概念也没有关系，我们在后面章节直接看代码可能会更清晰。

为了更好地理解Uniswap V3的创新之处在哪里，我们首先来看Uniswap V2的缺点有哪些。

Uniswap V2使用AMM机制实现了一个通用的交易市场。然而，并不是所有的交易对都是平等的，交易对可以根据价格的波动性分为以下两类

1. 价格波动性为中等或高的代币对。这一类包含绝大多数的代币，因为绝大多数代币并没有锚定(pegged to)到某些东西，因此其价格随着市场波动而波动。
2. 价格波动性低的代币对。这一类包含了有锚定的代币，主要为稳定币：USDT/USDC，USDC/DAI，USDT/DAI等等。也包括ETH/stETH，ETH/rETH（一些wrapped ETH）等类型。

这些类对于我们称作“流动性池配置”的概念有不同的要求。最主要的区别在于，锚定代币对需要非常高的流动性来降低大额交易对其的影响。USDC与USDT的价格必须保持在1附近，无论我要买卖多大数目的代币。由于Uniswap V2的通用AMM算法对于稳定币交易并没有很好的适配，其他的AMM（主要是[Curve](https://curve.fi)）则在稳定币交易中更加流行。

导致这个问题出现的原因在于，Uniswap V2池子的流动性是分布在无穷区域上的-即池子允许在任何价格的交易发生，从0到正无穷：


![The curve is infinite](/images/milestone_0/curve_infinite.png)

这听起来不是一个坏事，但事实上它导致了资本利用效率的不足。一个资产的历史价格通常是在某个区间内的，不管这个区间是大还是小。比如，ETH的历史价格大致在<span>$0.75</span>
到 <span>$4,800</span> 这个区间（数据来源[CoinMarketCap](https://coinmarketcap.com/currencies/ethereum/)）。在今天（2022年6月，1个ETH的现货价格是<span>$1800</span>，没有人会愿意用<span>$5000</span>购买一个ETH，所以在这个点提供流动性是毫无用处的。因此，在远离当前价格区间的、永远不会达到的某个点上提供流动性是毫无意义的

> 当然，我们都相信ETH的价格某天会达到$10000
> （译者注：仅代表原作者观点）

## 集中流动性

Uniswap V3引入了 *集中流动性(concentrated liquidity)* 的概念：LP可以选择他们希望在哪个价格区间提供流动性。这个机制通过将更多的流动性提供在一个相对狭窄的价格区间，从而大大提高了资本利用效率；这也使Uniswap的使用场景更加多样化：它现在可以对于不同价格波动性的池子进行不同的配置。这就是V3相对于V2的提升点。

简单地来说，一个Uniswap V3的交易对是许多个Uniswap V2的交易对。V2与V3的区别是，在V3中，一个交易对有许多的**价格区间**，而每个价格区间内都有**有限数量的资产**。从零到正无穷的整个价格区间被划分成了许多个小的价格区间，每一个区间中都有一定数量的流动性。而更关键的点在于，在每个小的价格区间中，**工作机制与Uniswap V2**一样。这也是为什么说一个Uniswap V3的池子就是许多个V2的池子。

下面，我们来对这种机制进行可视化。我们并不是重新选择一个有限的曲线，而是我们把它在价格$a$ 与价格$b$ 之间的部分截取出来，认为它们是曲线的边界。更进一步，我们把曲线进行平移使得边界点落在坐标轴上，于是得到了下图：


![Uniswap V3 price range](/images/milestone_0/curve_finite.png)

> 它看起来或许有点孤单， 因此Uniswap V3有许多的价格区间——这样它们就不会感到孤单了 🙂

正如我们在前一章中讲到的那样，交易token使得价格在曲线上移动，而价格区间限制了价格点的移动。当价格移动到曲线的一端时，我们说这个池子被**耗尽了**：其中一种代币的资产变成了0，无法再购买这种代币（当然，仅仅指在这个价格区间内）

假设起始价格在上面途中曲线的中间点。为了到达点$a$，我们需要购买池子里所有的$y$来使得池子里的$x$最大化；为了到达点$b$，我们需要买光池子里的$x$从而使$y$最大化。在这两个点，池子里都只剩一种token。


> Fun fact: this allows to use Uniswap V3 price ranges as limit-orders!

What happens when the current price range gets depleted during a trade? The price slips into the next price range. If the
next price range doesn't exist, the trade ends up fulfilled partially-we'll see how this works later in the book.

This is how liquidity is spread in [the USDC/ETH pool in production](https://info.uniswap.org/#/pools/0x8ad599c3a0ff1de082011efddc58f1908eb6e6d8):

![Liquidity in the real USDC/ETH pool](/images/milestone_0/usdceth_liquidity.png)

You can see that there's a lot of liquidity around the current price but the further away from it the less liquidity
there is–this is because liquidity providers strive to have higher efficiency of their capital. Also, the whole range is
not infinite, it's upper boundary is shown on the image.

## The Mathematics of Uniswap V3

Mathematically, Uniswap V3 is based on V2: it uses the same formulas, but they're... let's call it *augmented*.

To handle transitioning between price ranges, simplify liquidity management, and avoid rounding errors, Uniswap V3 uses
these new concepts:

$$L = \sqrt{xy}$$

$$\sqrt{P} = \sqrt{\frac{y}{x}}$$

$L$ is *the amount of liquidity*. Liquidity in a pool is the combination of token reserves (that is,
two numbers). We know that their product is $k$, and we can use this to derive the measure of liquidity, which is
$\sqrt{xy}$–a number that, when multiplied by itself, equals to $k$. $L$ is the geometric mean of $x$ and $y$.

$y/x$ is the price of token 0 in terms of 1. Since token prices in a pool are reciprocals of each other, we can use only
one of them in calculations (and by convention Uniswap V3 uses $y/x$). The price of token 1 in terms of token 0 is simply 
$\frac{1}{y/x}=\frac{x}{y}$. Similarly, $\frac{1}{\sqrt{P}} = \frac{1}{\sqrt{y/x}} = \sqrt{\frac{x}{y}}$.

Why using $\sqrt{p}$ instead of $p$? There are two reasons:

1. Square root calculation is not precise and causes rounding errors. Thus, it's easier to store the square root without
calculating it in the contracts (we will not store $x$ and $y$ in the contracts).
1. $\sqrt{P}$ has an interesting connection to $L$: $L$ is also the relation between the change in output amount and 
the change in $\sqrt{P}$.

    $$L = \frac{\Delta y}{\Delta\sqrt{P}}$$

> Proof:
$$L = \frac{\Delta y}{\Delta\sqrt{P}}$$
$$\sqrt{xy} = \frac{y_1 - y_0}{\sqrt{P_1} - \sqrt{P_0}}$$
$$\sqrt{xy} (\sqrt{P_1} - \sqrt{P_0}) = y_1 - y_0$$
$$\sqrt{xy} (\sqrt{\frac{y_1}{x_1}} - \sqrt{\frac{y_0}{x_0}}) = y_1 - y_0$$
$$\sqrt{y_1^2} - \sqrt{y_0^2} = y_1 - y_0$$
$$y_1 - y_0 = y_1 - y_0$$

## Pricing

Again, we don't need to calculate actual prices–we can calculate output amount right away. Also, since we're not going
to track and store $x$ and $y$, our calculation will be based only on $L$ and $\sqrt{P}$.

From the above formula, we can find $\Delta y$:

$$\Delta y = \Delta \sqrt{P} L$$

> See the third step in the proof above.

As we discussed above, prices in a pool are reciprocals of each other. Thus, $\Delta x$ is:

$$\Delta x = \Delta \frac{1}{\sqrt{P}} L$$

$L$ and $\sqrt{P}$ allow us to not store and update pool reserves. Also, we don't need to calculate $\sqrt{P}$ each time
because we can always find $\Delta \sqrt{P}$ and its reciprocal.

## Ticks

As we learned in this chapter, the infinite price range of V2 is split into shorter price ranges in V3. Each of these
shorter price ranges is limited by boundaries–upper and lower points. To track the coordinates of these boundaries,
Uniswap V3 uses *ticks*.

![Price ranges and ticks](/images/milestone_0/ticks_and_ranges.png)

In V3, the entire price range is demarcated by evenly distributed discrete ticks. Each tick has an index and corresponds
to a certain price:

$$p(i) = 1.0001^i$$

Where $p(i)$ is the price at tick $i$. Taking powers of 1.0001 has a desirable property: the difference between two
adjacent ticks is 0.01% or *1 basis point*.

> Basis point (1/100th of 1%, or 0.01%, or 0.0001) is a unit of measure of percentages in finance. You could've heard about
basis point when central banks announced changes in interest rates.

As we discussed above, Uniswap V3 stores $\sqrt{P}$, not $P$. Thus, the formula is in fact:

$$\sqrt{p(i)} = \sqrt{1.0001}^i = 1.0001 ^{\frac{i}{2}}$$

So, we get values like: $\sqrt{p(0)} = 1$, $\sqrt{p(1)} = \sqrt{1.0001} \approx 1.00005$, $\sqrt{p(-1)} \approx 0.99995$.

Ticks are integers that can be positive and negative and, of course, they're not infinite. Uniswap V3 stores $\sqrt{P}$
as a fixed point Q64.96 number, which is a rational number that uses 64 bits for the integer part and 96 bits for the
fractional part. Thus, prices are within the range: $[2^{-128}, 2^{128}]$. And ticks are within the range:

$$[log_{1.0001}2^{-128}, log_{1.0001}{2^{128}}] = [-887272, 887272]$$

> For deeper dive into the math of Uniswap V3, I cannot but recommend [this technical note](https://atiselsts.github.io/pdfs/uniswap-v3-liquidity-math.pdf)
by [Atis Elsts](https://twitter.com/atiselsts).