---
title: "协议费率"
weight: 4
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# 协议费率

在实现 Uniswap 的过程中，你可能会有疑问：“Uniswap 如何盈利？”。事实上，它并不盈利（至少到 2022 年 9 月为止还没有）。

在我们已经完成的实现中，交易员会给 LP 支付一定费用，然而 Uniswap Labs，作为这个 DEX 的项目方，并不在这个过程中。交易员和 LP 在使用 Uniswap DEX 的过程中都不会给 Uniswap Labs 支付任何费用。为什么会这样？

事实上，Uniswap Labs 是有办法从这个 DEX 中盈利的。然而，这个机制直到现在（2022 年 9 月）都还没有启用。每个 Uniswap 池子都有一个*协议费率*的机制。协议费用从交易费用中收取：一小部分的交易费用会被用来作为协议的费用，之后会被工厂合约的所有者（ Uniswap Labs ）提取。协议费率的大小是由 UNI 币的持有者来决定的，但是它必须在交易费率的 $1/4$ 到 $1/10$（包含） 之间。

简单起见，我们将不会在我们的实现中添加协议费率。但我们可以看一看在 Uniswap 中它是怎么实现的。

协议费率大小存储在 `slot0` 中：

```solidity
// UniswapV3Pool.sol
struct Slot0 {
    ...
    // the current protocol fee as a percentage of the swap fee taken on withdrawal
    // represented as an integer denominator (1/x)%
    uint8 feeProtocol;
    ...
}
```

需要一个全局变量来追踪已经获得的协议收入：
```solidity
// accumulated protocol fees in token0/token1 units
struct ProtocolFees {
    uint128 token0;
    uint128 token1;
}
ProtocolFees public override protocolFees;
```

协议费率在 `setFeeProtocol` 函数中设置：

```solidity
function setFeeProtocol(uint8 feeProtocol0, uint8 feeProtocol1) external override lock onlyFactoryOwner {
    require(
        (feeProtocol0 == 0 || (feeProtocol0 >= 4 && feeProtocol0 <= 10)) &&
            (feeProtocol1 == 0 || (feeProtocol1 >= 4 && feeProtocol1 <= 10))
    );
    uint8 feeProtocolOld = slot0.feeProtocol;
    slot0.feeProtocol = feeProtocol0 + (feeProtocol1 << 4);
    emit SetFeeProtocol(feeProtocolOld % 16, feeProtocolOld >> 4, feeProtocol0, feeProtocol1);
}
```

正如你所见，是可以对于每种 token 收取不同的费率的。这些值包含两个 `uint8`，被打包和存储进一个 `uint8` 里面：`feeProtocol1` 左移4位，并加上 `feeProtocol0`。想要获得 `feeProtocol0`，可以用 `slot0.feeProtocol` 除以 16 取余数。`feeProtocol1` 只需要 `slot0.feeProtocol` 右移4位。能进行这样的打包是因为 `feeProtocol0` 和 `feeProtocol1` 都不会超过 10。

在交易之前，我们需要根据交易方向来选择一个协议费率：

```solidity
function swap(...) {
    ...
    uint8 feeProtocol = zeroForOne ? (slot0_.feeProtocol % 16) : (slot0_.feeProtocol >> 4);
    ...
```

为了累积协议费用，我们在计算交易中一步的数量之后，把它从交易费用中减去：


```solidity
...
while (...) {
    (..., step.feeAmount) = SwapMath.computeSwapStep(...);

    if (cache.feeProtocol > 0) {
        uint256 delta = step.feeAmount / cache.feeProtocol;
        step.feeAmount -= delta;
        state.protocolFee += uint128(delta);
    }

    ...
}
...
```

在交易结束后，需要更新全局累积的协议费用：

```solidity
if (zeroForOne) {
    if (state.protocolFee > 0) protocolFees.token0 += state.protocolFee;
} else {
    if (state.protocolFee > 0) protocolFees.token1 += state.protocolFee;
}
```

最后，工厂合约的拥有者可以通过调用 `collectProtocol` 来收集累积的协议费用：

```solidity
function collectProtocol(
    address recipient,
    uint128 amount0Requested,
    uint128 amount1Requested
) external override lock onlyFactoryOwner returns (uint128 amount0, uint128 amount1) {
    amount0 = amount0Requested > protocolFees.token0 ? protocolFees.token0 : amount0Requested;
    amount1 = amount1Requested > protocolFees.token1 ? protocolFees.token1 : amount1Requested;

    if (amount0 > 0) {
        if (amount0 == protocolFees.token0) amount0--;
        protocolFees.token0 -= amount0;
        TransferHelper.safeTransfer(token0, recipient, amount0);
    }
    if (amount1 > 0) {
        if (amount1 == protocolFees.token1) amount1--;
        protocolFees.token1 -= amount1;
        TransferHelper.safeTransfer(token1, recipient, amount1);
    }

    emit CollectProtocol(msg.sender, recipient, amount0, amount1);
}
```