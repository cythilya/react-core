# Fiber Scheduler
這個模組的主要功能是處理任務的調度，也就是如何找到適合的時機去執行任務。

## 任務的優先順序
分為五種優先順序：最高、使用者定義的次高、一般、低、閒置，（[原始碼](https://github.com/facebook/react/blob/master/packages/scheduler/src/Scheduler.js#L21)）。

```javascript
var ImmediatePriority = 1; // 最高
var UserBlockingPriority = 2; // 使用者定義，次高
var NormalPriority = 3; // 一般
var LowPriority = 4; // 低
var IdlePriority = 5; // 閒置
```

