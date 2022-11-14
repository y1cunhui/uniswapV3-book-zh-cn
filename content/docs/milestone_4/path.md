---
title: "交易路径"
weight: 3
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 交易路径

假设我们只有以下几个池子：WETH/USDC, USDC/USDT, WBTC/USDT。如果我们想要把 WETH 换成 WBTC，我们需要进行多步交换（WETH→USDC→USDT→WBTC），因为没有直接的 WETH/WBTC 池子。我们可以手动进行这一步，或者我们可以改进我们的合约来支持这样链式的，或者叫多池子的交易。当然，我们要选择后者！

当进行多池子交易时，我们会把上一笔交易的输出作为下一笔交易的输入。例如：
1. 在 WETH/USDC 池子，我们卖出 WETH 买入 USDC；
2. 在 USDC/USDT 池子，我们卖出前一笔交易得到的 USDC 买入 USDT；
3. 在 WBTC/USDT 池子，我们卖出前一笔交易得到的 USDT 买入 WBTC。

我们可以把这样一个序列转换成如下路径：

```
WETH/USDC,USDC/USDT,WBTC/USDT
```

并在合约中沿着这个路径进行遍历来在同一笔交易中实现多笔交易。然而，回顾一下在前一小节中我们提到我们不再需要知道池子地址，而可以通过池子参数计算出地址。因此，上述的路径可以被转换成一系列的 token：

```
WETH, USDC, USDT, WBTC
```

并且由于 tick 间隔也是一个标识池子的参数，我们也需要把它包含在路径里：

```
WETH, 60, USDC, 10, USDT, 60, WBTC
```

其中的 60 和 10 都是 tick 间隔。我们在波动性较大的池子（例如 ETH/USDC, WBTC, USDT）中使用 60 的间隔，在稳定币池子中（例如 USDC/USDT）使用 10 的间隔。

现在，有了这样的路径，我们可以遍历这条路径来获取每个池子的参数：

1. `WETH, 60, USDC`;
2. `USDC, 10, USDT`;
3. `USDT, 60, WBTC`.

知道了这些参数，我们可以使用我们前一章实现的 `PoolAddress.computeAddress` 来算出池子地址。

> 在一个池子内交易的时候我们也可以使用这个概念：路径仅包含一个池子的参数。因此，交易路径可以适用于所有类型的交易。

让我们搭建一个库来进行路径相关操作。

## Path 库

在代码中，一个交易路径是一个字节序列。在 Solidity 中，一个路径可以这样构建：

```solidity
bytes.concat(
    bytes20(address(weth)),
    bytes3(uint24(60)),
    bytes20(address(usdc)),
    bytes3(uint24(10)),
    bytes20(address(usdt)),
    bytes3(uint24(60)),
    bytes20(address(wbtc))
);
```

它长这样：
```shell
0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 # weth address
  00003c                                   # 60
  A0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 # usdc address
  00000a                                   # 10
  dAC17F958D2ee523a2206206994597C13D831ec7 # usdt address
  00003c                                   # 60
  2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599 # wbtc address
```

以下是我们需要实现的函数：
1. 计算路径中池子的数量；
2. 看一个路径是否包含多个池子；
3. 提取路径中第一个池子的参数；
4. 进入路径中的下一个 token 对；
5. 解码池子参数

### 计算路径中池子的数量

让我们首先实现计算路径中池子的数量：

```solidity
// src/lib/Path.sol
library Path {
    /// @dev The length the bytes encoded address
    uint256 private constant ADDR_SIZE = 20;
    /// @dev The length the bytes encoded tick spacing
    uint256 private constant TICKSPACING_SIZE = 3;

    /// @dev The offset of a single token address + tick spacing
    uint256 private constant NEXT_OFFSET = ADDR_SIZE + TICKSPACING_SIZE;
    /// @dev The offset of an encoded pool key (tokenIn + tick spacing + tokenOut)
    uint256 private constant POP_OFFSET = NEXT_OFFSET + ADDR_SIZE;
    /// @dev The minimum length of a path that contains 2 or more pools;
    uint256 private constant MULTIPLE_POOLS_MIN_LENGTH =
        POP_OFFSET + NEXT_OFFSET;

    ...
```

我们首先定义一系列常量：
1. `ADDR_SIZE` 是地址的大小，20字节；
2. `TICKSPACING_SIZE` 是 tick 间隔的大小，3字节（`uint24`）；
3. `NEXT_OFFSET` 是到下一个 token 地址的偏移——为了获取这个地址，我们要跳过当前地址和 tick 间隔；
4. `POP_OFFSET` 是编码的池子参数的偏移 (token address + tick spacing + token address);
5. `MULTIPLE_POOLS_MIN_LENGTH` 是包含两个或以上池子的路径长度 (一个池子的参数 + tick
spacing + token address 的集合)。

为了计算路径中的池子数量，我们减去一个地址的大小（路径中的第一个或最后一个 token）并且用剩下的值除以 `NEXT_OFFSET` 即可：

```solidity
function numPools(bytes memory path) internal pure returns (uint256) {
    return (path.length - ADDR_SIZE) / NEXT_OFFSET;
}
```

### 判断一个路径是否有多个池子
为了判断一个路径中是否有多个池子，我们只需要将路径长度与 `MULTIPLE_POOLS_MIN_LENGTH` 比较即可：

```solidity
function hasMultiplePools(bytes memory path) internal pure returns (bool) {
    return path.length >= MULTIPLE_POOLS_MIN_LENGTH;
}
```

### 提取路径中第一个池子的参数

为了实现其他的函数，我们需要一个辅助的库，因为 Solidity 没有原生的 bytes 操作函数。我们需要能够从一个字节数组中提取出一个子数组的函数，以及将 `address` 和 `uint24` 转换成字节的函数。

幸运的是，已经有一个叫做 [solidity-bytes-utils](https://github.com/GNSPS/solidity-bytes-utils) 的开源库实现了这些。为了使用这个库，我们需要扩展 Path 库里面的 bytes 类型：

```solidity
library Path {
    using BytesLib for bytes;
    ...
}
```

现在我们可以实现 `getFirstPool` 了：

```solidity
function getFirstPool(bytes memory path)
    internal
    pure
    returns (bytes memory)
{
    return path.slice(0, POP_OFFSET);
}
```

这个函数仅仅返回了 "token address + tick spacing + token address" 这一段字节。

### 前往路径中下一个 token 对

我们将会在遍历路径的时候使用下面这个函数，来扔掉已经处理过的池子。注意到我们移除的是"token address + tick spacing"，而不是完整的池子参数，因为我们还需要另一个 token 地址来计算下一个池子的地址。

```solidity
function skipToken(bytes memory path) internal pure returns (bytes memory) {
    return path.slice(NEXT_OFFSET, path.length - NEXT_OFFSET);
}
```

### 解码第一个池子的参数

最后，我们需要解码路径中第一个池子的参数：

```solidity
function decodeFirstPool(bytes memory path)
    internal
    pure
    returns (
        address tokenIn,
        address tokenOut,
        uint24 tickSpacing
    )
{
    tokenIn = path.toAddress(0);
    tickSpacing = path.toUint24(ADDR_SIZE);
    tokenOut = path.toAddress(NEXT_OFFSET);
}
```

遗憾的是，`BytesLib` 没有实现 `toUint24` 这个函数，但我们可以自己实现它！在 `BytesLib`  中有很多 `toUintXX` 这样的函数，我们可以把其中一个转换成 `uint24` 类型的：

```solidity
library BytesLibExt {
    function toUint24(bytes memory _bytes, uint256 _start)
        internal
        pure
        returns (uint24)
    {
        require(_bytes.length >= _start + 3, "toUint24_outOfBounds");
        uint24 tempUint;

        assembly {
            tempUint := mload(add(add(_bytes, 0x3), _start))
        }

        return tempUint;
    }
}
```

我们是在一个新的库合约中实现这个函数，可以直接在 Path 库中使用它：

```solidity
library Path {
    using BytesLib for bytes;
    using BytesLibExt for bytes;
    ...
}
```