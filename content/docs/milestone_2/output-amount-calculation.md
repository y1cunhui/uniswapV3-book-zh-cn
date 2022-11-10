---
title: "输出金额计算"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# 输出金额计算

我们的 Uniswap 数学公式中还缺最后一个组成部分：计算卖出 ETH (即 token $x$ )时获得的资产数量。在前一章中，我们有一个类似的公式计算购买 ETH(购买 token $x$)的场景：

$$\Delta \sqrt{P} = \frac{\Delta y}{L}$$

这个公式计算卖出 token $y$ 时的价格变化。我们把这个差价加到现价上面，来得到目标价格：

$$\sqrt{P_{target}} = \sqrt{P_{current}} + \Delta \sqrt{P}$$

现在，我们需要一个类似的公式来计算卖出 token $x$（在本案例中为 ETH）买入 token $y$（在本案例中为 USDC）时的目标价格（在本案例中为卖出 ETH）。

回忆一下，token $x$ 的变化可以用如下方式计算：

$$\Delta x = \Delta \frac{1}{\sqrt{P}}L$$

从上面公式，我们可以推导出目标价格：

$$\Delta x = (\frac{1}{\sqrt{P_{target}}} - \frac{1}{\sqrt{P_{current}}}) L$$
$$= \frac{L}{\sqrt{P_{target}}} - \frac{L}{\sqrt{P_{current}}}$$


$$\\ \sqrt{P_{target}} = \frac{\sqrt{P}L}{\Delta x \sqrt{P} + L}$$

知道了目标价格，我们就能够用前一章类似的方式计算出输出的金额

更新一下对应的 Python 脚本
```python
# Swap ETH for USDC
amount_in = 0.01337 * eth

print(f"\nSelling {amount_in/eth} ETH")

price_next = int((liq * q96 * sqrtp_cur) // (liq * q96 + amount_in * sqrtp_cur))

print("New price:", (price_next / q96) ** 2)
print("New sqrtP:", price_next)
print("New tick:", price_to_tick((price_next / q96) ** 2))

amount_in = calc_amount0(liq, price_next, sqrtp_cur)
amount_out = calc_amount1(liq, price_next, sqrtp_cur)

print("ETH in:", amount_in / eth)
print("USDC out:", amount_out / eth)
```

输出：
```shell
Selling 0.01337 ETH
New price: 4993.777388290041
New sqrtP: 5598789932670289186088059666432
New tick: 85163
ETH in: 0.013369999999998142
USDC out: 66.80838889019013
```

上述结果显示，在之前提供流动性的基础上，卖出 0.01337 ETH 可以获得 66.8 USDC。

这看起来还不错，但是我们已经受够了使用 Python！下面我们将会在 Solidity 中实现所有的数学计算。
