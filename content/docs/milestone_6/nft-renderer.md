---
title: "NFT 渲染器"
weight: 4
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---
<译者注：在本小节中用词为 render，我们将统一把格式化数据过程翻译为渲染，防止分歧>

# NFT 渲染器

现在，我们需要构建一个 NFT 渲染器：一个处理 NFT 管理员合约中调用 `tokenURI` 的库。它会对于每个已经铸造的 token 渲染 JSON 元数据和一个 SVG。正如我们之前所说，我们需要使用 data URI 格式，它要求 base64 编码格式——这意味着我们需要在 Solidity 中的 base64 编码器。但首先，让我们先来看一下我们的 token 长什么样。

## SVG 模板

我构建了 Uniswap V3 NFT 的这样一个简化版本：

![SVG template for NFT tokens](/images/milestone_6/nft_template.png)

它的代码如下：
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 300 480">
  <style>
    .tokens {
      font: bold 30px sans-serif;
    }

    .fee {
      font: normal 26px sans-serif;
    }

    .tick {
      font: normal 18px sans-serif;
    }
  </style>

  <rect width="300" height="480" fill="hsl(330,40%,40%)" />
  <rect x="30" y="30" width="240" height="420" rx="15" ry="15" fill="hsl(330,90%,50%)" stroke="#000" />

  <rect x="30" y="87" width="240" height="42" />
  <text x="39" y="120" class="tokens" fill="#fff">
    WETH/USDC
  </text>

  <rect x="30" y="132" width="240" height="30" />
  <text x="39" y="120" dy="36" class="fee" fill="#fff">
    0.05%
  </text>

  <rect x="30" y="342" width="240" height="24" />
  <text x="39" y="360" class="tick" fill="#fff">
    Lower tick: 123456
  </text>

  <rect x="30" y="372" width="240" height="24" />
  <text x="39" y="360" dy="30" class="tick" fill="#fff">
    Upper tick: 123456
  </text>
</svg>
```

这是一个简单的 SVG 模板，我们将会实现一个 Solidity 合约来填充这个模板中的一些域并且把它在 `tokenURI` 中返回。以下这些与是在每个 token 中不同的：
1. 背景颜色，在最开始的两个 `rect` 中进行设置；色调(hue)属性（在模板中为 330）对每个 token 是不同的；
2. 对应流动性位置属于的池子的名字（模板中为 WETH/USDC）；
3. 池子费率（0.05%）；
4. 区间边界的 tick 值（123456）。

下面是两个我们合约会产生的 NFT 的例子：

![NFT example 1](/images/milestone_6/nft_example_2.png)
![NFT example 2](/images/milestone_6/nft_example_3.png)


## 依赖

Solidity 并没有提供原生的 base64 编码工具，所以我们需要使用第三方库。在这里，我们使用 [OpenZeppelin 的库](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Base64.sol)。

另一个麻烦的事情是 Solidity 对于字符串操作的支持非常匮乏。例如，我们没办法把整数穿换成字符串——但是我们在这里需要把池子费率和 tick 值渲染进 SVG 模板里面。我们会使用 [OpenZeppelin 的 Strings 库](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Strings.sol)来实现。

## 结果格式

渲染器输出的格式应该是如下这样：

```
data:application/json;base64,BASE64_ENCODED_JSON
```

JSON 文件会长这样：
```json
{
  "name": "Uniswap V3 Position",
  "description": "USDC/DAI 0.05%, Lower tick: -520, Upper text: 490",
  "image": "BASE64_ENCODED_SVG"
}
```

"image" 域的内容会是上面填充后的 SVG 模板并用 base64 编码后的结果。

## 实现渲染器

我们将会在一个额外的库合约中实现渲染器，防止 NFT 管理员合约变得太过复杂：

```solidity
library NFTRenderer {
    struct RenderParams {
        address pool;
        address owner;
        int24 lowerTick;
        int24 upperTick;
        uint24 fee;
    }

    function render(RenderParams memory params) {
        ...
    }
}
```

在 `render` 函数中，我们首先渲染一个 SVG，然后是一个 JSON。为了让代码更简洁，我们会把每一步分解成更小的步骤。

首先获取 token 的标识：

```solidity
function render(RenderParams memory params) {
    IUniswapV3Pool pool = IUniswapV3Pool(params.pool);
    IERC20 token0 = IERC20(pool.token0());
    IERC20 token1 = IERC20(pool.token1());
    string memory symbol0 = token0.symbol();
    string memory symbol1 = token1.symbol();

    ...
```

### SVG 渲染

接下来我们就可以渲染 SVG 模板了：
```solidity
string memory image = string.concat(
    "<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 300 480'>",
    "<style>.tokens { font: bold 30px sans-serif; }",
    ".fee { font: normal 26px sans-serif; }",
    ".tick { font: normal 18px sans-serif; }</style>",
    renderBackground(params.owner, params.lowerTick, params.upperTick),
    renderTop(symbol0, symbol1, params.fee),
    renderBottom(params.lowerTick, params.upperTick),
    "</svg>"
);
```

这个模板被分成了好多部分：
1. 首先是 header，包含 CSS 样式；
2. 然后渲染背景；
3. 然后渲染位置信息（token 标识和费率）；
4. 最后渲染底部信息（位置边界 tick）。

背景就是两个 `rect`。为了渲染背景，我们需要求出这个 token 对应的唯一的 hue。然后我们把所有的部分拼起来：

```solidity
function renderBackground(
    address owner,
    int24 lowerTick,
    int24 upperTick
) internal pure returns (string memory background) {
    bytes32 key = keccak256(abi.encodePacked(owner, lowerTick, upperTick));
    uint256 hue = uint256(key) % 360;

    background = string.concat(
        '<rect width="300" height="480" fill="hsl(',
        Strings.toString(hue),
        ',40%,40%)"/>',
        '<rect x="30" y="30" width="240" height="420" rx="15" ry="15" fill="hsl(',
        Strings.toString(hue),
        ',100%,50%)" stroke="#000"/>'
    );
}
```

顶部模板渲染 token 标识和池子费率：

```solidity
function renderTop(
    string memory symbol0,
    string memory symbol1,
    uint24 fee
) internal pure returns (string memory top) {
    top = string.concat(
        '<rect x="30" y="87" width="240" height="42"/>',
        '<text x="39" y="120" class="tokens" fill="#fff">',
        symbol0,
        "/",
        symbol1,
        "</text>"
        '<rect x="30" y="132" width="240" height="30"/>',
        '<text x="39" y="120" dy="36" class="fee" fill="#fff">',
        feeToText(fee),
        "</text>"
    );
}
```

费率渲染为一个小数。由于我们预先知道了所有可能的费率，我们并不需要把整数转换成小数，只需要硬编码即可：

```solidity
function feeToText(uint256 fee)
    internal
    pure
    returns (string memory feeString)
{
    if (fee == 500) {
        feeString = "0.05%";
    } else if (fee == 3000) {
        feeString = "0.3%";
    }
}
```

在底部我们渲染区间的 tick：

```solidity
function renderBottom(int24 lowerTick, int24 upperTick)
    internal
    pure
    returns (string memory bottom)
{
    bottom = string.concat(
        '<rect x="30" y="342" width="240" height="24"/>',
        '<text x="39" y="360" class="tick" fill="#fff">Lower tick: ',
        tickToText(lowerTick),
        "</text>",
        '<rect x="30" y="372" width="240" height="24"/>',
        '<text x="39" y="360" dy="30" class="tick" fill="#fff">Upper tick: ',
        tickToText(upperTick),
        "</text>"
    );
}
```

由于 tick 可能为正可能为负，我们需要正确渲染它们（带不带负号）：

```solidity
function tickToText(int24 tick)
    internal
    pure
    returns (string memory tickString)
{
    tickString = string.concat(
        tick < 0 ? "-" : "",
        tick < 0
            ? Strings.toString(uint256(uint24(-tick)))
            : Strings.toString(uint256(uint24(tick)))
    );
}
```

### JSON 渲染

现在，让我们回到 `render` 函数，来渲染 JSON。首先，我们需要渲染一个 token 描述：

```solidity
function render(RenderParams memory params) {
    ... SVG rendering ...

    string memory description = renderDescription(
        symbol0,
        symbol1,
        params.fee,
        params.lowerTick,
        params.upperTick
    );

    ...
```

token 描述是一个文本字符串，包含了我们在 token SVG 中所有相同的信息：

```solidity
function renderDescription(
    string memory symbol0,
    string memory symbol1,
    uint24 fee,
    int24 lowerTick,
    int24 upperTick
) internal pure returns (string memory description) {
    description = string.concat(
        symbol0,
        "/",
        symbol1,
        " ",
        feeToText(fee),
        ", Lower tick: ",
        tickToText(lowerTick),
        ", Upper text: ",
        tickToText(upperTick)
    );
}
```

现在我们可以开始组装 JSON 元数据了：

```solidity
function render(RenderParams memory params) {
    string memory image = ...SVG rendering...
    string memory description = ...description rendering...

    string memory json = string.concat(
        '{"name":"Uniswap V3 Position",',
        '"description":"',
        description,
        '",',
        '"image":"data:image/svg+xml;base64,',
        Base64.encode(bytes(image)),
        '"}'
    );
```

最后，返回这个结果：

```solidity
return
    string.concat(
        "data:application/json;base64,",
        Base64.encode(bytes(json))
    );
```

### 填充 `tokenURI`

现在，我们可以返回 NFT 管理员合约中的 `tokenURI` 函数，添加真正的渲染：

```solidity
function tokenURI(uint256 tokenId)
    public
    view
    override
    returns (string memory)
{
    TokenPosition memory tokenPosition = positions[tokenId];
    if (tokenPosition.pool == address(0x00)) revert WrongToken();

    IUniswapV3Pool pool = IUniswapV3Pool(tokenPosition.pool);

    return
        NFTRenderer.render(
            NFTRenderer.RenderParams({
                pool: tokenPosition.pool,
                owner: address(this),
                lowerTick: tokenPosition.lowerTick,
                upperTick: tokenPosition.upperTick,
                fee: pool.fee()
            })
        );
}
```

# Gas 花销

尽管有很多优势，在链上存储数据有一个严重的缺点：合约的部署会非常昂贵。在部署合约时，你会按照合约的大小来付费，而这里所有的字符串和模板都会显著增加 gas 开销。而当你的 SVG 越高级：拥有更多的形状，CSS 样式，动画等等，费用就会越昂贵。

我们上面实现的 NFT 渲染器并没有做 gas 优化，你可以看到在字符串中有大量重复的 `rect` 和 `text` 这种标签，而我们可以通过内部函数来防止多份存储。我牺牲了 gas 效率来保证合约的可读性比较好。在真正的数据存储在链上的 NFT 项目中，代码的可读性通常会非常的差，因为 gas 优化程度很高。

# 测试

最后一件事就是我们如何测试 NFT 图像。对 NFT 图像的所有改变都必须被追踪，来保证没有破坏渲染的地方。为了实现这点，我们需要测试 `tokenURI` 以及它的不同变种的输出（我们甚至可以预渲染出整个集合的图像然后测试，来确保部署后没有图片会出现问题）。

为了测试 `tokenURI` 的输出，我添加了这个断言函数：

```solidity
assertTokenURI(
    nft.tokenURI(tokenId0),
    "tokenuri0",
    "invalid token URI"
);
```

第一个参数是实际的输出，第二个参数是期望值所存储的文件名。这个锻压你会加载文件中的内容，并且与实际返回值相比较：

```solidity
function assertTokenURI(
    string memory actual,
    string memory expectedFixture,
    string memory errMessage
) internal {
    string memory expected = vm.readFile(
        string.concat("./test/fixtures/", expectedFixture)
    );

    assertEq(actual, string(expected), errMessage);
}
```

多亏了 `forge-std` 库提供的 `vm.readFile()` cheatcode 我们才能够在 Solidity 中实现它。这不仅仅更加简单方便，而且也更安全：我们可以配置文件系统的权限，仅允许被许可的文件操作。为了让上述测试能够运行，我们需要在 `foundry.toml` 中添加这条 `fs_permissions` 规则：

```toml
fs_permissions = [{access='read',path='.'}]
```

用下面的方法你可以读一个 SVG：

```shell
$ cat test/fixtures/tokenuri0 \
    | awk -F ',' '{print $2}' \
    | base64 -d - \
    | jq -r .image \
    | awk -F ',' '{print $2}' \
    | base64 -d - > nft.svg \
    && open nft.svg
```

> 确保你安装了 [jq](https://stedolan.github.io/jq/)。