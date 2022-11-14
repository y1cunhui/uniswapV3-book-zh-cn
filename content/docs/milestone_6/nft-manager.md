---
title: "NFT 管理员合约"
weight: 3
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# NFT 管理员合约

显然，我们并不会把 NFT 相关的功能添加到池子合约中——我们需要一个另外的合约来把 NFT 和流动性位置合并起来。回忆一下，在我们的实现过程中，我们构建了 `UniswapV3Manager` 来辅助我们与池子合约的交互（使得计算更简单，并能够进行多池子交易）。这个合约向我们展示了一个如何扩展 Uniswap 核心合约的优秀实例。现在，我们将会把这个想法进一步扩展。

我们需要一个管理合约，它实现了 ERC721 标准并且管理流动性位置。这个合约将会有 NFT 标准的功能（铸造、燃烧、转移、余额与所有权跟踪等等），同时也能够向池子添加流动性或者从池子中移除流动性。这个合约应该是池子中流动性的实际拥有者，因为我们不希望让用户不铸造 token 就添加流动性，或者移除了流动性却没有燃烧掉一个 token。我们希望每个流动性位置都与一个 NFT token 链接，并且始终保持同步。

让我们看一下我们需要在新合约中实现的功能：
1. 由于这是一个 NFT 合约，它需要有所有的 ERC721 函数，包括 `tokrnURI`，返回一个 NFT 图片的 URI；
2. `mint` 和 `burn`，来同时铸造和燃烧流动性以及 NFT；
3. `addLiquidity` 和 `removeLiquidity`，来在已有的位置上添加和移除流动性；
4. `collect`，在移除流动性之后收回费用。

让我们开始写代码吧。

## 最小合约

我们并不希望从零开始实现 ERC721 标准，所以我们使用已有的库。我们依赖中已经有了 [Solmate](https://github.com/transmissions11/solmate)，所以我们会使用[它的 ERC721 实现](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC721.sol)。

> 使用 [OpenZeppelin 的 ERC721 合约](https://github.com/OpenZeppelin/openzeppelin-contracts/tree/master/contracts/token/ERC721)也是可以的，不过这里选择使用更省 gas 的 Solmate 版本。

这是一个什么空的的最小的 NFT 管理员合约：

```solidity
contract UniswapV3NFTManager is ERC721 {
    address public immutable factory;

    constructor(address factoryAddress)
        ERC721("UniswapV3 NFT Positions", "UNIV3")
    {
        factory = factoryAddress;
    }

    function tokenURI(uint256 tokenId)
        public
        view
        override
        returns (string memory)
    {
        return "";
    }
}
```

在我们实现元数据和 SVG 渲染器之前，这里的 `tokenURI` 都会返回一个空字符串。我们添加这个函数以便于我们在实现合约剩余部分的时候，Solidity 编译器不会因此报错（Solmate ERC721 实现中的 `tokenURI` 函数被声明为 virtual，所以我们必须实现它）。

## 铸造

正如我们之前所说，铸造会包含两个操作：在池子中添加流动性和铸造一个 NFT。

为了保存池子流动性位置与 NFT 之间的关系，我们需要一个 mapping 以及一个结构体：

```solidity
struct TokenPosition {
    address pool;
    int24 lowerTick;
    int24 upperTick;
}
mapping(uint256 => TokenPosition) public positions;
```

找到一个位置我们需要：
1. 池子地址
2. 所有者地址
3. 上下界的 tick

由于 NFT 管理员合约是通过它创建的所有流动性位置的所有者，这部分不需要存储，我们只需要存储剩下两部分数据。`positions` mapping 里面的键值是 token ID；这个 mapping 把 NFT ID 与找到流动性位置所需要的数据联系在一起。

接下来我们实现铸造：

```solidity
struct MintParams {
    address recipient;
    address tokenA;
    address tokenB;
    uint24 fee;
    int24 lowerTick;
    int24 upperTick;
    uint256 amount0Desired;
    uint256 amount1Desired;
    uint256 amount0Min;
    uint256 amount1Min;
}

function mint(MintParams calldata params) public returns (uint256 tokenId) {
    ...
}
```

铸造的参数与 `UniswapV3Manager` 中的一致，多了一个 `recipient`，让我们把 NFT 铸造到一个其他地址。

在 `mint` 函数中，我们首先向池子中添加流动性：

```solidity
IUniswapV3Pool pool = getPool(params.tokenA, params.tokenB, params.fee);

(uint128 liquidity, uint256 amount0, uint256 amount1) = _addLiquidity(
    AddLiquidityInternalParams({
        pool: pool,
        lowerTick: params.lowerTick,
        upperTick: params.upperTick,
        amount0Desired: params.amount0Desired,
        amount1Desired: params.amount1Desired,
        amount0Min: params.amount0Min,
        amount1Min: params.amount1Min
    })
);
```

`_addLiquidity` 与 `UniswapV3Manager` 中 `mint` 函数的部分一致：把 tick 转换成 $\sqrt(P)$，计算流动性数量，然后调用 `pool.mint()`。

接下来，我们铸造一个 NFT：

```solidity
tokenId = nextTokenId++;
_mint(params.recipient, tokenId);
totalSupply++;
```

`tokenId` 设置为现在的 `nextTokenId`，然后后者自增。`_mint` 函数是 Solmate 的 ERC721 合约提供的。在铸造一个新的 token 之后，我们更新 `totalSupply`。

最后，我们需要把新的 token 和新的流动性位置信息存储下来：

```solidity
TokenPosition memory tokenPosition = TokenPosition({
    pool: address(pool),
    lowerTick: params.lowerTick,
    upperTick: params.upperTick
});

positions[tokenId] = tokenPosition;
```

这让我们后续可以通过 token ID 来找到流动性位置。

## 添加流动性

接下来，我们将实现一个函数，把流动性添加到已有的位置中。当我们希望我们已经提供过流动性的某个位置拥有更多流动性时我们可以调用。在这种情况下，我们并不会铸造一个新的 NFT，而只是增加一个已有位置中流动性的数量。在这里，我们仅需要提供 NFT token ID 和 要添加的 token 数量：

```solidity
function addLiquidity(AddLiquidityParams calldata params)
    public
    returns (
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1
    )
{
    TokenPosition memory tokenPosition = positions[params.tokenId];
    if (tokenPosition.pool == address(0x00)) revert WrongToken();

    (liquidity, amount0, amount1) = _addLiquidity(
        AddLiquidityInternalParams({
            pool: IUniswapV3Pool(tokenPosition.pool),
            lowerTick: tokenPosition.lowerTick,
            upperTick: tokenPosition.upperTick,
            amount0Desired: params.amount0Desired,
            amount1Desired: params.amount1Desired,
            amount0Min: params.amount0Min,
            amount1Min: params.amount1Min
        })
    );
}
```

这个函数确保了已经存在一个这样的 token，并且使用对应 position 的参数来调用 `pool.mint()`。

## 移除流动性

我们在 `UniswapV3Manager` 合约中并没有实现 `burn` 函数，因为我们希望用户作为流动性位置的所有者。现在，流动性所有者是这个 NFT 管理员合约，我们可以这样来实现燃烧流动性：


```solidity
struct RemoveLiquidityParams {
    uint256 tokenId;
    uint128 liquidity;
}

function removeLiquidity(RemoveLiquidityParams memory params)
    public
    isApprovedOrOwner(params.tokenId)
    returns (uint256 amount0, uint256 amount1)
{
    TokenPosition memory tokenPosition = positions[params.tokenId];
    if (tokenPosition.pool == address(0x00)) revert WrongToken();

    IUniswapV3Pool pool = IUniswapV3Pool(tokenPosition.pool);

    (uint128 availableLiquidity, , , , ) = pool.positions(
        poolPositionKey(tokenPosition)
    );
    if (params.liquidity > availableLiquidity) revert NotEnoughLiquidity();

    (amount0, amount1) = pool.burn(
        tokenPosition.lowerTick,
        tokenPosition.upperTick,
        params.liquidity
    );
}
```

我们同样检查 token ID 是否合法。同时，我们也需要确保位置有足够的流动性来燃烧。

## 收集 Token

NFT 管理员合约也可以在燃烧流动性之后收集 token。注意到，收集的 token 发送给了 `msg.sender`，因为这个合约代替用户管理了流动性：

```solidity
struct CollectParams {
    uint256 tokenId;
    uint128 amount0;
    uint128 amount1;
}

function collect(CollectParams memory params)
    public
    isApprovedOrOwner(params.tokenId)
    returns (uint128 amount0, uint128 amount1)
{
    TokenPosition memory tokenPosition = positions[params.tokenId];
    if (tokenPosition.pool == address(0x00)) revert WrongToken();

    IUniswapV3Pool pool = IUniswapV3Pool(tokenPosition.pool);

    (amount0, amount1) = pool.collect(
        msg.sender,
        tokenPosition.lowerTick,
        tokenPosition.upperTick,
        params.amount0,
        params.amount1
    );
}
```

## 燃烧

最后是燃烧。与这个合约中其他的函数不同，这个函数并不会对池子进行任何操作：它仅仅燃烧一个 NFT。为了燃烧，对应的位置必须是空的并且 token 已经被收集。因此，如果我们想要燃烧一个 NFT，我们需要：
1. 调用 `removeLiquidity` 来移除整个区间流动性；
2. 调用 `collect` 来收集移除流动性获得的 token；
3. 调用 `burn` 来燃烧这个 NFT token。


```solidity
function burn(uint256 tokenId) public isApprovedOrOwner(tokenId) {
    TokenPosition memory tokenPosition = positions[tokenId];
    if (tokenPosition.pool == address(0x00)) revert WrongToken();

    IUniswapV3Pool pool = IUniswapV3Pool(tokenPosition.pool);
    (uint128 liquidity, , , uint128 tokensOwed0, uint128 tokensOwed1) = pool
        .positions(poolPositionKey(tokenPosition));

    if (liquidity > 0 || tokensOwed0 > 0 || tokensOwed1 > 0)
        revert PositionNotCleared();

    delete positions[tokenId];
    _burn(tokenId);
    totalSupply--;
}
```

完成了！