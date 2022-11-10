---
title: "用户界面"
weight: 8
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 用户界面

现在，我们可以让我们的 web 应用看起来更像一个真正的 DEX 了。我们可以移除那些硬编码的数额，并且允许用户输入任意的数量。同样，我们还可以让用户在任意一个方向交易，所以我们需要一个按钮来改变交易方向。在更新之后，swap 的表单长这样：

```html
<form className="SwapForm">
  <SwapInput
    amount={zeroForOne ? amount0 : amount1}
    disabled={!enabled || loading}
    readOnly={false}
    setAmount={setAmount_(zeroForOne ? setAmount0 : setAmount1, zeroForOne)}
    token={zeroForOne ? pair[0] : pair[1]} />
  <ChangeDirectionButton zeroForOne={zeroForOne} setZeroForOne={setZeroForOne} disabled={!enabled || loading} />
  <SwapInput
    amount={zeroForOne ? amount1 : amount0}
    disabled={!enabled || loading}
    readOnly={true}
    token={zeroForOne ? pair[1] : pair[0]} />
  <button className='swap' disabled={!enabled || loading} onClick={swap_}>Swap</button>
</form>
```

每个输入都根据交易方向赋值给一个变量，交易方向由 `zeroForOne` 这个状态来控制。下面的输入空间总是只读的，因为这部分的数字是由报价合约计算出来。

`setAmount_` 函数做了两件事：更新上面输入框中的值，调用报价合约来计算下面输入框中的值

```js
const updateAmountOut = debounce((amount) => {
  if (amount === 0 || amount === "0") {
    return;
  }

  setLoading(true);

  quoter.callStatic
    .quote({ pool: config.poolAddress, amountIn: ethers.utils.parseEther(amount), zeroForOne: zeroForOne })
    .then(({ amountOut }) => {
      zeroForOne ? setAmount1(ethers.utils.formatEther(amountOut)) : setAmount0(ethers.utils.formatEther(amountOut));
      setLoading(false);
    })
    .catch((err) => {
      zeroForOne ? setAmount1(0) : setAmount0(0);
      setLoading(false);
      console.error(err);
    })
})

const setAmount_ = (setAmountFn) => {
  return (amount) => {
    amount = amount || 0;
    setAmountFn(amount);
    updateAmountOut(amount)
  }
}
```

注意到，我们对 `quoter` 使用了 `callStatic` —— 这就是我们在上一小节讨论到的：我们需要强制 Ethers.js 进行 static call。如果不这样，由于 `quote` 不是 pure 或者 view 的，Ethers.js 会尝试发送一个交易。

这样就完成了！现在的 UI 允许任意数量和任意方向的交易了。
