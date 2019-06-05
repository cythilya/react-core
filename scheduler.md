# Scheduler 任務調度器

這個模組的主要功能是處理任務的調度，也就是如何找到適合的時機去執行特定任務。

## 名詞解釋

- 任務：？
- 優先順序：？
- 過期時間：？
- 環狀佇列：？
- 時間切割（Time slicing）：？
- 任務的調度：？

## 基礎知識

### 環狀佇列（circular queue）

如下圖所示，元素排序的順序是由小到大，firstNode 指向第一個節點，而每個節點都會有 previous 和 next 兩個指標，分別指向前一個和後一個節點。因為這個佇列是循環的，所以第一個節點的 previous 會指向最後一個節點，而最後一個節點的 next 會指向第一個節點，這麼做的好處是在往前和往後的查找過程中，不需要特別判斷是否到頭或是到尾了，反正就是一直走就對了。

![circular queue](https://cythilya.github.io/assets/react-core/circular_queue_initial.png)

插入 1，後續修改如下。

- 由於 1 < 2，因此 1 成為 firstNode，1 的 previous 指向 98，1 的 next 指向 2。
- 2 的 previous 指向 1。
- 98 的 next 指向 1。

![circular queue](https://cythilya.github.io/assets/react-core/circular_queue_insert_1.png)

插入 5，後續修改如下。

- 由於 5 < 6，因此 5 的 previous 指向 4，5 的 next 指向 6。
- 6 的 previous 指向 5。
- 4 的 next 指向 5。

![circular queue](https://cythilya.github.io/assets/react-core/circular_queue_insert_5.png)

插入 99，後續修改如下。

- 由於沒有一個節點比 99 大，因此 99 要放在最後一個，99 的 previous 指向 98，99 的 next 指向 1。
- 1 的 previous 指向 99。
- 98 的 next 指向 99。

![circular queue](https://cythilya.github.io/assets/react-core/circular_queue_insert_99.png)

若要刪除節點，都只能刪除第一個，然後將原先第二個節點設為 firstNode，並重新配置前後節點的 next 與 previous。

下面來看虛擬碼，或參考（[原始碼](https://github.com/facebook/react/blob/master/packages/scheduler/src/Scheduler.js#L302)）。

```javascript
interface Node {
  id: number,
  priorityLevel: number,
  expirationTime: number,
  previous: Node,
  next: Node,
};
let firstNode = null; // 初始狀態是空的串列

function insertNode(newNode: Node) {
  if (!firstNode) {
    // 由於一開始是空的串列，因此第一個節點指向前一個和下一個的指標都會指向自己
    firstNode = newNode.previous = newNode.next = newNode;
  } else { // 串列不為空，因此要為新來的節點找適合的位置
    let node = null; // 記錄目前查找的節點
    let next = null; // 記錄會插到哪一個節點的前面

    do {
      if (newNode.expirationTime < node.expirationTime) {
        next = node;
        break;
      }
      node = node.next;
    } while (node !== firstNode) // 避免無限循環查找，因此若回到第一個節點就停下來
  }
}

function deleteFirstNode() {
 // ...
}
```

### 取得目前時間

`window.performance.now`

### 在下一幀時執行特定任務

`window.requestAnimationFrame`

### 發送和接收訊息

`window.MessageChannel`

## 任務調度的方法

### 第一步：決定任務的優先順序

分為五種優先順序：最高、使用者定義的次高、一般、低、閒置，（[原始碼](https://github.com/facebook/react/blob/master/packages/scheduler/src/Scheduler.js#L21)）。

```javascript
var ImmediatePriority = 1; // 最高
var UserBlockingPriority = 2; // 使用者定義，次高
var NormalPriority = 3; // 一般
var LowPriority = 4; // 低
var IdlePriority = 5; // 閒置
```

五種優先順序分別對應五種過期時間，（[原始碼](https://github.com/facebook/react/blob/master/packages/scheduler/src/Scheduler.js#L30)）。

在 V8 32-bit system 中，最大的正整數是 `2 ^ 30 - 1 = 1073741823`，用來設定「IDLE_PRIORITY」永不過期的過期時間。

```javascript
var maxSigned31BitInt = 1073741823;

// Times out immediately
var IMMEDIATE_PRIORITY_TIMEOUT = -1; // 立刻過期

// Eventually times out
var USER_BLOCKING_PRIORITY = 250; // 250ms 後過期
var NORMAL_PRIORITY_TIMEOUT = 5000; // 5000ms 後過期
var LOW_PRIORITY_TIMEOUT = 10000; // 10000ms 後過期

// Never times out
var IDLE_PRIORITY = maxSigned31BitInt; // 永不過期
```

當任務加入到環狀佇列（circular queue）以後，會經由計算 `Performance.now() + timeout` 來得到任務的過期時間，當目前時間愈來愈接近過期時間時，優先順序就會愈高。

### 第二步：依照過期時間更新優先順序

時間計算

使用 `Performance.now()` 取得目前時間，範例如下。

```javascript
getCurrentTime = function() {
  return Performance.now(); // 若不支持 Performance.now，則使用 Date.now() 作為 fallback
};
```

因此，在剛加入佇列的時候，一個任務的過期時間就是 `Performance.now() + timeout`（[原始碼](https://github.com/facebook/react/blob/master/packages/scheduler/src/Scheduler.js#L302)）。當隨著時間推進而接近過期時間時，優先順序就會跟著提高；當過期時間小於當前時間時，就會變成最高優先執行的任務。而必須被馬上執行。

任務會以環狀佇列的方式存放，並使用 `unstable_scheduleCallback` 將任務以「過期時間」來排序，愈接近過期時間的優先權愈高。

- priorityLevel
- callback
- deprecated_options

```javascript
function unstable_scheduleCallback(priorityLevel, callback, deprecated_options) {
  // ...
}
```

### 第三步：選擇任務並執行執行任務

將任務依照過期時間排序好後，準備開始執行任務，這也就是 `ensureHostCallbackIsScheduled` 的功能。

- 第一個任務應立即執行。
- 新加入的任務取代先前的第一個節點時，應停止先前的任務，改執行這個新加入的第一個任務。

`ensureHostCallbackIsScheduled` 用來在 idle 時選擇適合執行的任務。

## 第四步：在空閒時要做什麼？

利用瀏覽器在每一幀繪製完成的空閒時間做事情，亦即使用 `requestIdleCallback pollyfill` 在空閒時間做事情。目前是使用 `requestAnimationFrame` + `MessageChannel` 實作了 `requestIdleCallback`。

## References

- [React Scheduler 源碼詳解（1）](https://juejin.im/post/5c32c0c86fb9a049b7808665)

<!--
```javascript
```

，（[原始碼](https://github.com/facebook/react/blob/master/packages/scheduler/src/Scheduler.js#L)）。

-->
