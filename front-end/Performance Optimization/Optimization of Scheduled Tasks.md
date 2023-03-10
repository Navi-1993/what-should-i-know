在实际业务中，经常会遇到间隔多少秒或多少分钟执行一遍业务逻辑的情况。每个页面都可能散落着 `setInterval` 这种代码片段，它简单也易于理解，但这种代码一多，但实际上在js 运行中会产生许多个 `interval task`，这时候你如果留意 cpu 的资源占用，会发现随着 `interval task` 增多，`cpu` 资源被更多的占用，从而导致卡顿的情况——定时器是过度使用 CPU 的一个来源。

比较常见的业务场景有电商列表，每个组件都有自己独立的 tick 在进行更新倒计时，每 0.1 秒更新 item1 的 price，每 1 秒更新一次 item2 的 price，每 1 分钟更新 item3的库存，等等。如果 `interval` 运用不合理，就会导致页面性能低下的情况。

故抽空做此设计，让全局只有一个精确度为 0.1 秒的定时循环任务在执行，在这个循环中管理所有定时任务的添加，执行与移除。减少 cpu 的占用，优化性能，解决这种容易被忽略的问题。

  

**具体的性能差异可以看下面的实际运行对比**

-   当有 5 万个 `setInterval` 1000 毫秒定时任务处理 1 + 1 这种算数运算时的 cpu 消耗，此时 cpu 占用 31%上下浮动，但页面渲染缓慢，交互也会卡顿

![](https://cloud-minapp-44261.cloud.ifanrusercontent.com/1pZrK3sKkKM61kSF.png)

-   当有 5 万个 Ticker[1000].eventQueue 间隔 1 秒处理 1 + 1 这种算数运算时的 cpu 消耗，此时 cpu 占用 0.2%浮动

![](https://cloud-minapp-44261.cloud.ifanrusercontent.com/1pZrK3lGPdeVTVhs.png)

  

___

### 设计思路：

**循环的核心： engine**

**engine 的协议**

考虑到 `JavaScript` 的运行环境不一定都是浏览器 或 `node`，也有一些定制的运行时，所以抽象出了 `engine` 这个概念。

`engine` 将作为循环的核心 `api`，在 `Ticker` 运行过程中，不断地递归 `engine`。

`engine` 是可以更换的，但它需要满足一些标准，在这里我们以 `setTimeout` 与 `window.requestAnimationFrame` 的函数签名作为规范

1.  `engine` 是一个 `Function` ，它有 1 个必填参数及一到多个可选参数
    1.  必填参数需要是个 `Function`
2.  `engine` 执行后需要返回一个从 0 开始自增的整数 id

```ts
property) Ticker.#engine: ((handler: TimerHandler, timeout?: number | undefined, ...arguments: any[]) => number) & ((callback: FrameRequestCallback) => number)
```
(

  

**在前端业务场景中，定时任务一般分 2 种**

1.  前台任务：指应用正在使用时生效，当网页窗口被最小化，或小程序 `onHide` 时，定时任务就会停下来，这一类统称为前台任务，适合使用 `window.requestAnimationFrame` 这类 api 来做 `Ticker` 的 循坏 `engine`。它会在页面渲染每一帧时运行更新定时器的核心逻辑，当窗口被最小化，不会渲染新的帧，也即不会继续运行更新定时器的核心逻辑。
2.  持续任务：指应用不区分前后台，只要应用正在运行中，就会持续判断是否需要执行定时任务群，这种适用于使用 `setTimeout` 来作为 `Ticker` 循环的 `engine` 方法，即使处于后台，也会持续运行更新定时器的核心逻辑。
	1. 但是持续任务也可能因为浏览器的一些功能而导致持续任务不执行，比较常见的有 chrome 的 Native Window Occlusion ，该功能会使浏览器降低定时器的运行频率，这一点将在下文提出具体的解决方案和实践。

  

**任务队列：matrix**

1.  为了便于获取，减少操作时间，决定以 `object` 作为容器，定时任务的间隔时间 `interval` 作为容器的 `key`。每有对应 `interval` 的 定时任务 `task` 就添加到对应 `object[interval].queue `中，故描述它的数据类型为：

```ts
type Matrix = {
	[key: string]: {
		queue: Function[]
	}
}
```


  

**循环中判断是否执行定时任务**

我们已经确定默认使用 `setTimeout` 作为循环的核心，每次 `setTimeout` 执行完毕都会再执行递归 `setTimeout`，从而保证全局运行中的定时器只有 1 个 `setTimeout`。（当实例 Ticker 使用的 `engine` 是 `requestAnimationFrame` 时则是以帧执行循环，大概是 `16.66ms`，视设备屏幕的刷新率与软件容器允许的刷新率决定）

每次执行 `setTimeout` 任务，都会获取当前时间 `now`，若 `now - object[interval].updateTime >= interval` 则应该执行 `object[interval].queue` 中的所有任务，并更新 `object[interval].updateTime`

  

**循环休眠**

这里倒是没太多赘述的，功能与逻辑很简单。

1.  在需要让 `Ticker` 休息一段时间时，`clearTimeout` 这个唯一的定时器
2.  等待一段时间
3.  让 `Ticker` 继续 `run`

  

  

**指定的定时任务休眠**

当需要让指定任务在睡眠多少秒后再执行时，需要为这个定时任务对象配置一个 sleep 属性，作用为描述这个定时任务还要睡眠多久再执行

1.  若没有配置则可在循环中继续执行定时任务
2.  若配置了 `sleep` 则应该进行如下判断与更新
3.  当 `Ticker` 的 `setTimeout` 执行时，会获取当前设备时间 `now`，并减去对应定时任务的 `interval`，得到 `duration`
4.  `sleep -= duration`
5.  判断，若 `sleep <= 0` ，则应该执行该定时任务

  

### canvas 图例解析

todo

  

### 最佳实践

  

  

### 其他建议

- 使用中，可以利用 `es6` 模块的机制，先实例一个 `ticker`，再对其 `export`，达到单例的效果
```ts
export const setTimeoutTicker = new Ticker()
export const animateFrameTicker = new Ticker(window.requestAnimationFrame)
export const anyLoopEngineTicker = new Ticker(any what you need)
```

- 若是 **react18+** 或 **vue3+** 等语法，建议是基于该 **Ticker** class 再封装一层, 导出 **useSetTimeout** 与 **useAnimationFrame** 使用

### 供参考的 hooks

```ts
import Ticker from '../utils/ticker'
import type {Event} from '../utils/ticker'
const ticker = new Ticker()

type UseTick = {
  fn: Event['fn']
  interval: ReturnType<Date['getTime']>
  options?: {
    sleep?: Event['sleep']
    leading?: boolean
  }
}

export default function useTick(
  fn: UseTick['fn'],
  interval: UseTick['interval'],
  options?: UseTick['options']
): [() => ReturnType<Ticker['addTickEvent']>, () => ReturnType<Ticker['removeTickEvent']>] {
  const event = {
    fn,
    sleep: options?.sleep,
  }
  return [() => ticker.addTickEvent(event, interval), () => ticker.removeTickEvent(event)]
}

// usage
import useTick from '../../hooks/use-tick'
const [run, stop] = useTick(()=>{}, 1000)
run()
```

- 浏览器的 `Native Window Occlusion` 功能对定时器的功能影响
- (timer-throtting)[https://developer.chrome.com/blog/timer-throttling-in-chrome-88/]

> Native Window Occlusion is a feature in Google Chrome that allows the browser to intelligently manage the display of windows and tabs on the screen. This feature helps to prevent overlapping of windows and tabs, making it easier for users to navigate and work with multiple tabs and windows simultaneously.
> When a new window or tab is opened, Chrome automatically checks if it will overlap with any existing windows or tabs on the screen. If it does, Chrome will adjust the position of the new window or tab to prevent overlapping.
> This feature is especially useful for users who frequently work with multiple tabs and windows open at the same time. It helps to reduce clutter and improve productivity by ensuring that windows and tabs are always displayed in a clear and organized manner.

具体的影响：
当一个页面 tab 被最小化一段时间，该页面的定时器频率会被降低，如 1000ms 频率的定时器将被降低至 60000ms，页面失活越久，这个降频区间会越大。
那么当这种时候，定时器就无法按照我们预计的期望工作，导致业务可能出现异常。
例如 IM 软件的心跳检测，它就需要哪怕设备息屏或网页最小化后仍继续工作，若因为浏览器的 `Occlusion`  导致降频，就会影响 IM 的心跳检测，导致 web socket 断开，IM 也就停止了工作。

针对这种情况，可以考虑使用 `web worker` 创建一个线程，返回一个 `setTimeout` 给 `Ticker` 替换 `engine`。

```js
// 子线程
// uniapp 里使用微信小程序的 worker
// path: static/workers/tick-engine
worker.onMessage(interval => {
  const timer = setTimeout(() => worker.postMessage(timer), interval)
})
```

```ts
// 主线程 封装 worker engine 示例
import Ticker from '../utils/ticker'

function createNewWorker(scriptPath: string) {
  const worker = wx?.createWorker?.(scriptPath, {
    useExperimentalWorker: true,
  })
  worker?.onProcessKilled(() => {
    createNewWorker(scriptPath)
  })
  return worker
}
let worker = createNewWorker('static/workers/tick-engine.js')

const workerEngine = (callback: Function, interval: number) => {
  let id
  worker?.postMessage(interval)
  worker?.onMessage((id: number) => {
    callback()
    id = id
  })
  return id
}

const workerTicker = new Ticker({
  engine: workerEngine as any,
  destroyer: worker?.terminate,
})
```


```ts
// 封装为 hooks
import Ticker from '../utils/ticker'
import type {Event} from '../utils/ticker'

type UseWorkerTicker = {
  fn: Event['fn']
  interval: ReturnType<Date['getTime']>
  options?: {
    sleep?: Event['sleep']
    leading?: boolean
  }
}

export default function useWorkerTicker(
  fn: UseWorkerTicker['fn'],
  interval: UseWorkerTicker['interval'],
  options?: UseWorkerTicker['options']
): [() => ReturnType<Ticker['addTickEvent']>, () => ReturnType<Ticker['removeTickEvent']>] {
  const event = {
    fn,
    sleep: options?.sleep,
  }
  return [() => workerTicker.addTickEvent(event, interval), () => workerTicker.removeTickEvent(event)]
}


// 使用 hooks

import useWorkerTicker from '../../hooks/use-worker-ticker'
const [run, stop] = useWorkerTicker(()=>{}, 1000)
run()
```

### 相关源码

```ts
import delay from 'delay'

export type InitTickOptions = {
  engine?: typeof setTimeout & typeof requestAnimationFrame
  destroyer?: typeof clearTimeout & typeof cancelAnimationFrame
  interval?: number
}
export type Time = ReturnType<Date['getTime']>
export type Event = {
  fn: () => void
  sleep?: Time
}

export default class Ticker {
  #interval: Time = 100
  #engine: typeof setTimeout & typeof requestAnimationFrame = setTimeout
  #destroyer: typeof clearTimeout & typeof cancelAnimationFrame = clearTimeout
  public engineId: number = 0
  public eventMatrix: {
    [key: string]: {
      eventQueue: Event[]
      updateTime: Time
    }
  } = {}
  constructor(options?: InitTickOptions) {
    // options 装填完毕，tick 开始初始化
    this.#engine = options?.engine || this.#engine
    this.#destroyer = options?.destroyer || this.#destroyer
    this.#interval = options?.interval || this.#interval
    this.run()
  }
  // 获取当前设备时间
  getNow(): Time {
    return new Date().getTime()
  }
  // 添加标记事件
  addTickEvent(event: Event, interval: Time, leading?: boolean): void {
    if (leading === true) event.fn() // 若传入的 leading 为 true，在添加任务时会立刻执行一次
    if (this.eventMatrix[interval]) this.eventMatrix[interval].eventQueue.push(event)
    else
      this.eventMatrix[interval] = {
        eventQueue: [event],
        updateTime: this.getNow(),
      }
  }
  removeTickEvent(event: Event): void {
    Object.entries(this.eventMatrix).forEach(([interval, events]) => {
      const eventIndex = events.eventQueue.findIndex(item => item.fn === event.fn)
      if (eventIndex !== -1) events.eventQueue.splice(eventIndex, 1)
      // 结束后检查，若 length = 0 的 interval 队列，则删除对应队列属性，减少 #engine 循环的负担
      if (events.eventQueue.length === 0) delete this.eventMatrix[interval]
    })
  }
  run(): void {
    this.engineId = this.#engine(() => {
      const now = this.getNow()
      Object.entries(this.eventMatrix).forEach(([interval, events]) => {
        if (now - events.updateTime >= Number(interval)) {
          events.eventQueue.forEach(item => {
            if (typeof item?.sleep === 'number') {
              // 若存在 sleep，则更新 sleep 时间
              item.sleep = this.getNow() - events.updateTime
            }
            // 若不存在 sleep 或者 sleep 时间已经到了，执行事件
            if (!item?.sleep || item?.sleep <= 0) item.fn()
          })
          events.updateTime = now
        }
      })
      this.run()
    }, this.#interval)
  }
  async sleep(sleepTime: Time) {
    this.#destroyer(this.engineId)
    await delay(sleepTime)
    this.run()
  }
  stop() {
    this.#destroyer(this.engineId)
  }
}
```