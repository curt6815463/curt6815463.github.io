---
title: "React Fiber 初探"
date: 2022-05-22T16:43:34+08:00
draft: true
description: "Fiber 是個重要但平常又不太需要瞭解的架構，這篇文章帶你了解大概的流程以及使用原因"
cover:
  image: "./posts/react-fiber/react_fiber_cover.jpg"
  alt: "alt"
  caption: "caption"
---

React Fiber 廣義上可以代表整個新版架構，

而狹義上 Fiber 只是一個 `object`，這個 object 是 reconciler 藉由 react element 作為模板建立出來。

### 什麼是 React Element

平常寫 React Component 時，通常會在 render function 裡面使用 JSX 語法，而背後其實只是 JSX compiler 幫我們加入 `React.createElement`。

例如：

```js
  render(){
    return(
      <div onClick={() => null}>
        <div>Hello Element</div>
      </div>
    )
  }
```

會被轉成

```js
  render(){
    return React.createElement(
      'div',
      { onClick: myClick },
      React.createElement('div', null, 'Hello Element');
    );
  }
```

如果直接將 createElement 的結果 console.log 出來大概會長這樣

```jsx
  {
    $$typeof: Symbol(react.element),
    type: "div"
    key: null
    props: {
      children: {
        $$typeof: Symbol(react.element),
        type: "div",
        key: null,
        props: { children: 'Hello Element' },
        ref: null,
        ...
      },
      onClick: myClick,
    }
    ref: null
    ...
  }
```

它們就是 react element ，那 Fiber object 呢？

### Fiber Object 如何來的？

reconciler 會將 react element 作為參數，傳入 [`createFiberFromElement`](https://github.com/facebook/react/blob/ce13860281f833de8a3296b7a3dad9caced102e9/packages/react-reconciler/src/ReactFiber.new.js#L604) function 來產生 Fiber Object，而 Object 大概是長這樣。

```js
  {
    actualDuration: 0,
    actualStartTime: 3869.9000000953674,
    alternate: null,
    child: null,
    childLanes: 0,
    dependencies: null,
    elementType: "div",
    firstEffect: null,
    flags: 0,
    index: 0,
    key: null,
    lanes: 0,
    lastEffect: null,
    memoizedProps: null,
    memoizedState: null,
    mode: 9,
    nextEffect: null,
    pendingProps: null,
    ref: null,
    return: FiberNode,
    selfBaseDuration: 0,
    sibling: null,
    stateNode: div,
    tag: 5,
    treeBaseDuration: 0,
    type: "div",
    updateQueue: null,
  }
```

從這我們可以得知， react element 製造出 fiber object 。

### Fiber Object 要做什麼？

ㄧ個 Fiber Object 通常被稱作 a unit of work ，它是最小的工作單位，理由是他裡面存了很多工作 `資訊`，準備交給 reconciler 來處理，而這些 work 都是可被 `追蹤`、 `暫停` 、 `捨棄` 、 `安排`。

### Why

在開始進入 Fiber 架構前，我們應該先思考，為什麼 React 需要它以及它出現的原因？任何新的架構總是有他的目地或想要解決的問題，
而 react 真正想處理的地方則是 `reconciler`。

### 什麼是 reconciler

react 利用 tree 的資料結構，也可稱作 Virtual DOM，來代表各個 `節點` 的資料及狀態，

這裡的節點可先當作是：網頁的 DOM（其他平台不一定是 DOM，例如： iOS 上)

reconciler 會負責建立 tree（Virtual DOM），然後將其交給 renderer，請它用建立出來的 tree（Virtual DOM）畫出實際的介面給 User。

除此之外，如果畫面需要更動（更新、刪除、插入......等）時，reconciler 會再建立出另一個 tree（Virtual DOM），並透過 `diff` 算法來互相比較，藉此得知哪些節點需要更新，最後在交給 renderer 畫出實際介面。

### 什麼是 renderer

既然提到 renderer 再來簡單補充一下。

renderer 負責將 reconciler 建出的 tree（Virtual DOM） 畫出實際的介面給 User。

以網頁上來說，則是配合 DOM 來處理。

不同的環境會有不同的 renderer，iOS App 有自己的方式、Android App 也有自己的一套。

而 react 在網頁上使用的 renderer lib 則是 `react-dom` ，這也是在寫 react 時， `react` 與 `react-dom` 需要分開 import 的原因。

### 所以 reconciler 是有什麼問題？

前面提到 reconciler 會負責建 tree，在 fiber 架構前，建立的方式是使用 `遞迴`，這會使得程式產生：`call stack`

call stack 是個不錯的演算邏輯，但遇到 react 畫面的處理上，會有個問題：

`call stack 無法被中斷`，必須等到 stack 清空，程式才能繼續往下執行。

在 tree 很龐大的時候，call stack 會花太多時間執行，導致有些畫面的更新讓 User 感覺太慢卡卡的，尤其是 `動畫` 。

### Browser 更新邏輯

// 需要補齊

理論上只要每次畫面更新的頻率在 16 毫秒(ms)以內，大部的人類不會察覺到差異（≥ 60 FPS)，所以 react 使用 `requestIdleCallback` api 來處理不重要的 work，requestIdleCallback 裡會告訴你 Main Thread 還剩多少時間可處理 work，變成 0ms 時就把 work 給暫停掉甚至丟掉，假如這個 work 很重要，那就會放到 `requestAnimationFrame` 內。

### 怎麼解決？使用 Fiber 架構

在 Fiber 架構下的 reconciler， 使用 `while` 建立 `Linked List tree` ，while 的條件上除了確認是否還有 `下一個` 外，還會用個 flag： `shouldYield()` ，來確認 main thread 有沒有空。

> P.S. 這裡的 下一個 可能是 child、sibling

```js
// https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js?source=post_page---------------------------#L1126
// react source code

while (nextUnitOfWork !== null && !shouldYield()) {
  nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
}
```

這樣只要 main thread 太忙，就把 `shouldYield()` 變 true，中斷 tree 的建立，

直到 main thread 有空 `shouldYield()` 設回 false，再從中斷的地方繼續建 tree。

### 建立 Fiber Tree 的大致流程

React component 呼叫 `render()` 時，會產生 react element tree，這個 tree 每次都是新的並不會重複使用，然而 reconciler 自己會在內部建立另一種 tree，稱為 `Fiber Tree` 。

第一次 `render()` 時，reconciler 會將 react element tree 上的節點，使用 `while` 遍例，將其一一複製成 Fiber 節點， 形成一顆新的 Fiber Tree，這時這個 Tree 也叫做 `Current Tree` 。題外話，每一個 react element 的節點都是可以對應到產生出的 Fiber Node。

下一次 `render()` 時（update 時)，reconciler 會建出另一顆新的 tree 叫做 `Work in progress tree` ，在建立的過程會順便把需要更新的 Node `標記` 起來，這些被標記的 Node 會被串接成一條線性的 `List` ，稱為 `Effect List` （等等會用到）。

當 Work in progress 樹建好之後，此時兩棵樹的 reference 對調（ `Swap Tree` )

Current Tree → 成為 Work in progress Tree

Work in progress Tree → 成為 Current Tree

再下一次 render() ，Work in progress tree 不會再建立一次，會直接重複使用並且一樣找出需要更新的 Node 並標記起來形成 Effectt List ，如此一直重複下去。

### Fiber Tree 的結構

前面提到 Fiber Object 上存了一些工作資訊，其中裡面有三個屬性代表著他們之間的關係。

- child - 子節點
- sibling - 鄰居節點
- return - 父節點

![Untitled](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/2264e298-8e6e-408f-98eb-fb278dadcd25/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20220523%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20220523T025444Z&X-Amz-Expires=86400&X-Amz-Signature=a59788143f0af654b1291c49b87ba6a147a6b21db597c00b253813e7579556c7&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22Untitled.png%22&x-id=GetObject)

比較要注意的是，這是一個 single child 的結構，如果有很多子節點，那只有一個 child 會被當 child，其他變成 sibling。

每一個節點都是一個 work，reconciler 處理的順序上是

1. 子節點 (child)
2. 自己
3. sibling

```js
// 補圖
```

### 處理 Fiber Work

大致瞭解建樹的過程以及結構後，我們再來理解一下樹裡面的內容在做什麼，主要可以分為兩個步驟

1. render phase
2. commit phase

### Render Phase

我第一次讀 fiber 的時候，不小心把 Render Phase 跟 react component 裡的 `render()` 搞混，大家要注意這是不同的。

Render phase 的過程也稱為 `work loop` ，裡面執行 work 時是 `非同步` 的

在稍前有貼過他的部分 source code

```jsx
function workLoop(isYieldy) {
  if (!isYieldy) {
    // Flush work without yielding
    while (nextUnitOfWork !== null) {
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    }
  } else {
    // Flush asynchronous work until the deadline runs out of time.
    while (nextUnitOfWork !== null && !shouldYield()) {
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    }
  }
}
```

work loop 開始處理會將第一個 Fiber Node 做為參數丟入 `performUnitOfWork` function，然後裡面會接著呼叫 `beginWork` ，假如還有 child Node，beginWork 會 return child，將其丟回 work loop 繼續執行 performUnitOfWork，如果沒有 child，則是執行 `completeUnitOfWork` 。

```jsx
// code is from https://indepth.dev/posts/1008/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react
function completeUnitOfWork(workInProgress) {
  while (true) {
    let returnFiber = workInProgress.return;
    let siblingFiber = workInProgress.sibling;

    nextUnitOfWork = completeWork(workInProgress);

    if (siblingFiber !== null) {
      // If there is a sibling, return it
      // to perform work for this sibling
      return siblingFiber;
    } else if (returnFiber !== null) {
      // If there's no more work in this returnFiber,
      // continue the loop to complete the parent.
      workInProgress = returnFiber;
      continue;
    } else {
      // We've reached the root.
      return null;
    }
  }
}

function completeWork(workInProgress) {
  console.log("work completed for " + workInProgress.name);
  return null;
}
```

completeUnitOfWork 會先呼叫 `completeWork` 來完成 `自己` 的 work，但除此之外還要確認。

1. 是否有 `sibling` Node，如果有就 return 回 beginWork，beginWork 如果有收到 sibling 則 return 回 work loop，接著重複上面的流程（performUnitOfWork）
2. 如果沒有 sibling 則確認是否有 return (parent)

   2 - 1 如果有，則把當前 work reference 變成 return，並且在下次 loop 去 complete 它

   2 - 2 如果沒有，代表已經到了 root 節點可以結束 work loop 了

在 work loop 過程中最主要的任務，就是將有 `side effect` 的 Node 標記起來，並讓這些 Node 形成 `effect list` ，方法是透過 Fiber Object 上的 `effectTag` 、 `nextEffect` 屬性來紀錄。

side effect 有可能是呼叫 `lifecycle` 、 `setState` 、 `props 改變` 、 `手動更改 DOM 內容` ......等，這些都可能影響其他 component，因此無法在 `非同步` 的 render phase 完成。

### Commit 階段

首先到了這個階段，react 擁有三個東西

1. current tree

代表目前畫面的狀態樹，User 只會看到他的樣子

1. workInProgress tree

render 階段產出的狀態樹，準備用來更新畫面

1. effect list

從處理 workInProgress tree 中（也就是 render 階段），被標記出的 List。

Commit 階段主要就是透過遍歷 effect list，將最新狀態更新到 tree 上，並把更新的 tree 變成 current tree，這樣就可以省去遍歷沒有 side effect 的 Node 的時間，要注意這個階段都是 `同步` 進行，並且是真實的改變畫面上的 DOM，此外如果 component 有 lifecycle 也是在這個階段被執行

getSnapshotBeforeUpdate → componentWillUnmount → `更新畫面 DOM` → componentDidMount → componentDidUpdate （細節可參考此文的 [commit phase](https://indepth.dev/posts/1008/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react))

### 結論

Fiber 架構的好處在於使用 `非同步` 的方式處理每個 `work`，因此當瀏覽器太忙有些事要做的時候，可以將 work 隨時暫停或是丟掉，而我自己認為實作的關鍵在於將 `遞迴` 處理樹的部分，轉為使用 `while` 並且搭配 requestIdleCallback 來確認剩餘時間，當然裡面還有做了很多細節處理及架構優化。

一開始想寫這個主題是希望能夠有個簡單的流程，讓自己跟其他人能大致瞭解 Fiber，不過越寫就發現自己不懂的地方越多，沒有一開始想像的這麼簡單，即便已經寫到結語也是一樣 ＸＤ，希望這篇文章能幫助到你理解 Fiber，如果沒有很大機率是我寫的太糟了，建議您閱讀以下的 reference 會幫助很大的。

### Reference:

- [Inside Fiber: in-depth overview of the new reconciliation algorithm in React](https://indepth.dev/posts/1008/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react)
- [The how and why on React’s usage of linked list in Fiber to walk the component’s tree](https://indepth.dev/posts/1007/the-how-and-why-on-reacts-usage-of-linked-list-in-fiber-to-walk-the-components-tree)
- [React Doc - Reconciliation](https://reactjs.org/docs/reconciliation.html#gatsby-focus-wrapper)
- [React 開發者一定要知道的底層機制 — React Fiber Reconciler](https://medium.com/starbugs/react-開發者一定要知道的底層架構-react-fiber-c3ccd3b047a1)
- [What Is React Fiber? React.js Deep Dive #2](https://www.youtube.com/watch?v=0ympFIwQFJw)
- [Lin Clark - A Cartoon Intro to Fiber - React Conf 2017](https://www.youtube.com/watch?v=ZCuYPiUIONs)
- [SMOOSHCAST: React Fiber Deep Dive with Dan Abramov](https://www.youtube.com/watch?v=aS41Y_eyNrU)
