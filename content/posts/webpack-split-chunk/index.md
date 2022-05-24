---
title: "Webpack Split Chunk"
date: 2019-01-30T10:22:50+08:00
draft: false
---

雖然這個技術已經存在一段時間，但在現今大 Web 時代，若要增進網站的使用者經驗（UX）、效能、程式碼重複使用性……等等，依舊少不了 Code Splitting。

基本上此篇會 follow Webpack 官網上的 [Code Splitting - Guild](https://webpack.js.org/guides/code-splitting/) 並加上一些自己的想法來做介紹。

本篇適合給剛使用 Webpack 打包自己專案，並瞭解基本設定的開發者，如果你已熟悉它們的教程，那麼此篇其實可以考慮使用 cmd + w 或是 ctrl + w 進入彩蛋模式

### 什麼是 Code Splitting ?

從字面上來看 Code Splitting 即為將我們的 code（程式碼）給 Split（拆開），也就是說原本使用 Webpack 的打包出來的檔案，可能只有一個，但透過 Code Splitting 技術之後，可以將這一個檔案變成無數多個。

### 為什麼要這樣做？

隨著需求日漸增加的專案，檔案以及資源只會越來越龐大，試想如果你一進到 SWAG Web App ，立刻就載入一份 50 MB 的 JS 檔案會是什麼情況？在網路不穩時可能會導致 Loading 長達 5 ~ 10 秒。

以 UX 來說，正常人類能夠感知到時間的差別為 `0.1 秒`，如果介於 `0.1 秒 ~ 1 秒` 之間會被使用者發現有東西正在載入或是運行，但完全可以接受，但如果到了 `1 秒 ~ 5 秒` ，使用者會出現焦慮甚至懷疑程式出錯了，更不用提 5 秒之後，大部分使用者可能會選擇離開。

事實上使用者到 SWAG Web App 時，並不需要所有 JS 程式碼都一次載進來，也就是如果他到了這個頁面 https://app.swag.live 我們沒有必要將 https://app.swag.live/discover 的資源也載入進去。

需要什麼才拿什麼，就能有效降低 Loading 平均時間

## 實作

### 前置作業

首先創建一包資料夾、產生 package.json 並安裝 webpack、webpack-cli 及 lodash （lodash 是為了測試 code splitting 用的第三方套件）PS: Npm 6.4, Webpack 4.29

```bash
mkdir codesplitting
cd codesplitting
npm init -y
npm i webpack webpack-cli lodash
```

### 第一式：Mutiple Entry

```js
const path = require("path");
module.exports = {
  entry: {
    page1: "./src/page1.js",
    page2: "./src/page2.js",
  },
  mode: "development",
  output: {
    path: path.resolve(__dirname, "dist"),
    filename: "[name].bundle.js",
  },
};
```

新增 `src/page1.js`，並輸入

```js
console.log("this is page1");
```

新增 `src/page2.js`，並輸入

```js
console.log("this is page2");
```

接著在 command line 執行 `npx webpack` 就可以看到專案下多了一個 `dist` 資料夾，裡面放著打包好的檔案 `page1.bundle.js && page2.bundle.js`

將它門打開來看可能發現一堆看不懂的 code，不過分別搜尋 `console.log('this is page1') && console.log('this is page2')` 確實包含在裡面。

OK！這樣已經完成最初階的 CodeSplitting，我們可以透過前端 route 的手法讓使用者者切到 `/page1` 的時候載入 `page1.bundle.js` 就好， `/page2` 則是 `page2.bundle.js`

那如果讓 page1、2 一起使用 `lodash` 會發生什麼事？

```js
// /src/page1.js
import lodash from "lodash";
console.log("this is page1");
// /src/page2.js
import lodash from "lodash";
console.log("this is page2");
```

一樣使用 npx webpack 指令後會發現 page1、2.bundle.js 都變成一樣肥。

為了方便驗證，我們來使用 webpack-bundle-analyzer plugin。

1. npm install — save-dev webpack-bundle-analyzer
2. webpack.config.js 改成

```js
const path = require("path");
const BundleAnalyzerPlugin =
  require("webpack-bundle-analyzer").BundleAnalyzerPlugin;
module.exports = {
  entry: {
    page1: "./src/page1.js",
    page2: "./src/page2.js",
  },
  mode: "development",
  output: {
    path: path.resolve(__dirname, "dist"),
    filename: "[name].bundle.js",
  },
  plugins: [new BundleAnalyzerPlugin()],
};
```

再執行一次 npx webpack 時，可以在 http://127.0.0.1:8888 看到它分析的結果

![loadsh-chunk](/posts/webpack-split-chunk/not_split_lodash_chunk.jpg)

這樣是非常不合理，應該把 lodash.js 獨立成另外一個檔案（chunk)，讓 page1、2 共同使用

### 第二式：SplitChunksPlugin

為了避免同樣的 module 被重複打包，可以使用 webpack 提供的套件 SplitChunksPlugin，原本叫做 CommonsChunkPlugin，但在後來的版本被納入內建使用，只要在 `optimization.splitChunks` 設定即可

修改一下 webpack.config.js

```js
const path = require("path");
const BundleAnalyzerPlugin =
  require("webpack-bundle-analyzer").BundleAnalyzerPlugin;
module.exports = {
  entry: {
    page1: "./src/page1.js",
    page2: "./src/page2.js",
  },
  mode: "development",
  output: {
    path: path.resolve(__dirname, "dist"),
    filename: "[name].bundle.js",
  },
  optimization: {
    splitChunks: {
      chunks: "all",
    },
  },
  plugins: [new BundleAnalyzerPlugin()],
};
```

輸入 npx webpack

![loadsh-chunk](/posts/webpack-split-chunk/has_split_lodash_chunk.jpg)

可以看到 lodash 被獨立成單獨的檔案

~Async chunks VS Non-async chunks~

如果將 `chunks: 'all'` 改成 `chunks: 'async'` 再執行 `npx webpack` ，會發現結果又回到原本的樣子，為什麼呢？

因為使用 mutiple entry 載入的檔案會被歸類為 Non-async chunk，設定 `chunk:'async'` 後，SplitChunksPlugin 只會對 Async chunk 進行 Code-Splitting 的優化

那麼怎樣算是 Async chunk 呢？

### 第三式：Dynamic Imports

只要使用 `import(...)` 載入的檔案，webpack 會自動將它打包成一個獨立檔案，同時也被視為 Async chunk

它跟 mutiple entry 有相同的效果，但是卻可以在一個檔案內用 dynamic 方式 import 多個檔案，我們來修改範例

在 src/ 底下新增 index.js 檔案

```js
import("./page1.js");
import("./page2.js");
console.log("this is index.js");
```

修改 webpack.config.js

```js
entry: "./src/index.js";
```

執行 npx webpack

![loadsh-chunk](/posts/webpack-split-chunk/console_output_chunk_result.jpg)

![loadsh-chunk](/posts/webpack-split-chunk/has_split_chunk_all.jpg)

可以看到 page1、2 都被個別獨立成一個檔案，而且他們之間共用的套件（lodash) 也被獨立成例外一個檔名。

### 如何使用 React 簡單辦到

這裡貼上 React 文件上 Route 與 Code Splitting 的範例

![loadsh-chunk](/posts/webpack-split-chunk/react_code_split_sample.jpg)

簡單地來說，所有 component 都使用 import(...) 載入並掛上 Route 物件，這樣一來就可以根據不同頁面載入其所需的資源（ js 檔案）

### 總結

讀完以上簡介後可以發現設定 webpack code-splitting 並不難，只要簡單幾步就可以很有效的優化專案載入效能

然而 SplitChunks 裡面也有其他設定可以讓你將檔案切的更碎，此時就需評估是否有這個必要，會不會造成過多的 loading 反而影響了使用者體驗？

Reference

[Code-Splitting - React](https://reactjs.org/docs/code-splitting.html)
[Webpack 4 — Mysterious SplitChunks Plugin](https://medium.com/dailyjs/webpack-4-splitchunks-plugin-d9fbbe091fd0)
[SplitChunksPlugin | webpack](https://webpack.js.org/plugins/split-chunks-plugin/)
[Code Splitting | webpack](https://webpack.js.org/guides/code-splitting/)
