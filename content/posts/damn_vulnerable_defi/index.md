---
title: "Damn Vulnerable Defi 解題"
date: 2023-09-12T16:34:12+08:00
draft: false
tags: ["defi"]
featuredImage: "cover.png"
---

[Damn Vulnerable Defi](https://www.damnvulnerabledefi.xyz/) 是在 AppWorks 上課時所用到的教材，有一系列關於 defi 漏洞的題目，當時作業只有解前幾題。這邊把當時的筆記以及後來多解的題目都放上來。

<!--more-->

原版的框架是 hardhat，但為了和課程同步，我們使用了由熱心人士改寫的 foundry 版本，這個是我 fork 的 [repo](https://github.com/goodhat/damn-vulnerable-defi-foundry) 。

# [Unstoppable](https://github.com/goodhat/damn-vulnerable-defi-foundry/tree/master/src/Contracts/unstoppable)

{{< admonition type=tip title="Solution" open=false >}}
直接用 ERC20 的 transfer 把 token 轉給合約。
{{< /admonition >}}

試試水溫題。目標要讓一個閃電貸合約的閃電貸功能失效。稍微閱讀一下題目就可以知道我們的目標是要讓 `poolBalance` 不等於 `balanceBefore。`

```solidity
    if (poolBalance != balanceBefore) revert AssertionViolated();
```

前者是 depositTokens() 在維護，後者則是直接呼叫 ERC20 的 balanceOf() 取得，由此可知，我們可以不透過 depositTokens()，直接用 ERC20 的 transfer 把 token 轉給合約，這樣就會讓兩者產生不同步，也就讓 flashLoan 這個 function 失效了。

# [Naive Receiver](https://github.com/goodhat/damn-vulnerable-defi-foundry/tree/master/src/Contracts/naive-receiver)

{{< admonition type=tip title="Solution" open=false >}}
為 naive-receiver 呼叫 flashloan 十次。
{{< /admonition >}}

有兩個合約，一個簡易 lending pool 具備 flashloan 功能；一個 naive-receiver 具備 flashloan 的 callback。我們的目標是讓 naive-receiver 的 ether 全部跑到 lending pool 裏面。

這個 lending protocol 是可以幫別人 flashloan，而且還收取高額的手續費 1 ether。因此我們就可以幫 naive-receiver flashloan，讓它傻傻地被抽走大量的手續費，最終沒錢，確實 naive。

Naive-receiver 一開始有 10 ether，而我們的目標是讓他的 balance = 0，因此只要幫他呼叫 flashloan 10 次即可。如果要在一個 transaction 做完，就要寫成合約。

# [Truster](https://github.com/goodhat/damn-vulnerable-defi-foundry/tree/master/src/Contracts/truster)

{{< admonition type=tip title="Solution" open=false >}}
利用 target.functionCall(data) 這個彈性極大的 external call 來幹壞事，approve 全部的 tokens。
{{< /admonition >}}

有個提供 flashloan 的 pool，我們的目標是取走裡面所有的錢。

前幾題都是秒解，但這題終於讓我卡了幾分鐘。

後來發現 lending pool 的 callback 是任何 target 的 low level call，於是就利用這個去 call 了 token 的 approve，把全部的 token 都 approve 給 attacker。這樣 flashloan 完之後，就可以用 transferFrom 把錢全部移走。

不過題目有說可以用一個 transaction 就好，但上述方法要用兩個 transaction，除非利用部署合約大法。不知道還有沒有其他方式可以只用一個 transaction。

# [SideEntrance](https://github.com/goodhat/damn-vulnerable-defi-foundry/tree/master/src/Contracts/side-entrance)

{{< admonition type=tip title="Solution" open=false >}}
Reentrance 攻擊。
{{< /admonition >}}

這題的 lender pool 沒有檢查 reentrancy，所以可以在 flashloan 的時候，用 deposit 來還錢。最後再 withdraw 全部的錢。

```solidity
function attack() external {
    lenderPool.flashLoan(1_000e18);
    lenderPool.withdraw();
    payable(msg.sender).sendValue(1_000e18);
}

function execute() external payable override {
    lenderPool.deposit{value: msg.value}(); // 用 deposit 還錢
}
```

# [The Rewarder Pool](https://github.com/goodhat/damn-vulnerable-defi-foundry/tree/master/src/Contracts/the-rewarder)

{{< admonition type=tip title="Solution" open=false >}}
這題的重點是要在週期最一開始執行閃電貸，deposit 到 reward pool。
{{< /admonition >}}

這題是要利用閃電貸掠奪質押獎勵。

花了一點力氣才搞懂計算 reward 的時間軸：時間軸會以五天為一週期，每個週期的最一開始只要有人呼叫 deposit 或者 distributeRewards 就會去取當下 accounting token 的 snapshot 作為上一週期的 token 分佈，而所有使用者可以在該週期內領取上一週期的 rewards。

因此攻擊方式就是在下一週期的一開始馬上去借一大筆 dvt，並且質押到 rewarder pool 領取，這樣上一週期的 token 分佈就會包含到這筆閃電貸，攻擊者就能取走大部分 reward。

# [Selfie](https://github.com/goodhat/damn-vulnerable-defi-foundry/tree/master/src/Contracts/selfie)

{{< admonition type=tip title="Solution" open=false >}}
去閃電貸然後很快地 take a snapshot，就可以騙過治理合約。跟 The Rewarder Pool 有點相似。
{{< /admonition >}}

有兩個合約，一個是閃電貸合約，裡面有個 drainAllFunds，只能被另一個治理合約呼叫。我們的目標就是要呼叫這個 drainAllFunds 來偷走所有的 tokens。

步驟挺簡單的，就是去閃電貸，然後呼叫 token.snapshot()，有了大量 token 之後就可以去治理合約提出 action，這裡我們就可以提出 drainAllFunds 的 action。在冷卻期過後，就可以 execute 該 action，把全部的錢抽乾。

# [Compromised](https://github.com/goodhat/damn-vulnerable-defi-foundry/tree/master/src/Contracts/compromised)

{{< admonition type=tip title="Solution" open=false >}}
利用洩漏的私鑰來操縱價格。
{{< /admonition >}}

這題和前面幾題蠻不一樣的。從題目敘述可以很快知道這應該是一個私鑰洩漏的漏洞，把題目提供的 hex 拿去 decode 會得到一串 base64 encode 的字串，再用 base64 decode 就會得到一個 uint256 型別的字串，也就是私鑰。

```solidity
// hex 4d48686a4e6a63345a575978595745304e545a6b59545931597a5a6d597a55344e6a466b4e4451344f544a6a5a475a68597a426a4e6d4d34597a49314e6a42695a6a426a4f575a69593252685a544a6d4e44637a4e574535
// base64 MHhjNjc4ZWYxYWE0NTZkYTY1YzZmYzU4NjFkNDQ4OTJjZGZhYzBjNmM4YzI1NjBiZjBjOWZiY2RhZTJmNDczNWE5?
// uint 256 string '0xc678ef1aa456da65c6fc5861d44892cdfac0c6c8c2560bf0c9fbcdae2f4735a9'
```

題目給的兩個私鑰是屬於報價地址的，而題目的 oracle 是用三個報價地址提供的價格取中位數，因此取得兩個私鑰相當於控制了價格。於是我們只要買入前壓低價格，賣出前將價格提高到 exchange balance 就能抽光 exchange。

# [Puppet](https://github.com/goodhat/damn-vulnerable-defi-foundry/tree/master/src/Contracts/puppet)

{{< admonition type=tip title="Solution" open=false >}}
由於 pool 的流動性很淺，因此少量的 swap 就可以大幅改變價格。跟 Compromised 一樣都是操縱預言機的價格。
{{< /admonition >}}

有個合約 puppet 只要抵押兩倍價值的 eth 就可以借出 dvt，然而這個合約判斷價值的方式是去看 uniswap v1 pool 中的池子比例。因此我們可以倒賣 dvt 到 pool 中，此時 puppet 就會認為 dvt 的價值很低，就可以用很低的 eth，借出所有 puppet 擁有的 dvt。

這題用了 foundry cheat sheet 中的 deployCode 來部署 UniswapV1 合約。

# [Puppet V2](https://github.com/goodhat/damn-vulnerable-defi-foundry/tree/master/src/Contracts/puppet-v2)

{{< admonition type=tip title="Solution" open=false >}}
同 puppet。
{{< /admonition >}}

和 puppet 只差在 pool 變成 uniswap v2 了，但是 puppetV2 依然是去看 pool 池子比例來決定價格，因此做法和 puppet 一樣，只需要改變 interface 以及另外處理 weth 的兌換。
