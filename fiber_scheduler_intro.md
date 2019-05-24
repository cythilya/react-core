# Fiber Scheduler
這個模組的主要功能是處理任務的調度，也就是如何找到適合的時機去執行任務。

## 雙向佇列
待補。

## 任務的優先順序
分為五種優先順序：最高、使用者定義的次高、一般、低、閒置，（[原始碼](https://github.com/facebook/react/blob/master/packages/scheduler/src/Scheduler.js#L21)）。

```javascript
var ImmediatePriority = 1; // 最高
var UserBlockingPriority = 2; // 使用者定義，次高
var NormalPriority = 3; // 一般
var LowPriority = 4; // 低
var IdlePriority = 5; // 閒置
```

五種優先順序分別對應五種過期時間，（[原始碼](https://github.com/facebook/react/blob/master/packages/scheduler/src/Scheduler.js#L30)）。

```javascript
// Max 31 bit integer. The max integer size in V8 for 32-bit systems.
// Math.pow(2, 30) - 1
// 0b111111111111111111111111111111
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

當任務加入到雙向佇列以後，會經由計算 `performance.now() + timeout` 來得到任務的過期時間，當目前時間愈來愈接近過期時間時，優先順序就會愈高。


## References
- [React Scheduler 源碼詳解（1）](https://juejin.im/post/5c32c0c86fb9a049b7808665)

---
```javascript
```
