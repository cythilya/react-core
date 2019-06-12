# Scheduler 任務調度器

這個模組的主要功能是處理任務的調度，也就是如何找到適合的時機去執行特定任務。

## 名詞解釋

- 任務：要執行的工作，Fiber 將 React 渲染的工作切分為一個個任務，透過 Scheduler 來調度任務、控制渲染。
- 優先順序：每個任務會給予一個重要程度的標示，這個重要程度會對應一個過期時間。
- 過期時間：任務的到期時間，其中，過期時間 = 起始時間 + 依據優先順序所給定的過期時間。
- 環狀佇列：串連任務的資料結構。
- 時間切割（Time slicing）：？
- 任務的調度：？

## Scheduler 到底在幹嘛？

任務調度的方法，依序為

1. 決定任務的優先順序
2. 依照過期時間排列優先順序
3. 選擇任務並執行執行任務
4. 空閒時？

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
  id: number;
  priorityLevel: number;
  expirationTime: number;
  previous: Node;
  next: Node;
}
let firstNode = null; // 初始狀態是空的串列

function insertNode(newNode: Node) {
  if (!firstNode) {
    // 由於一開始是空的串列，因此第一個節點指向前一個和下一個的指標都會指向自己
    firstNode = newNode.previous = newNode.next = newNode;
  } else {
    // 串列不為空，因此要為新來的節點找適合的位置
    let node = null; // 記錄目前查找的節點
    let next = null; // 記錄會插到哪一個節點的前面

    do {
      if (newNode.expirationTime < node.expirationTime) {
        next = node;
        break;
      }
      node = node.next;
    } while (node !== firstNode); // 避免無限循環查找，因此若回到第一個節點就停下來

    if (next === firstNode) {
      // 新來的節點比第一個節點小，因此新來的節點要插在第一個節點之前，亦即新來的節點成為第一個節點
      firstNode = newNode;
    } else if (next === null) {
      // 找了一圈都沒有找到，代表新加入的節點比所有節點的 expirationTime 都大，因此要排在最後面
      next = firstNode;
    }

    // 重新安排節點的順序，目前狀況：previous -- newNode -- next
    let previous = next.previous;
    previous.next = next.previous = newNode;
    newNode.previous = previous;
    newNode.next = next;
  }
}

function deleteFirstNode() {
  let last = null;
  let next = null;

  if (firstNode === null) {
    // 佇列中沒有節點
    return false;
  }

  if (firstNode === firstNode.next) {
    // 佇列中只有一個節點
    firstNode = null;
  } else {
    // 指定第二個節點為新的 firstNode
    last = firstNode.previous; // 找到最後一個節點
    firstNode = last.next = next; // 重新配置第一個節點為原先的第二的節點，並將最後一個節點的 next 指向新的第一個節點
    firstNode.previous = next.previous = last; // 重新配置第一個節點的 previous 指向最後一個節點
  }
}
```

### 優先順序

任務會被指定一個優先順序，共分為五種優先順序：最高、使用者定義的次高、一般、低、閒置，（[原始碼](https://github.com/facebook/react/blob/master/packages/scheduler/src/Scheduler.js#L21)）。

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

### 時間

請參考 [unstable_scheduleCallback](https://github.com/facebook/react/blob/master/packages/scheduler/src/Scheduler.js#L302)，這裡主要做以下這些事情...

- 變數 [hasNativePerformanceNow](https://github.com/facebook/react/blob/master/packages/scheduler/src/forks/SchedulerHostConfig.default.js#L22) 用來判斷是否支援 `Performance.now(...)`。
- `localDate` 即是 [`Date`](https://github.com/facebook/react/blob/master/packages/scheduler/src/forks/SchedulerHostConfig.default.js#L28)，另存變數 `Date` 為 `localDate` 的目的是避免 `Date` 經過 polyfill 而有所衝突。
- 使用 `Performance.now(...)` 取得目前時間，若不支援 `Performance.now(...)`，則使用 `Date.now(...)` 作為 fallback（[原始碼](https://github.com/facebook/react/blob/master/packages/scheduler/src/forks/SchedulerHostConfig.default.js#L70)）。

```javascript
if (hasNativePerformanceNow) {
  const Performance = performance;
  getCurrentTime = function() {
    return Performance.now();
  };
} else {
  getCurrentTime = function() {
    return localDate.now();
  };
}
```

- 取得目前時間是為了計算起始時間（[startTime](https://github.com/facebook/react/blob/master/packages/scheduler/src/Scheduler.js#L307)）。
- 過期時間（[expirationTime](https://github.com/facebook/react/blob/master/packages/scheduler/src/Scheduler.js#L317)）等於 startTime + 依照先前設定的優先順序所給定的過期時間。例如，若瀏覽器已打開 1 秒，startTime 是 1000ms = 1000ms 且優先順序為 NormalPriority 給定的過期時間是 5000ms，因此過期時間就是 1000 + 5000 = 6000ms（[原始碼](https://github.com/facebook/react/blob/master/packages/scheduler/src/Scheduler.js#L319)）。

```javascript
switch (priorityLevel) {
  case ImmediatePriority:
    expirationTime = startTime + IMMEDIATE_PRIORITY_TIMEOUT;
    break;
  case UserBlockingPriority:
    expirationTime = startTime + USER_BLOCKING_PRIORITY;
    break;
  case IdlePriority:
    expirationTime = startTime + IDLE_PRIORITY;
    break;
  case LowPriority:
    expirationTime = startTime + LOW_PRIORITY_TIMEOUT;
    break;
  case NormalPriority:
  default:
    expirationTime = startTime + NORMAL_PRIORITY_TIMEOUT;
}
```

- 接著，任務就依照這個過期時間在環狀佇列中排序，尋找自己適合的位置（[原始碼](https://github.com/facebook/react/blob/master/packages/scheduler/src/Scheduler.js#L338)）。而原本在佇列中的任務，當目前時間愈來愈接近過期時間時，優先順序就會愈高；當過期時間小於當前時間時，就會變成最高優先執行的任務。而必須被馬上執行。

### 選取任務
選取佇列中的第一個任務。

將任務依照過期時間排序好後，就要決定選取任務來執行的時機點，這就是 [`scheduleHostCallbackIfNeeded`](https://github.com/facebook/react/blob/master/packages/scheduler/src/Scheduler.js#L57) 的功能。

選取任務的時機點

- idle
- 加入第一個任務時應立即執行。
- 新加入的任務取代先前的第一個節點時，應停止先前的任務，改執行這個新加入的第一個任務 [requestHostCallback（先前稱為 requestIdleCallback）](https://github.com/facebook/react/blob/master/packages/scheduler/src/forks/SchedulerHostConfig.default.js#L265)。

<!-- 以下尚未更新 -->
看這一段「requestHostCallback(也就是 requestIdleCallback) 這部分原始碼的實現比較複雜, 可以將其分解為以下幾個重要的步驟(有一些細節點可以看註釋):」

你不知道的 requestIdleCallback
https://www.jishuwen.com/d/2I9l/zh-tw

### 執行任務
requestHostCallback 的功用是在瀏覽器的每一幀的剩餘空閒時間內執行優先度相對較低的任務，也就是再一次選取佇列中的第一個任務。

## 在空閒時要做什麼？

利用瀏覽器在每一幀繪製完成的空閒時間做事情，亦即使用 `requestIdleCallback pollyfill` 在空閒時間做事情。目前是使用 `requestAnimationFrame` + `MessageChannel` 實作了 `requestIdleCallback`。

#### 在下一幀時執行特定任務

`window.requestAnimationFrame`

#### 發送和接收訊息

`window.MessageChannel`

## References

- [React Scheduler 源碼詳解（1）](https://juejin.im/post/5c32c0c86fb9a049b7808665)
- [你不知道的 requestIdleCallback](https://www.jishuwen.com/d/2I9l/zh-tw)

<!--
```javascript
```

，（[原始碼](https://github.com/facebook/react/blob/master/packages/scheduler/src/Scheduler.js#L)）。

-->
