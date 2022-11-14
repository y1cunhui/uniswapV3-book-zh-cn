---
title: "多池子交易"
weight: 4
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 多池子交易

我们现在进入这个 milestone 的核心——在我们的合约中实现多池子交易。我们将不会涉及到池子合约，因为它是一个核心合约，仅包含核心功能。多池子交易是一个使用上的特性，所以我们会在管理员合约和报价合约中对其进行实现。

## 更新管理员合约

### 单池和多池交易

在我们的现有实现中，管理合约中的 `swap` 函数仅支持单池交易，并且会需要池子地址作为参数：

```solidity
function swap(
    address poolAddress_,
    bool zeroForOne,
    uint256 amountSpecified,
    uint160 sqrtPriceLimitX96,
    bytes calldata data
) public returns (int256, int256) { ... }
```

我们现在会把它分成两个函数：单池交易和多池交易。这些函数的参数会有所不同：

```solidity
struct SwapSingleParams {
    address tokenIn;
    address tokenOut;
    uint24 tickSpacing;
    uint256 amountIn;
    uint160 sqrtPriceLimitX96;
}

struct SwapParams {
    bytes path;
    address recipient;
    uint256 amountIn;
    uint256 minAmountOut;
}
```

1. `SwapSingleParams` 的参数为池子参数、输入数量，以及一个限制价格——这与我们之前的基本一致。注意到，我们不再需要 `data` 字段。
2. `SwapParams` 的参数为路径、输出金额接受方、输入数量，以及最小输出数量。最后一个参数替代了 `sqrtPriceLimitX96`，因为在多池子交易中我们不再能使用池子合约中的滑点保护了（使用限价机制实现）。我们需要另实现一个滑点保护，检查最终的输出数量并且与 `minAmountOut` 对比：当最终输出数量小于 `minAmountOut` 的时候交易会失败。

### 核心交易逻辑

我们现在实现一个内部的 `_swap` 函数，会被单池交易和多池交易的函数调用。它的功能就是准备参数并且调用 `Pool.swap`：

```solidity
function _swap(
    uint256 amountIn,
    address recipient,
    uint160 sqrtPriceLimitX96,
    SwapCallbackData memory data
) internal returns (uint256 amountOut) {
    ...
```

`SwapCallbackData` 是一个新的数据结构，包含我们需要在 `swap` 函数和 `UniswapV3Callback` 之间传递的数据：

```solidity
struct SwapCallbackData {
    bytes path;
    address payer;
}
```

`path` 是交易路径，`payer` 是在这笔交易中付出 token 的地址——在多池交易中这个付款者会有所不同。

在 `_swap` 中我们要做的第一件事就是使用 `Path` 库来提取池子参数：

```solidity
// function _swap(...) {
(address tokenIn, address tokenOut, uint24 tickSpacing) = data
    .path
    .decodeFirstPool();
```

然后我们确认交易方向：

```solidity
bool zeroForOne = tokenIn < tokenOut;
```

接下来执行真正的交易：
```solidity
// function _swap(...) {
(int256 amount0, int256 amount1) = getPool(
    tokenIn,
    tokenOut,
    tickSpacing
).swap(
        recipient,
        zeroForOne,
        amountIn,
        sqrtPriceLimitX96 == 0
            ? (
                zeroForOne
                    ? TickMath.MIN_SQRT_RATIO + 1
                    : TickMath.MAX_SQRT_RATIO - 1
            )
            : sqrtPriceLimitX96,
        abi.encode(data)
    );
```

这部分与我们之前已有的功能一致，不过在这里我们调用 `getPool` 来找到池子。`getPool` 函数排序 token 并且调用 `PoolAddress.computeAddress`：

```solidity
function getPool(
    address token0,
    address token1,
    uint24 tickSpacing
) internal view returns (IUniswapV3Pool pool) {
    (token0, token1) = token0 < token1
        ? (token0, token1)
        : (token1, token0);
    pool = IUniswapV3Pool(
        PoolAddress.computeAddress(factory, token0, token1, tickSpacing)
    );
}
```

进行交易之后，我们需要找到哪个数量是对应输出值：

```solidity
// function _swap(...) {
amountOut = uint256(-(zeroForOne ? amount1 : amount0));
```

这样就完成了。接下来让我们看一下单池交易如何实现：

### 单池交易

`swapSingle` 仅仅是 `_swap` 包装起来而已：

```solidity
function swapSingle(SwapSingleParams calldata params)
    public
    returns (uint256 amountOut)
{
    amountOut = _swap(
        params.amountIn,
        msg.sender,
        params.sqrtPriceLimitX96,
        SwapCallbackData({
            path: abi.encodePacked(
                params.tokenIn,
                params.tickSpacing,
                params.tokenOut
            ),
            payer: msg.sender
        })
    );
}
```

注意到在这里我们构造了一个单池的路径：单池交易是仅有一个池子的多池交易。

### 多池交易

多池交易仅仅比单池交易复杂一点点。我们来看一下如何实现：

```solidity
function swap(SwapParams memory params) public returns (uint256 amountOut) {
    address payer = msg.sender;
    bool hasMultiplePools;
    ...
```

第一笔交易是由用户付费，因为用户提供最开始输入的 token。

接下来，我们开始遍历路径中的池子：

```solidity
...
while (true) {
    hasMultiplePools = params.path.hasMultiplePools();

    params.amountIn = _swap(
        params.amountIn,
        hasMultiplePools ? address(this) : params.recipient,
        0,
        SwapCallbackData({
            path: params.path.getFirstPool(),
            payer: payer
        })
    );
    ...
```

在每一次循环中，我们用这些参数调用 `_swap` 函数：
1. `params.amountIn` 跟踪输入的数量。在第一笔交易中这个数量由用户提供，在后面的交易中这个数量是来自于前一笔交易的输出数量。
2. `hasMultiplePools ? address(this) : params.recipient`——如果路径中有多个池子，收款方是管理合约，它存储中间交易得到的 token。如果在路径中仅剩一个交易（最后一笔），收款人应该是之前参数中设定的地址（通常是创建交易的人）。
3. `sqrtPriceLimitX96` 设置为 0，来禁用池子合约中的滑点保护
4. 最后一个参数是传递给 `uniswapV3SwapCallback` 的数据——我们稍后谈到。

在完成一笔交易后，我们需要前往路径中的下一个池子，或者返回：

```solidity
    ...

    if (hasMultiplePools) {
        payer = address(this);
        params.path = params.path.skipToken();
    } else {
        amountOut = params.amountIn;
        break;
    }
}
```

在这里我们修改付款人并且从路径中移除已处理的池子。

最后，新的滑点保护：

```solidity
if (amountOut < params.minAmountOut)
    revert TooLittleReceived(amountOut);
```

### Swap Callback

让我们看一下更新后的 callback：

```solidity
function uniswapV3SwapCallback(
    int256 amount0,
    int256 amount1,
    bytes calldata data_
) public {
    SwapCallbackData memory data = abi.decode(data_, (SwapCallbackData));
    (address tokenIn, address tokenOut, ) = data.path.decodeFirstPool();

    bool zeroForOne = tokenIn < tokenOut;

    int256 amount = zeroForOne ? amount0 : amount1;

    if (data.payer == address(this)) {
        IERC20(tokenIn).transfer(msg.sender, uint256(amount));
    } else {
        IERC20(tokenIn).transferFrom(
            data.payer,
            msg.sender,
            uint256(amount)
        );
    }
}
```

callback 函数在 `data_` 段获得包含路径和付款人地址的 `SwapCallbackData`。它从路径中提取 token 地址，识别交易方向，以及该合约需要转出的金额。接下来，它根据付款人的不同而进行不同行为：
1. 如果付款人是当前合约（在连续交易时，当前合约作为中间人），它直接将本合约账户下的 token 转到下一个池子（调用这个 callback 的池子）。
2. 如果付款人是一个不同的地址（创建交易的用户），它从用户那里把 token 转给池子。

## 更新报价合约

报价合约是另一个我们需要更新的合约，因为我们希望用它来计算出多池交易最后得到的输出金额。与管理合约类似，我们会有两种 `quote` 函数：一个单池的一个多池的。我们先看前者：

### 单池报价
我们仅仅需要在当前的 `quote` 实现上进行一点小改变：
1. 重命名为 `quoteSingle`；
2. 把参数放进结构体（主要是出于美观考虑）；
3. 在参数中使用 token 地址和 tick 间隔，而不是池子地址。

```solidity
// src/UniswapV3Quoter.sol
struct QuoteSingleParams {
    address tokenIn;
    address tokenOut;
    uint24 tickSpacing;
    uint256 amountIn;
    uint160 sqrtPriceLimitX96;
}

function quoteSingle(QuoteSingleParams memory params)
    public
    returns (
        uint256 amountOut,
        uint160 sqrtPriceX96After,
        int24 tickAfter
    )
{
    ...
```

在函数体中唯一的改变时使用 `getPool` 来获取池子地址：

```solidity
    ...
    IUniswapV3Pool pool = getPool(
        params.tokenIn,
        params.tokenOut,
        params.tickSpacing
    );

    bool zeroForOne = params.tokenIn < params.tokenOut;
    ...
```

### 多池报价

多池报价的实现与多池交易类似，不过使用更少的参数。

```solidity
function quote(bytes memory path, uint256 amountIn)
    public
    returns (
        uint256 amountOut,
        uint160[] memory sqrtPriceX96AfterList,
        int24[] memory tickAfterList
    )
{
    sqrtPriceX96AfterList = new uint160[](path.numPools());
    tickAfterList = new int24[](path.numPools());
    ...
```

在参数中，我们仅仅需要输入数量和交易路径。这个函数的返回值与 `quoteSingle` 类似，不过多了每一次交易后的 "price after" 和 "tick after"，因此返回值为数组。

```solidity
uint256 i = 0;
while (true) {
    (address tokenIn, address tokenOut, uint24 tickSpacing) = path
        .decodeFirstPool();

    (
        uint256 amountOut_,
        uint160 sqrtPriceX96After,
        int24 tickAfter
    ) = quoteSingle(
            QuoteSingleParams({
                tokenIn: tokenIn,
                tokenOut: tokenOut,
                tickSpacing: tickSpacing,
                amountIn: amountIn,
                sqrtPriceLimitX96: 0
            })
        );

    sqrtPriceX96AfterList[i] = sqrtPriceX96After;
    tickAfterList[i] = tickAfter;
    amountIn = amountOut_;
    i++;

    if (path.hasMultiplePools()) {
        path = path.skipToken();
    } else {
        amountOut = amountIn;
        break;
    }
}
```

这个循环的逻辑与多池交易函数中类似：
1. 获取当前池子参数；
2. 在当前池子中调用 `quoteSingle`；
3. 保存返回值；
4. 重复直到路径中没有池子，然后返回。
