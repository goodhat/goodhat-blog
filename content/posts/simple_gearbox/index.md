---
title: "AppWorks 專題：簡化版 Gearbox"
date: 2023-09-08T00:51:37+08:00
tags: ["appworks", "defi"]
draft: false
featuredImage: "gearbox.png"
---

在 AppWorks Blockchain 課程的最後，每個學員都要完成一個 project，題目不限，只要能夠顯示自己在這 18 週所學到的東西即可，而我選擇了當時導師推薦的一個 defi project 來臨摹，也就是 [Gearbox](https://github.com/Gearbox-protocol/core-v2)。

<!--more-->

> 不負責任聲明：此篇文寫拖稿至課程結束後的兩個月，已經忘記某些 gearbox 的原理以及當初自己的實作細節了...

## Gearbox 簡介

Gearbox 本質上是一個借貸的 defi 協議，和每個借貸協議一樣，協議中的角色分為「貸款方」和「借款方」，貸款方將將代幣丟到池子中賺取被動利息，而借款方從池子中把錢借走，還款時則要付上利息，除了借貸雙方外，也仰賴「清算方」監控借款方部位的健康狀態。以上都跟 Compound 和 AAVE 等老牌借貸協議相同，Gearbox 不同的地方在於：**Gearbox 不需要借款者超額抵押**，只需要用一部份的保證金就可以槓桿開倉，這樣的機制很像我們平常使用的中心化交易所。

不用超額抵押？但在去中心化的世界中，代幣借出去就真的借走了，要怎麼確定借款者不會捲款跑路呢？為了防止捲款情形，Gearbox 設下了非常多的限制：

1. 使用者只能在協議建立的錢包開倉。
2. 所有操作都一定要透過 Gearbox 才能進行。
3. Gearbox 有設立白名單，只有在白名單內的第三方協議，例如：Uniswap、Convex 等，使用者才能夠與其互動。

打個比方，當我們入金到 Gearbox 時，好比進入了一個賭場，你不會真的擁有任何代幣，所有錢都在賭場幫我們開的帳戶，而我們只會拿掉一台賭場給的平板顯示我們剩餘的資產。這些資產只能在賭場內使用，所以也只能玩賭場有提供的遊戲。只要符合賭場規則，我們想怎麼賺怎麼賠都沒問題。假如運氣好賺了不少，可以在離開賭場前把帳戶裡的代幣都領出來；反之，只要虧損超過一定的比例，賭場就會沒收你的帳戶，叫你走人。用賭場描述其實有不太正確的地方，帳戶中的錢並不是真的只能待在賭場中，其實是可以領出來的，只要留在裡面的錢還夠讓賭場放心即可。

> 不應該用賭場來舉例的，好像把使用 Gearbox 講成在賭博...

## 核心合約

Gearbox 算是一個中度規模的專案，在工程上切成好幾個不同功用的合約，這邊挑出其中五個最重要的來講：

1. [CreditAccount](https://github.com/Gearbox-protocol/core-v2/blob/main/contracts/credit/CreditAccount.sol)：

   每個使用者在 Gearbox 開倉時都會產生的合約帳戶就是 `CreditAccount` 。這個合約的內容非常簡單，除了用來 transfer ERC20 token 的 function 外，還有一個 execute 可以用來對其他合約執行 low level call。只有 `CreditManager` 才能和 `CreditAccount` 互動。

2. [CreditManager](https://github.com/Gearbox-protocol/core-v2/blob/main/contracts/credit/CreditManager.sol)：

   Gearbox 的核心邏輯都在這裡，包含權限管理、對合約帳戶執行交易、抵押品計算等重要功能。

3. [CreditFacade](https://github.com/Gearbox-protocol/core-v2/blob/main/contracts/credit/CreditFacade.sol)
   使用者實際互動的合約，有點像是 Uniswap V2 中的 periphery 合約，相對地，`CreditManager` 就是 Uniswap V2 中的 core 合約。

4. [Adapter](https://github.com/Gearbox-protocol/integrations-v2/tree/main/contracts/adapters)
   前面有提到 Gearbox 有個白名單的機制，使用者只能跟白名單中的第三方合約互動，這些合約通常是一些比較有保障的老牌 defi 協議，例如 Uniswap、Curve、Compound、euler 等。每一個第三方合約都有對應的 `Adapter`，讓 Gearbox 的合約可以跟外部合約對接。

5. [PoolService](https://github.com/Gearbox-protocol/core-v2/blob/main/contracts/pool/PoolService.sol)
   顧名思義，就是掌管資金池的合約，控制資金出入以及計算利息。

## 核心功能及流程

因為當初時間有限，所以只實作 Gearbox 中最核心的功能，成果就是一個極度簡化版的 Gearbox，實作的功能包含槓桿開倉 (open credit account)、執行 Uniswap V2 交易 (multicall)、清算 (liquidate credit account)。

### Open Credit Account

{{< image src="./open_account.svg" caption="Open Credit Account 流程" width="100%" >}}

使用者開倉時要選擇入金數量以及槓桿，對 CreditFacade 發起交易，然後由 CreditManager 建立 CreditAccount，並跟 PoolService 借代幣。假設使用者投入了 1000 個代幣並開了五倍槓桿，那麼整個流程走完後，使用者的合約帳戶中會有 5000 代幣，並且負債 4000 代幣。

前面忘了提，在 Gearbox 中每個提供借款的代幣都是各自獨立的，以 USDC 為例，使用者在 USDC 的開倉，就只會跟 USDC 的 CreditFacade 互動，也只能開 USDC 的槓桿。如果使用者也想要開 ETH 的倉，就必須去跟另一組 Credit 系列的合約互動，而且因為 CreditManager 是分開的，因此兩邊的資產不能互相 cover，也就是說就算 USDC 的倉位賺很多，也沒辦法阻止 ETH 賠到被清算。

當初在知道 Gearbox 這個限制後，我原本想的 project 題目就是要打破這個限制，讓同個使用者在不同 underlying 的 credit account 的抵押品可以一起統計，但礙於個人能力不足，就放棄了。後來也漸漸明白為何 Gearbox 當初沒有選擇混在一起的設計，不僅可以簡化合約工程，將不同資金池切開也能減少系統化的風險。

{{< admonition type=note title="Note" open=true >}}
所有的資產都會在 credit account 中。
{{< /admonition >}}

### Multicall

{{< image src="./multicall.svg" caption="Open Credit Account 流程" width="100%" >}}

我個人認為這是 Gearbox 最有趣的功能，multicall 讓使用者可以在一個交易中執行多個且複雜的行為，這樣的概念其實不算新鮮，例如有個來自台灣的產品 Furucombo 就是類似的概念，只是我還是個合約新手，對於這種玩弄 low level call 的功能特別有興趣！

Gearbox 的 multicall 的流程稍微有點複雜，以 UniswapV2Router `swapTokensForExactTokens` 為例，因為要呼叫的 UniswapV2Router 屬於白名單內的第三方，因此 CreditFacade 第一步是去呼叫對應的 adapter 的 function。Adapter 裡面定義的就是 Gearbox 應該要如何使用這些第三方合約，對 `swapTokensForExactTokens` 來說很重要的是一定要 approve token 給 router，所以第一步就是叫 CreditManager 讓 CreditAccount approve 對應的 ERC20 token 給 router；第二步是執行真正的 swap；第三步則是 revoke allowance。

對 `swapTokensForExactTokens` 來說這樣就完成了，接下來就會執行下一個 call 直到結束。中途如果 credit account 的資產少於最低抵押金額也沒關係，因為 Gearbox 只會在 multicall 全部結束後才檢查帳戶內的抵押品總價值是否有達標準，這樣的特性和閃電貸有點像。

Multicall 讓 Gearbox 得以利用 web app 前端擴展產品服務的可能性，就算是合約沒有定義的功能，或者是第三方合約後來才推出的新功能，只要寫好對應的 calldata ，都能透過 web app 提供給使用者，大幅增加 Gearbox 的可玩性。在我研究 gearbox 時，它們 web app 中最熱門的就是透過槓桿增加 Convex 的獲益。

{{< admonition type=note title="Note" open=true >}}
Multicall 利用 low-level call 大幅增加 Gearbox 的彈性。
{{< /admonition >}}

### Liquidation

{{< image src="./liquidate.svg" caption="Open Credit Account 流程" width="100%" >}}

既然是借貸協議，那肯定會有清算的功能。傳統的借貸協議如 Compound 仰賴有人來替借款人還債以換取其抵押品，Gearbox 有點類似，又有點不太一樣。相同的地方在於 Gearbox 也是開放所有人來清算資產。然而，不同的地方在於，清算的過程不是單純地買下資產和負債，而是有點像幫借款人整理 credit account 的資產負債，使得帳戶回到健康狀態。**因為借款人的資產都還留在 credit account 中，清算方只要提供一串 multicall 幫借款方打理好資產負債（如上圖）**！假如錢不夠又想要去 Compound 清算的話還得閃電貸，在 Gearbox 完全不需要。

補充說明一下，Gearbox 有定義每種資產相對於某個 underlying 的抵押價值下限 [LT (Liquidation Threshold)](https://docs.gearbox.finance/overview/credit-account/allowedlist-policy#allowed-assets-list)。舉個例子，假如 ETH 對 USDC 的 LT = 0.85，就代表資產中 ETH 的價值要乘上 0.85 才能和負債相比。LT 的存在意義是一種緩衝，大部分的借貸協議都有類似的機制，來防止市況太差導致一瞬間借款人資不抵債。Gearbox 針對這個機制提供了簡短而清楚的[介紹](https://docs.gearbox.finance/traders-and-farmers/pro-leverage-bible)，蠻值得一看。

{{< admonition type=note title="Note" open=true >}}
清算方只要提供一串 multicall 幫借款方打理好資產負債
{{< /admonition >}}

## 簡化版 Gearbox

前面介紹的功能就是我在專題中有實作的，[github 連結在此](https://github.com/goodhat/Simple-Gearbox)。我使用 foundry 開發，除了 contract 外，也有 fork mainnet 寫 test。

另外，我所參考的是 Gearbox v2，現在似乎已經有 v3 囉！

這邊提幾個被我省略的地方：

- Account Facotry：
  在區塊鏈上建立一個新的合約帳戶是非常消耗 gas 的，即便使用 proxy pattern 節省 gas，對使用者來說依然是一筆開銷，而且當使用者不再使用 Gearbox ，當初建立的合約就會晾在鏈上。為了避免不必要的浪費， Gearbox 設計了一種很有趣的機制， [`AccountFactory`](https://github.com/Gearbox-protocol/core-v2/blob/main/contracts/core/AccountFactory.sol) 會事先用 proxy pattern 創建出好幾個合約帳戶，並且用 linked list 的方式把所有帳戶連在一起。在 linked list 中的合約帳戶代表是沒有人使用的，當有人要開倉， Gearbox 就會把 linked list 的 head 合約釋出來給使用者，並將 head 指向下一個合約；當有人要關倉， Gearbox 就會回收該合約帳戶，並且把它接到 linked list 的 tail。挺聰明且有趣的設計！
- 多種清算方式：
  我只實作了最簡單的清算情況，真實的 Gearbox 有對不同的資產負債狀態提供不同的清算機制。

- Access control and address provider：
  Gearbox 為了管理方便，有幾個負責管理權限及地址的核心合約。

- Oracle：
  為了測試方便，我都是自己餵價格，真正的 Gearbox 當然是使用 oracle。

- 利率：
  因為偷懶，我直接把利率設為 0，這大幅簡化了計算利息的部分。

## 結語

當時在課程很早的時候就說每個人都要做 final project，而我在選題時卡了很久，我知道我想做 defi 的東西練習看看，但不是沒有想法，不然就是想法太複雜。最後是諮詢了導師之後，他們建議可以看看 Gearbox。回頭看來，我覺得這個建議非常好，Gearbox 不會太難，但又不簡單，剛好符合學習的甜蜜點。在爬梳合約的過程中真的是痛苦和快樂參半，但不管過程如何，結果是豐碩的。

## 參考資料

1. [Gearbox Protocol Document](https://docs.gearbox.finance/)
2. [Gearbox github](https://github.com/Gearbox-protocol)
3. [Product Evolution V2. Gearbox Protocol from 1 to 2: Going Further!](https://medium.com/gearbox-protocol/product-evolution-v2-gearbox-protocol-from-1-to-2-going-further-dcedf3b5d959)
