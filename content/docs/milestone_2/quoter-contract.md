---
title: "报价合约"
weight: 7
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 报价合约

为了让我们的池子合约能够集成到前端，我们需要一种方式能够在交易之前就计算出对应的数量。用户会输入它们希望卖出的 token 数量，然后就能计算并且展示出它们会获得的 token 数量。我们将通过报价合约来实现这一功能。

由于 Uniswap V3 中的流动性是分散在多个价格区间中的，我们不能够仅仅通过一个公式计算出对应数量（像在 Uniswap V2 中那样）。Uniswap V3 的设计需要我们用一种不同的方法：为了获得交易数量，我们初始化一个真正的交易，并且在 callback 函数中打断它，获取到之前计算出的对应数量。也就是，我们将会模拟一笔真实的交易来计算输出数量！

我们会创建这样一个辅助合约：

```solidity
contract UniswapV3Quoter {
    struct QuoteParams {
        address pool;
        uint256 amountIn;
        bool zeroForOne;
    }

    function quote(QuoteParams memory params)
        public
        returns (
            uint256 amountOut,
            uint160 sqrtPriceX96After,
            int24 tickAfter
        )
    {
        ...
```

Quoter 合约仅实现了一个 public 的函数——`quote`。Quoter是一个对于所有池子的通用合约，因此它将池子地址作为一个参数。其他参数（`amountIn` 和 `zeroForOne`）都是模拟 swap 需要的参数。

```solidity
try
    IUniswapV3Pool(params.pool).swap(
        address(this),
        params.zeroForOne,
        params.amountIn,
        abi.encode(params.pool)
    )
{} catch (bytes memory reason) {
    return abi.decode(reason, (uint256, uint160, int24));
}
```

这个函数唯一实现的功能是调用池子合约的 `swap` 函数。这个调用应当 revert ——我们将会在 callback 中实现这一点。当 revert 发生的时候，对应的 reason 会解码并且返回。`quote` 永远不会 revert。注意到，在 data 字段，我们仅仅传入合约的地址——因为在 callback 函数中，我们需要用它来获取对应池子的 `slot0`。


```solidity
function uniswapV3SwapCallback(
    int256 amount0Delta,
    int256 amount1Delta,
    bytes memory data
) external view {
    address pool = abi.decode(data, (address));

    uint256 amountOut = amount0Delta > 0
        ? uint256(-amount1Delta)
        : uint256(-amount0Delta);

    (uint160 sqrtPriceX96After, int24 tickAfter) = IUniswapV3Pool(pool)
        .slot0();
```

在 swap 的 callback 中，我们收集我们想要的值：输出数量，新的价格以及对应的 tick。接下来，我们将会把这些值保存下来并且 revert：

```solidity
assembly {
    let ptr := mload(0x40)
    mstore(ptr, amountOut)
    mstore(add(ptr, 0x20), sqrtPriceX96After)
    mstore(add(ptr, 0x40), tickAfter)
    revert(ptr, 96)
}
```
为了节约 gas，这部分使用 [Yul](https://docs.soliditylang.org/en/latest/assembly.html) 来实现，这是一个在 Solidity 中写内联汇编的语言。让我们来逐句分析一下：

1. `mload(0x40)` 读取下一个可用 memory slot 的指针（EVM 中的 memory 组织成 32 字节的 slot 形式）；
2. 在这个 memory slot，`mstore(ptr, amountOut)` 写入 `amountOut` 。
3. `mstore(add(ptr, 0x20), sqrtPriceX96After)` 在 `amountOut` 后面写入 `sqrtPriceX96After`。
4. `mstore(add(ptr, 0x40), tickAfter)` 在 `sqrtPriceX96After` 后面写入 `tickAfter`。
5. `revert(ptr, 96)` 会 revert 这个调用，并且返回 ptr 指向位置的 96 字节数据。

所以，我们实际上就是把我们需要的值的字节表示连接起来（也就是 `abi.encode()` 做的事）。注意到偏移永远是 32 字节，即使 `sqrtPriceX96After` 大小只有 20 字节(`uint160`)，`tickAfter` 大小只有 3 字节(`uint24`)。这也是为什么我们能够使用 `abi.decode()` 来解码数据：因为 `abi.encode()` 就是把所有的数编码成 32 字节。

这样就完成了~

## 回顾

我们来回顾一下以便于更好地理解这个算法：
1. `quote` 调用一个池子的 `swap` 函数，参数是输入的 token 数量和交易方向。
2. `swap` 进行一个真正的交易，运行一个循环来填满用户需要的数量。
3. 为了从用户那里获得 token，`swap` 会调用 caller 的 callback 函数。
4. 调用者（报价合约）实现了 callback，在其中它 revert 并且附带了输出数量、新的价格、新的 tick 这些信息。
5. revert 一直向上传播到最初的 `quote` 调用
6. 在 `quote` 中，这个 revert 被捕获，reason 被解码并作为了 `quote` 调用的返回值。

希望上面的解释能让你对这个流程更加清晰！

## Quoter 的限制

这样的设计有一个非常明显的限制之处：由于 `quote` 调用了池子合约的 `swap` 函数，而 `swap` 函数既不是 pure 的也不是 view 的（因为它修改了合约状态），`quoter` 也同样不能成为一个 pure 或者 view 的函数，即使它永远都不会修改合约的状态。但是，我们却能够把 `quote` 作为一个 `getter` 函数来使用，仅仅读取合约的数据而不进行写。这样的不同之处意味着，EVM 会使用 [CALL](https://www.evm.codes/#f1) opcode 而不是 [CALLSTATIC](https://www.evm.codes/#fa)。这不算是一个太大的问题，因为 Quoter 总会 revert，并且 revert 会重置调用中修改的一切合约状态——这保证了 `quote` 并不会修改池子合约的任何状态（也即没有实际的交易发生）。

另一个有些不同的点在于，当我们使用某个库调用 `quote` 函数时通常会触发一个交易。为了解决这个问题，我们需要强制这个库使用 static call。我们将会在下一小节中看到我们如何在 Ethers.js 中实现这一点。
