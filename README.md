# React Core
## Fiber
### 目前要解決什麼問題？
目前遇到的問題是當元件包裹的層次很深時，頂層元件可能因狀態改變須重新渲染，接著其下的子元件都要重新渲染，導致 call stack 很深，必須花很久時間執行，並且不能被中斷，導致 main thread 被長久 block 住，造成效能不佳、使用者無法對 UI 互動，也就是「執行太多事情又不能被中斷」的問題。

### 如何解決效能問題？
Fiber 就是為了解決效能問題而產生的，解決方法是讓以上過程「可以被中斷」，也就是說讓剛剛的過程不是一次重新更新完畢，而是分段完成、把工作切細，一次做完一個小工作即可，在每次做完一個小工作能有空檔去做其他的事情，這個小工作就稱為「Fiber」。

![Lin Clark - A Cartoon Intro to Fiber - React Conf 2017 ](https://blog.techbridge.cc/img/huli/fiber/cartoon.png)

圖片來源：[Lin Clark - A Cartoon Intro to Fiber - React Conf 2017 ](https://www.youtube.com/watch?v=ZCuYPiUIONs)

實際的作法就是每個小工作丟到 virtual stack frame 裡面，利用 js 模擬 call stack，讓 js 能擁有掌控權，而不是完全交由瀏覽器的所制訂的 js 運行機制來處理。

### 細談 Fiber 機制
Fiber 的工作分為兩階段

- render/reconciliation：找出需要改變的部份，可以被中斷、可以被重新執行，因此可能會呼叫多次
  - 對應的生命週期：componentWillMount、componentWillReceiveProps、shouldComponentUpdate、componentWillUpdate
- commit：將改變應用到 DOM 上，不能被中斷
  - 對應的生命週期：componentDidMount、componentDidUpdate、componentWillUnmount

### 副作用？
React 生命週期函數的呼叫次數和方式、順序會和以往不同，例如：若在 componentWillMount 呼叫 API 已取得資料，可能會因呼叫多次而 call 多次 API，浪費頻寬。解法是移到 componentDidMount 就能保證只呼叫一次。

比較好的做法是利用新的 API - getDerivedStateFromProps、getSnapshotBeforeUpdate。

### 未來展望？
- context API
- lifecycle 的改變
- time slicing：[Sneak Peek: Beyond React 16](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html) 這篇提到的 time slicing，把整個 App 的體驗變得更順暢。
- 非同步渲染 async rendering：[Update on Async Rendering](https://reactjs.org/blog/2018/03/27/update-on-async-rendering.html)，但會引起 componentWillMount、componentWillReceiveProps、componentWillUpdate 的一些問題，可改用 getDerivedStateFromProps、getSnapshotBeforeUpdate

### 實際程式碼的比較
待補

## 參考資料
- [淺談 React Fiber 及其對 lifecycles 造成的影響](https://blog.techbridge.cc/2018/03/31/react-fiber-and-lifecycle-change/#Fiber-%E5%88%B0%E5%BA%95%E6%98%AF%E4%BB%80%E9%BA%BC%EF%BC%9F)
