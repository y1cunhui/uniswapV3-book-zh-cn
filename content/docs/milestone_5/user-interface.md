---
title: "用户界面"
weight: 6
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 用户界面

在这个 milestone 中，我们增加了移除池子流动性和收集累计费用的功能。因此，我们需要在用户界面上增加这些改变，允许用户移除流动性。

## 获取 Position

为了让用户能够选择移除流动性的数量，我们首先需要从池子中获取用户的 position。为了简单起见，我们可以在管理合约中增加一个辅助函数，返回用户在某个特定池子中的 position：

```solidity
function getPosition(GetPositionParams calldata params)
    public
    view
    returns (
        uint128 liquidity,
        uint256 feeGrowthInside0LastX128,
        uint256 feeGrowthInside1LastX128,
        uint128 tokensOwed0,
        uint128 tokensOwed1
    )
{
    IUniswapV3Pool pool = getPool(params.tokenA, params.tokenB, params.fee);

    (
        liquidity,
        feeGrowthInside0LastX128,
        feeGrowthInside1LastX128,
        tokensOwed0,
        tokensOwed1
    ) = pool.positions(
        keccak256(
            abi.encodePacked(
                params.owner,
                params.lowerTick,
                params.upperTick
            )
        )
    );
}
```

这让我们不需要在前端计算池子地址和 position 信息了。

接下来，在用户输入一个 position 区间后，我们获取对应的 position：

```js
const getAvailableLiquidity = debounce((amount, isLower) => {
  const lowerTick = priceToTick(isLower ? amount : lowerPrice);
  const upperTick = priceToTick(isLower ? upperPrice : amount);

  const params = {
    tokenA: token0.address,
    tokenB: token1.address,
    fee: fee,
    owner: account,
    lowerTick: nearestUsableTick(lowerTick, feeToSpacing[fee]),
    upperTick: nearestUsableTick(upperTick, feeToSpacing[fee]),
  }

  manager.getPosition(params)
    .then(position => setAvailableAmount(position.liquidity.toString()))
    .catch(err => console.error(err));
}, 500);
```

## 获取池子地址

由于我们需要调用池子的 `burn` 和 `collect` 函数，我们还是需要在前端计算池子地址。回忆一下池子合约地址是通过 `CREATE2` 计算出来的，需要盐值和合约代码的哈希。幸运的是，Ethers.js 有一个 `getCreate2Address` 函数，允许我们在 JavaScript 中计算 `CREATE2`：

```js
const sortTokens = (tokenA, tokenB) => {
  return tokenA.toLowerCase() < tokenB.toLowerCase ? [tokenA, tokenB] : [tokenB, tokenA];
}

const computePoolAddress = (factory, tokenA, tokenB, fee) => {
  [tokenA, tokenB] = sortTokens(tokenA, tokenB);

  return ethers.utils.getCreate2Address(
    factory,
    ethers.utils.keccak256(
      ethers.utils.solidityPack(
        ['address', 'address', 'uint24'],
        [tokenA, tokenB, fee]
      )),
    poolCodeHash
  );
}
```

然而，池子的代码哈希必须被硬编码，因为我们不希望把代码储存到前端然后再计算哈希。我们可以使用 Forge 来获取对应哈希：

```bash
$ forge inspect UniswapV3Pool bytecode| xargs cast keccak 
0x...
```

然后把结果存储在 js 里的一个常数中：
```js
const poolCodeHash = "0x9dc805423bd1664a6a73b31955de538c338bac1f5c61beb8f4635be5032076a2";
```

## 移除流动性

在获取了流动性数量和池子地址之后，我们调用 `burn`：

```js
const removeLiquidity = (e) => {
  e.preventDefault();

  if (!token0 || !token1) {
    return;
  }

  setLoading(true);

  const lowerTick = nearestUsableTick(priceToTick(lowerPrice), feeToSpacing[fee]);
  const upperTick = nearestUsableTick(priceToTick(upperPrice), feeToSpacing[fee]);

  pool.burn(lowerTick, upperTick, amount)
    .then(tx => tx.wait())
    .then(receipt => {
      if (!receipt.events[0] || receipt.events[0].event !== "Burn") {
        throw Error("Missing Burn event after burning!");
      }

      const amount0Burned = receipt.events[0].args.amount0;
      const amount1Burned = receipt.events[0].args.amount1;

      return pool.collect(account, lowerTick, upperTick, amount0Burned, amount1Burned)
    })
    .then(tx => tx.wait())
    .then(() => toggle())
    .catch(err => console.error(err));
}
```

如果燃烧成功，我们立刻调用 `collect`，来收集对应的累积费用。
