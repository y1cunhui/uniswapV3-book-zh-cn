---
title: "计算流动性"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# 计算流动性

没有流动性就无法进行交易，因此为了能够完成我们的第一笔交易，我们首先需要向池子中添加一些流动性。为了向池子合约添加流动性，我们需要知道：

1. 一个价格区间，即LP希望他的流动性仅在这个区间上提供和被利用
2. 提供流动性的数量，也即提供的两种代币的数量，我们需要将它们转入池子合约。

在本节中，我们会手动计算上述变量的值；在后续章节中，我们会在合约中对此进行实现。首先我们来考虑价格区间

## 价格区间计算
回忆一下上一章所讲，在Uniswap V3中，整个价格区间被划分成了ticks：每个tick对应一个价格，以及有一个编号。在我们的第一个实现中，我们的现货价格设置为1ETH对5000USDC。购买ETH会移除池子中的一部分ETH，从而使得价格变得高于5000USDC。我们希望在一个包含此价格的区间中提供流动性，并且要确保最终的价格**落在这个区间内**。（跨区间的交易将会在后续章节提到）。

我们需要找到3个tick：
1. 对应现货价格的tick（1ETH-5000USDC）
2. 提供流动性的价格区间上下界对应的tick。在这里，下界为4545u，上界为5500u。

（译者注：4545u，$4545，4545USDC均代表相同含义，在本书中可能会混合使用）

从之前的章节中我们知道下述公式：

$$\sqrt{P} = \sqrt{\frac{y}{x}}$$

由于我们把ETH作为资产$x$，USDC作为资产$y$，每个tick对应的值为：


$$\sqrt{P_c} = \sqrt{\frac{5000}{1}} = \sqrt{5000} \approx 70.71$$

$$\sqrt{P_l} = \sqrt{\frac{4545}{1}} \approx 67.42$$

$$\sqrt{P_u} = \sqrt{\frac{5500}{1}} \approx 74.16$$

在这里，$P_c$代表现货价格，$P_l$代表区间下界，$P_u$代表区间上界。

接下来，我们可以计算价格对应的ticks。使用下面公式：

$$\sqrt{P(i)}=1.0001^{\frac{i}{2}}$$

我们可以得到关于$i$的公式：

$$i = log_{\sqrt{1.0001}} \sqrt{P(i)}$$

> 公式中的两个根号实际上是可以消去的，但由于我们会使用$\sqrt{p}$进行计算，我们选择保留根号

这几个对应的tick分别为:
1. 现货tick: $i_c = log_{\sqrt{1.0001}} 70.71 = 85176$
2. 下界tick: $i_l = log_{\sqrt{1.0001}} 67.42 = 84222$
3. 上界tick: $i_u = log_{\sqrt{1.0001}} 74.16 = 86129$

> 计算过程使用的是Python：
> ```python
> import math
>
> def price_to_tick(p):
>     return math.floor(math.log(p, 1.0001))
>
> price_to_tick(5000)
> > 85176
>```

价格区间的计算就是这样！

最后需要提到的是，在Solidity中，Uniswap使用Q64.96来存储$\sqrt{p}$。这是一个整数位由64位表示、小数位由96位表示的定点数格式。在我们上面的计算中，价格按照浮点数形式计算：`70.71`, `67.42`, `74.16`。我们需要将它们转换成Q64.96格式，也非常简单：只需要将这个数乘以$2^{96}$，即可得到：

$$\sqrt{P_c} = 5602277097478614198912276234240$$

$$\sqrt{P_l} = 5314786713428871004159001755648$$

$$\sqrt{P_u} = 5875717789736564987741329162240$$

> 在Python中：
> ```python
> q96 = 2**96
> def price_to_sqrtp(p):
>     return int(math.sqrt(p) * q96)
>
> price_to_sqrtp(5000)
> > 5602277097478614198912276234240
> ```
> Notice that we're multiplying before converting to integer. Otherwise, we'll lose precision.


## Token Amounts Calculation

Next step is to decide how many tokens we want to deposit into the pool. The answer is: as many as we want. The amounts
are not strictly defined, we can deposit as much as it is enough to buy a small amount of ETH without making the current
price leave the price range we put liquidity into. During development and testing we'll be able to mint any amount of tokens,
so getting the amounts we want is not a problem.

For our first swap, let's deposit 1 ETH and 5000 USDC.

> Recall that the proportion of current pool reserves tells the current spot price. So if we want to put more tokens into
the pool and keep the same price, the amounts must be proportional, e.g.: 2 ETH and 10,000 USDC; 10 ETH and 500,000 USDC, etc.

## Liquidity Amount Calculation

Next, we need to calculate $L$ based on the amounts we'll deposit. This is a tricky part, so hold tight!

From the theoretical introduction, you remember that:
$$L = \sqrt{xy}$$

However, this formula is for the infinite curve 🙂 But we want to put liquidity into a limited price range, which is just
a segment of that infinite curve. We need to calculate $L$ specifically for the price range we're going to deposit liquidity
into. We need some more advanced calculations.

To calculate $L$ for a price range, let's look at one interesting fact we have discussed earlier: price ranges can be
depleted. It's absolutely possible to buy the entire amount of one token from a price range and leave the pool with only
the other token.

![Range depletion example](/images/milestone_1/range_depleted.png)

At the points $a$ and $b$, there's only one token in the range: ETH at the point $a$ and USDC at the point $b$.

That being said, we want to find an $L$ that will allow the price to move to either of the points. We want enough
liquidity for the price to reach either of the boundaries of a price range. Thus, we want $L$ to be calculated based on
the maximum amounts of $\Delta x$ and $\Delta y$.

Now, let's see what the prices are at the edges. When ETH is bought from a pool, the price is growing; when USDC is bought,
the price is falling. Recall that the price is $\frac{y}{x}$. So, at the point $a$, the price is lowest of the range;
at the point $b$, the price is highest.

>In fact, prices are not defined at these points because there's only one reserve in the pool, but what we need to
understand here is that the price around the point $b$ is higher than the start price, and the price at the point $a$ is
lower than the start price.

Now, break the curve from the image above into two segments: one to the left of the start point and one to the right of
the start point. We're going to calculate **two** $L$'s, one for each of the segments. Why? Because each of the two
tokens of a pool contributes to **either of the segments**. And since we want to distribute liquidity evenly along the
entire curve, we want to pick the minimal of the two $L$'s.


![Liquidity on the curve](/images/milestone_1/curve_liquidity.png)

And the final detail I need to focus your attention on here is: **new liquidity must not change the current price**. That
is, it must be proportional to the current proportion of the reserves. And this is why the two $L$'s can be different–when
the proportion is not preserved. And we pick the small $L$ to reestablish the proportion.

I hope this will make more sense after we implement this in code! Now, let's look at the formulas.

Let's recall how $\Delta x$ and $\Delta y$ are calculated:

$$\Delta x = \Delta \frac{1}{\sqrt{P}} L$$
$$\Delta y = \Delta \sqrt{P} L$$

We can expands these formulas by replacing the delta P's with actual prices (we know them from the above):

$$\Delta x = (\frac{1}{\sqrt{P_b}} - \frac{1}{\sqrt{P_c}}) L$$
$$\Delta y = (\sqrt{P_c} - \sqrt{P_a}) L$$

$P_a$ is the price at the point $a$, $P_b$ is the price at the point $b$, and $P_c$ is the current price.

Let's find the $L$ from the first formula:

$$\Delta x = (\frac{1}{\sqrt{P_b}} - \frac{1}{\sqrt{P_c}}) L$$
$$\Delta x = \frac{L}{\sqrt{P_b}} - \frac{L}{\sqrt{P_c}}$$
$$\Delta x = \frac{L(\sqrt{P_b} - \sqrt{P_c})}{\sqrt{P_b} \sqrt{P_c}}$$
$$L = \Delta x \frac{\sqrt{P_b} \sqrt{P_c}}{\sqrt{P_b} - \sqrt{P_c}}$$

And from the second formula:
$$\Delta y = (\sqrt{P_c} - \sqrt{P_a}) L$$
$$L = \frac{\Delta y}{\sqrt{P_c} - \sqrt{P_a}}$$

So, these are our two $L$'s, one for each of the segments:

$$L = \Delta x \frac{\sqrt{P_b} \sqrt{P_c}}{\sqrt{P_b} - \sqrt{P_c}}$$
$$L = \frac{\Delta y}{\sqrt{P_c} - \sqrt{P_a}}$$


Now, let's plug the prices we calculated earlier into them:

$$L = \Delta x \frac{\sqrt{P_b}\sqrt{P_c}}{\sqrt{P_b}-\sqrt{P_c}} = 1 ETH * \frac{67.42 * 70.71}{70.71 - 67.42}$$
After converting to Q64.96, we get:

$$L = 1519437308014769733632$$

And for the other $L$:
$$L = \frac{\Delta y}{\sqrt{P_c}-\sqrt{P_a}} = \frac{5000USDC}{74.16-70.71}$$
$$L = 1517882343751509868544$$

Of these two, we'll pick the smaller one.

> In Python:
> ```python
> sqrtp_low = price_to_sqrtp(4545)
> sqrtp_cur = price_to_sqrtp(5000)
> sqrtp_upp = price_to_sqrtp(5500)
> 
> def liquidity0(amount, pa, pb):
>     if pa > pb:
>         pa, pb = pb, pa
>     return (amount * (pa * pb) / q96) / (pb - pa)
>
> def liquidity1(amount, pa, pb):
>     if pa > pb:
>         pa, pb = pb, pa
>     return amount * q96 / (pb - pa)
> 
> liq0 = liquidity0(amount_eth, sqrtp_cur, sqrtp_upp)
> liq1 = liquidity1(amount_usdc, sqrtp_cur, sqrtp_low)
> liq = int(min(liq0, liq1))
> > 1517882343751509868544
> ```

## Token Amounts Calculation, Again

Since we choose the amounts we're going to deposit, the amounts can be wrong. We cannot deposit any amounts at any price
ranges; liquidity amount needs to be distributed evenly along the curve of the price range we're depositing into. Thus, even
though users choose amounts, the contract needs to re-calculate them, and actual amounts will be slightly different (at
least because of rounding).

Luckily, we already know the formulas:

$$\Delta x = \frac{L(\sqrt{P_b} - \sqrt{P_c})}{\sqrt{P_b} \sqrt{P_c}}$$
$$\Delta y = L(\sqrt{P_c} - \sqrt{P_a})$$

> In Python:
> ```python
> def calc_amount0(liq, pa, pb):
>     if pa > pb:
>         pa, pb = pb, pa
>     return int(liq * q96 * (pb - pa) / pa / pb)
> 
> 
> def calc_amount1(liq, pa, pb):
>     if pa > pb:
>         pa, pb = pb, pa
>     return int(liq * (pb - pa) / q96)
>
> amount0 = calc_amount0(liq, sqrtp_upp, sqrtp_cur)
> amount1 = calc_amount1(liq, sqrtp_low, sqrtp_cur)
> (amount0, amount1)
> > (998976618347425408, 5000000000000000000000)
> ```
> As you can see, the number are close to the amounts we want to provide, but ETH is slightly smaller.

> **Hint**: use `cast --from-wei AMOUNT` to convert from wei to ether, e.g.:  
> `cast --from-wei 998976618347425280` will give you `0.998976618347425280`.