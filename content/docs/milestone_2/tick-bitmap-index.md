---
title: "Tick Bitmap Index"
weight: 4
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# Tick Bitmap Index
(译者注：由于 bitmap 也是数据结构中的常见名词，所以在此不做翻译)

作为我们开始动态交易的第一步，我们需要建立一个 ticks 的索引。在前一个 milestone 中，我们手动计算并硬编码了目标位置的 tick：

```solidity
function swap(address recipient, bytes calldata data)
    public
    returns (int256 amount0, int256 amount1)
{
  int24 nextTick = 85184;
  ...
}
```

当流动性在不同的价格区间中时，我们很难简单地**计算**出目标 tick。事实上，我们需要根据不同区间中的流动性来**找到**它。因此，我们需要对于所有拥有流动性的 tick 建立一个索引，之后使用这个索引来寻找 tick 直到“填满”当前交易所需的流动性。在本小节中，我们将会实现一个这样的索引。

## Bitmap

Bitmap 是一个用压缩的方式提供数据索引的常用数据结构。一个 bitmap 实际上就是一个 0 和 1 构成的数组，其中的每一位的位置和元素内容都代表某种外部的信息。每个元素可以是 0 或者 1，可以被看做是一个 flag：当值为 0 的时候，flag没有被设置；当值为 1 的时候，flag 被设置。这个方法流行的原因是整个数组可以作为二进制被存在一个数字中。

例如，数组 `111101001101001` 就是数字 31337。这个数字需要两个字节来存储（0x7a69），而这两字节能够存储 16 个 flag（1字节=8位）。

Uniswap V3 使用这个技术来存储关于 tick 初始化的信息，也即哪个 tick 拥有流动性。当 flag 设置为(1)，对应的 tick 有流动性；当 flag 设置为(0)，对应的 tick 没有被初始化，即没有流动性。让我们来看一下如何实现。

## TickBitmap 合约

在池子合约中，tick 索引存储为一个状态变量：

```solidity
contract UniswapV3Pool {
    using TickBitmap for mapping(int16 => uint256);
    mapping(int16 => uint256) public tickBitmap;
    ...
}
```

这里的存储方式是一个mapping，key 的类型是 `int16`，value 的类型是 `uint256`。想象一个无穷的 0/1 数组：

![Tick indexes in tick bitmap](/images/milestone_2/tick_bitmap.png)

数组中每个元素都对应一个 tick。为了更好地在数组中寻址，我们把数组按照字的大小划分：每个子数组为 256 位。为了找到数组中某个 tick 的位置，我们使用如下函数：

```solidity
function position(int24 tick) private pure returns (int16 wordPos, uint8 bitPos) {
    wordPos = int16(tick >> 8);
    bitPos = uint8(uint24(tick % 256));
}
```

即：我们首先找到对应的字所在的位置，然后再找到字中的位的位置。`>>8` 即除以 256。除以 256 的商为字的位置，余数为位的位置。

举个例子，我们来计算一下我们某个 tick 的位置：

```python
tick = 85176
word_pos = tick >> 8 # or tick // 2**8
bit_pos = tick % 256
print(f"Word {word_pos}, bit {bit_pos}")
# Word 332, bit 184
```

### 翻转 flag

当在池子中添加流动性时，我们需要在 bitmap 中设置一些 tick 的 flag：一个下界一个上界。我们通过 `flipTick` 方法来实现：

```solidity
function flipTick(
    mapping(int16 => uint256) storage self,
    int24 tick,
    int24 tickSpacing
) internal {
    require(tick % tickSpacing == 0); // ensure that the tick is spaced
    (int16 wordPos, uint8 bitPos) = position(tick / tickSpacing);
    uint256 mask = 1 << bitPos;
    self[wordPos] ^= mask;
}
```

> 直到本书的后面章节之前，`tickSpacing` 的值一直为 1.

找到对应的 tick 位置之后，我们需要生成一个掩码(mask)。一个掩码是一个仅有某一位（tick 对应的位）为1的数字。为了计算掩码，我们只需要计算 `2**bit_pos` (等于 `1 << bit_pos`)：

```python
mask = 2**bit_pos # or 1 << bit_pos
print(bin(mask))
#0b10000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

接下来，为了翻转一个 flag，我们将掩码与对应的 word 进行异或：

```python
word = (2**256) - 1 # set word to all ones
print(bin(word ^ mask))
#0b11111111111111111111111111111111111111111111111111111111111111111111111->0<-1111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111
```

可以看到第 184 位被翻转成了 0

### 找到下一个 tick

接下来是通过 bitmap 索引来寻找带有流动性的 tick。

在 swap 过程中，我们需要找到现在 tick 左边或者右边的下一个有流动性的 tick。在前一章中，我们[手动计算并硬编码这个值](https://github.com/Jeiwan/uniswapv3-code/blob/85b8605c37a9065c141a234ee2c18d9507eeba22/src/UniswapV3Pool.sol#L142)。但现在我们需要使用 bitmap 索引来找到这个值。我们会在 `TickMath.nextInitializedTickWithinOneWord` 方法中实现它。在这个函数中，需要实现两个场景：
1. 当卖出 token $x$ (在这里即 ETH)时，找到在同一个 word 内当前tick的**右边**下一个有流动性的tick。
2. 当卖出 token $y$ (在这里即 USDC)时，找到在同一个 word 内当前tick的**左边**下一个有流动性的tick。

这分别对应两个方向交易时价格的移动：

![Finding next initialized tick during a swap](/images/milestone_2/find_next_tick.png)

> 注意到，在代码中，方向是相反的：当购买 token $x$ 时，我们实际上在搜寻**左边**的流动性 tick；当卖出 token $x$ 时，我们搜寻**右边**的 tick。这仅仅在 word 内部成立，而 word 之间的顺序还是正序的。

如果当前 word 内不存在有流动性的 tick，我们将会在下一个循环中，在相邻的 word 中继续寻找。

现在让我们来实现：
```solidity
function nextInitializedTickWithinOneWord(
    mapping(int16 => uint256) storage self,
    int24 tick,
    int24 tickSpacing,
    bool lte
) internal view returns (int24 next, bool initialized) {
    int24 compressed = tick / tickSpacing;
    ...
```

1. 第一个参数让这个函数成为 `mapping(int16 => uint256)` 的一个方法；
2. `tick` 代表现在的 `tick`；
3. `tickSpaceing` 在本章节中一直为 1；
4. `lte` 是一个设置方向的 flag。为 true 时，我们是卖出 token $x$，在右边寻找下一个 tick；false 时相反。

```solidity
if (lte) {
    (int16 wordPos, uint8 bitPos) = position(compressed);
    uint256 mask = (1 << bitPos) - 1 + (1 << bitPos);
    uint256 masked = self[wordPos] & mask;
    ...
```

当卖出 token $x$ 时，我们需要：
1. 获得现在 tick 的对应位置
2. 求出一个掩码，当前位及所有右边的位为 1
3. 将掩码覆盖到当前 word 上，得出右边的所有位

```solidity
    ...
    initialized = masked != 0;
    next = initialized
        ? (compressed - int24(uint24(bitPos - BitMath.mostSignificantBit(masked)))) * tickSpacing
        : (compressed - int24(uint24(bitPos))) * tickSpacing;
    ...
```

接下来，`masked` 不为 0 表示右边至少有一个 tick 对应的位为 1。如果有这样的 tick，那右边就有流动性；否则就没有（在当前 word 中）。根据结果，我们要么求出下一个有流动性的 tick 位置，或者下一个 word 的最左边一位——这让我们能够在下一个循环中搜索下一个 word 里面的有流动性的 tick。


```solidity
    ...
} else {
    (int16 wordPos, uint8 bitPos) = position(compressed + 1);
    uint256 mask = ~((1 << bitPos) - 1);
    uint256 masked = self[wordPos] & mask;
    ...
```

类似地，当卖出 $y$ 时：
1. 获取下一个 tick 的位置；
2. 求出一个不同的掩码，当前位置所有左边的位为 1；
3. 应用这个掩码，获得左边的所有位。

同样，如果当前 word 左边没有有流动性的 tick，返回上一个 word 的最右边一位：

```solidity
    ...
    initialized = masked != 0;
    // overflow/underflow is possible, but prevented externally by limiting both tickSpacing and tick
    next = initialized
        ? (compressed + 1 + int24(uint24((BitMath.leastSignificantBit(masked) - bitPos)))) * tickSpacing
        : (compressed + 1 + int24(uint24((type(uint8).max - bitPos)))) * tickSpacing;
}
```

这样就完成了！

正如你所见，`nextInitializedTickWithinOneWord` 并不一定真正找到了我们想要的 tick——它的搜索范围仅仅包括当前 word。事实上，我们并不希望在这个函数里循环遍历所有的 word，因为我们并没有传入边界的参数。这个函数会在 `swap` 中正确运行——我们马上就会看到。
