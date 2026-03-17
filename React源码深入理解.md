# React 源码深度剖析：从初始化到更新的完整流程

本文档以 React 18 源码为基础，通过三条主线深入剖析 React 内部机制：
1. **初始化流程**：`ReactDOM.createRoot(root).render(<App/>)` 如何启动整个应用
2. **更新流程**：`setState` 触发更新时 React 内部如何处理
3. **调度机制**：Scheduler 如何实现任务调度和时间切片

---

## 目录

- [第一部分：核心数据结构](#第一部分核心数据结构)
- [第二部分：初始化流程（createRoot + render）](#第二部分初始化流程createroot--render)
- [第三部分：更新流程（setState）](#第三部分更新流程setstate)
- [第四部分：调度机制（Scheduler）](#第四部分调度机制scheduler)
- [第五部分：完整调用链总结](#第五部分完整调用链总结)

---

## 第一部分：核心数据结构

在深入流程之前，先理解 React 内部的核心数据结构。

### 1.1 FiberRoot（应用根节点）

`FiberRoot` 是整个 React 应用的根节点容器，每次调用 `createRoot` 都会创建一个 `FiberRoot`。

```typescript
// 源码位置：react-reconciler/src/ReactFiberRoot.new.js

function FiberRootNode(
  containerInfo,    // DOM 容器节点
  tag,              // 根类型：LegacyRoot(0) 或 ConcurrentRoot(1)
  hydrate,          // 是否为 hydration 模式
  identifierPrefix,
  onRecoverableError,
) {
  this.tag = tag;
  this.containerInfo = containerInfo;      // 挂载的 DOM 容器
  this.pendingChildren = null;
  this.current = null;                     // 指向当前 Fiber 树的根节点（HostRoot Fiber）
  this.pingCache = null;
  this.finishedWork = null;                // 渲染完成待提交的 Fiber 树
  this.timeoutHandle = noTimeout;
  this.context = null;
  this.pendingContext = null;

  // 调度相关
  this.callbackNode = null;                // Scheduler 返回的任务节点
  this.callbackPriority = NoLane;          // 当前调度任务的优先级

  // Lane 优先级相关
  this.eventTimes = createLaneMap(NoLanes);      // 事件触发时间数组
  this.expirationTimes = createLaneMap(NoTimestamp); // 过期时间数组
  this.pendingLanes = NoLanes;             // 待处理的 lanes
  this.suspendedLanes = NoLanes;           // 挂起的 lanes
  this.pingedLanes = NoLanes;              // 被 ping 的 lanes
  this.expiredLanes = NoLanes;             // 已过期的 lanes
  this.finishedLanes = NoLanes;            // 已完成的 lanes

  this.entangledLanes = NoLanes;           // 纠缠的 lanes
  this.entanglements = createLaneMap(NoLanes);
}
```

**FiberRoot 的核心职责：**
- 保存整个应用的状态和配置
- 维护 `current` 指针指向当前渲染完成的 Fiber 树
- 管理所有更新的优先级（lanes）
- 与 Scheduler 调度器交互

### 1.2 Fiber（工作单元节点）

`Fiber` 是 React 内部对组件的抽象表示，每个组件、DOM 节点都对应一个 Fiber 节点。

```typescript
// 源码位置：react-reconciler/src/ReactFiber.new.js

function FiberNode(
  tag: WorkTag,        // 节点类型
  pendingProps: mixed, // 新的 props
  key: null | string,  // key 属性
  mode: TypeOfMode,    // 模式（Concurrent/Legacy）
) {
  // === 实例属性 ===
  this.tag = tag;              // 节点类型：FunctionComponent(0), ClassComponent(1), HostRoot(3), HostComponent(5)等
  this.key = key;              // 用于 diff 的 key
  this.elementType = null;     // 元素类型（memo/forwardRef 等包装前的类型）
  this.type = null;            // 组件函数/类/DOM 标签名
  this.stateNode = null;       // 对应的真实节点（DOM 节点或类组件实例）

  // === Fiber 树结构 ===
  this.return = null;          // 父 Fiber
  this.child = null;           // 第一个子 Fiber
  this.sibling = null;         // 下一个兄弟 Fiber
  this.index = 0;              // 在兄弟节点中的索引

  this.ref = null;             // ref 引用

  // === 状态相关 ===
  this.pendingProps = pendingProps;   // 本次渲染的 props
  this.memoizedProps = null;          // 上次渲染的 props
  this.updateQueue = null;            // 更新队列（类组件/HostRoot）
  this.memoizedState = null;          // 上次渲染的 state（函数组件这里存 Hook 链表）
  this.dependencies = null;           // context 依赖

  this.mode = mode;                   // 运行模式

  // === 副作用相关 ===
  this.flags = NoFlags;               // 副作用标记（Placement/Update/Deletion等）
  this.subtreeFlags = NoFlags;        // 子树的副作用标记
  this.deletions = null;              // 待删除的子节点

  // === 优先级相关 ===
  this.lanes = NoLanes;               // 本节点的更新优先级
  this.childLanes = NoLanes;          // 子树的更新优先级

  // === 双缓存 ===
  this.alternate = null;              // 指向另一棵树上的对应节点（current <-> workInProgress）
}
```

**Fiber 节点类型（WorkTag）：**

```typescript
// 源码位置：react-reconciler/src/ReactWorkTags.js

export const FunctionComponent = 0;        // 函数组件
export const ClassComponent = 1;           // 类组件
export const IndeterminateComponent = 2;   // 未确定类型的组件（首次渲染前）
export const HostRoot = 3;                 // Fiber 树的根节点
export const HostPortal = 4;               // Portal
export const HostComponent = 5;            // 原生 DOM 节点（div/span等）
export const HostText = 6;                 // 文本节点
export const Fragment = 7;                 // Fragment
export const Mode = 8;                     // StrictMode 等
export const ContextConsumer = 9;          // Context.Consumer
export const ContextProvider = 10;         // Context.Provider
export const ForwardRef = 11;              // forwardRef
export const Profiler = 12;                // Profiler
export const SuspenseComponent = 13;       // Suspense
export const MemoComponent = 14;           // memo
export const SimpleMemoComponent = 15;     // 简单 memo（无 compare 函数）
export const LazyComponent = 16;           // lazy
```

### 1.3 Hook（函数组件状态）

函数组件的状态通过 Hook 链表管理，挂在 Fiber 的 `memoizedState` 上。

```typescript
// 源码位置：react-reconciler/src/ReactFiberHooks.new.js

type Hook = {
  memoizedState: any,      // 当前 Hook 的状态值
  baseState: any,          // 基线状态（用于并发更新）
  baseQueue: Update<any> | null,  // 基线更新队列
  queue: UpdateQueue<any> | null, // 更新队列
  next: Hook | null,       // 下一个 Hook（形成链表）
};

type UpdateQueue<S, A> = {
  pending: Update<S, A> | null,        // 待处理的更新（环形链表）
  interleaved: Update<S, A> | null,    // 交错更新
  lanes: Lanes,                         // 队列的优先级
  dispatch: (A => void) | null,        // 触发更新的函数（setState）
  lastRenderedReducer: ((S, A) => S) | null,  // 上次渲染使用的 reducer
  lastRenderedState: S | null,         // 上次渲染的 state
};

type Update<S, A> = {
  lane: Lane,              // 更新的优先级
  action: A,               // setState 传入的值或函数
  hasEagerState: boolean,  // 是否已计算 eagerState
  eagerState: S | null,    // 提前计算的 state
  next: Update<S, A>,      // 下一个更新（环形链表）
};
```

### 1.4 Lane（优先级模型）

React 18 使用 Lane 模型管理优先级，用位掩码表示。

```typescript
// 源码位置：react-reconciler/src/ReactFiberLane.new.js

export type Lanes = number;
export type Lane = number;

export const TotalLanes = 31;

export const NoLanes: Lanes =                        0b0000000000000000000000000000000;
export const NoLane: Lane =                          0b0000000000000000000000000000000;

export const SyncLane: Lane =                        0b0000000000000000000000000000001;  // 同步优先级（最高）

export const InputContinuousHydrationLane: Lane =    0b0000000000000000000000000000010;
export const InputContinuousLane: Lane =             0b0000000000000000000000000000100;  // 连续输入（如拖拽）

export const DefaultHydrationLane: Lane =            0b0000000000000000000000000001000;
export const DefaultLane: Lane =                     0b0000000000000000000000000010000;  // 默认优先级

const TransitionLanes: Lanes =                       0b0000000001111111111111111000000;  // Transition 优先级
const RetryLanes: Lanes =                            0b0000111110000000000000000000000;  // 重试优先级

export const IdleLane: Lane =                        0b0100000000000000000000000000000;  // 空闲优先级（最低）
export const OffscreenLane: Lane =                   0b1000000000000000000000000000000;  // 离屏优先级
```

**优先级从高到低：** SyncLane > InputContinuousLane > DefaultLane > TransitionLanes > RetryLanes > IdleLane

---

## 第二部分：初始化流程（createRoot + render）

当我们调用 `ReactDOM.createRoot(container).render(<App />)` 时，React 内部执行了以下流程：

### 2.1 整体流程图

```
ReactDOM.createRoot(container)
    │
    ├─> createRoot()                     // 验证容器、创建选项
    │       │
    │       └─> createContainer()        // 创建容器
    │               │
    │               └─> createFiberRoot()  // 创建 FiberRoot
    │                       │
    │                       ├─> new FiberRootNode()      // 创建 FiberRoot 节点
    │                       │
    │                       ├─> createHostRootFiber()    // 创建 HostRoot Fiber
    │                       │
    │                       ├─> root.current = fiber     // FiberRoot.current -> HostRootFiber
    │                       │   fiber.stateNode = root   // HostRootFiber.stateNode -> FiberRoot
    │                       │
    │                       └─> initializeUpdateQueue()  // 初始化更新队列
    │
    ├─> markContainerAsRoot()            // 在 DOM 上打标记
    │
    ├─> listenToAllSupportedEvents()     // 事件委托绑定
    │
    └─> return new ReactDOMRoot(root)    // 返回 Root 对象
            │
            └─> root.render(<App />)     // 调用 render 方法
                    │
                    └─> updateContainer()  // 触发更新
                            │
                            ├─> requestUpdateLane()      // 获取更新优先级
                            │
                            ├─> createUpdate()           // 创建更新对象
                            │
                            ├─> enqueueUpdate()          // 将更新加入队列
                            │
                            └─> scheduleUpdateOnFiber()  // 调度更新
```

### 2.2 createRoot 详解

```typescript
// 源码位置：react-dom/src/client/ReactDOMRoot.js

export function createRoot(
  container: Element | DocumentFragment,
  options?: CreateRootOptions,
): RootType {
  // 1. 验证容器是否有效
  if (!isValidContainer(container)) {
    throw new Error('createRoot(...): Target container is not a DOM element.');
  }

  // 2. 处理选项
  let isStrictMode = false;
  let concurrentUpdatesByDefaultOverride = false;
  let identifierPrefix = '';
  let onRecoverableError = defaultOnRecoverableError;
  let transitionCallbacks = null;

  if (options !== null && options !== undefined) {
    if (options.unstable_strictMode === true) {
      isStrictMode = true;
    }
    // ... 其他选项处理
  }

  // 3. 创建 FiberRoot（核心）
  const root = createContainer(
    container,          // DOM 容器
    ConcurrentRoot,     // 并发模式标记（值为 1）
    null,               // hydrationCallbacks
    isStrictMode,
    concurrentUpdatesByDefaultOverride,
    identifierPrefix,
    onRecoverableError,
    transitionCallbacks,
  );

  // 4. 在 DOM 容器上打标记
  markContainerAsRoot(root.current, container);

  // 5. 绑定事件委托
  const rootContainerElement =
    container.nodeType === COMMENT_NODE
      ? container.parentNode
      : container;
  listenToAllSupportedEvents(rootContainerElement);

  // 6. 返回 ReactDOMRoot 实例
  return new ReactDOMRoot(root);
}
```

### 2.3 createFiberRoot 详解

```typescript
// 源码位置：react-reconciler/src/ReactFiberRoot.new.js

export function createFiberRoot(
  containerInfo: any,
  tag: RootTag,           // ConcurrentRoot = 1
  hydrate: boolean,       // false
  initialChildren: ReactNodeList,
  hydrationCallbacks: null | SuspenseHydrationCallbacks,
  isStrictMode: boolean,
  concurrentUpdatesByDefaultOverride: null | boolean,
  identifierPrefix: string,
  onRecoverableError: null | ((error: mixed) => void),
  transitionCallbacks: null | TransitionTracingCallbacks,
): FiberRoot {
  // 1. 创建 FiberRoot 节点
  const root: FiberRoot = new FiberRootNode(
    containerInfo,
    tag,
    hydrate,
    identifierPrefix,
    onRecoverableError,
  );

  // 2. 创建 HostRoot Fiber（Fiber 树的根节点）
  const uninitializedFiber = createHostRootFiber(
    tag,
    isStrictMode,
    concurrentUpdatesByDefaultOverride,
  );

  // 3. 建立双向连接
  root.current = uninitializedFiber;           // FiberRoot -> HostRootFiber
  uninitializedFiber.stateNode = root;         // HostRootFiber -> FiberRoot

  // 4. 初始化 HostRootFiber 的状态
  const initialState: RootState = {
    element: initialChildren,  // null（首次渲染）
    isDehydrated: hydrate,
    cache: initialCache,
    transitions: null,
  };
  uninitializedFiber.memoizedState = initialState;

  // 5. 初始化更新队列
  initializeUpdateQueue(uninitializedFiber);
  // 更新队列结构：
  // {
  //   baseState: fiber.memoizedState,
  //   firstBaseUpdate: null,
  //   lastBaseUpdate: null,
  //   shared: {
  //     pending: null,      // 环形链表
  //     interleaved: null,
  //     lanes: NoLanes,
  //   },
  //   effects: null,
  // }

  return root;
}
```

### 2.4 createHostRootFiber 详解

```typescript
// 源码位置：react-reconciler/src/ReactFiber.new.js

export function createHostRootFiber(
  tag: RootTag,
  isStrictMode: boolean,
  concurrentUpdatesByDefaultOverride: null | boolean,
): Fiber {
  let mode;

  if (tag === ConcurrentRoot) {
    // 并发模式
    mode = ConcurrentMode;
    if (isStrictMode === true) {
      mode |= StrictLegacyMode;
      if (enableStrictEffects) {
        mode |= StrictEffectsMode;
      }
    }
    // ... 其他模式处理
  } else {
    // Legacy 模式
    mode = NoMode;
  }

  // 创建 Fiber 节点
  return createFiber(HostRoot, null, null, mode);
  // HostRoot = 3，表示这是 Fiber 树的根节点
}

const createFiber = function(
  tag: WorkTag,
  pendingProps: mixed,
  key: null | string,
  mode: TypeOfMode,
): Fiber {
  return new FiberNode(tag, pendingProps, key, mode);
};
```

### 2.5 render 方法详解

```typescript
// 源码位置：react-dom/src/client/ReactDOMRoot.js

ReactDOMRoot.prototype.render = function(children: ReactNodeList): void {
  const root = this._internalRoot;
  if (root === null) {
    throw new Error('Cannot update an unmounted root.');
  }
  // 调用 updateContainer 触发更新
  updateContainer(children, root, null, null);
};
```

### 2.6 updateContainer 详解

```typescript
// 源码位置：react-reconciler/src/ReactFiberReconciler.new.js

export function updateContainer(
  element: ReactNodeList,      // <App /> 组件
  container: OpaqueRoot,       // FiberRoot
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): Lane {
  const current = container.current;  // 获取 HostRoot Fiber

  // 1. 获取事件时间和更新优先级
  const eventTime = requestEventTime();
  const lane = requestUpdateLane(current);

  // 2. 获取 context（通常为空对象）
  const context = getContextForSubtree(parentComponent);
  if (container.context === null) {
    container.context = context;
  } else {
    container.pendingContext = context;
  }

  // 3. 创建更新对象
  const update = createUpdate(eventTime, lane);
  // update 结构：
  // {
  //   eventTime,
  //   lane,
  //   tag: UpdateState,     // 0
  //   payload: null,        // 下面会设置为 {element}
  //   callback: null,
  //   next: null,
  // }

  // 4. 设置 payload（要渲染的元素）
  update.payload = {element};  // element = <App />

  // 5. 将更新加入队列（环形链表）
  enqueueUpdate(current, update, lane);

  // 6. 调度更新（核心！）
  const root = scheduleUpdateOnFiber(current, lane, eventTime);
  if (root !== null) {
    entangleTransitions(root, current, lane);
  }

  return lane;
}
```

### 2.7 scheduleUpdateOnFiber 详解（调度入口）

这是 React 更新调度的核心入口函数，无论是初始化还是后续更新都会调用它。

```typescript
// 源码位置：react-reconciler/src/ReactFiberWorkLoop.new.js

export function scheduleUpdateOnFiber(
  fiber: Fiber,
  lane: Lane,
  eventTime: number,
): FiberRoot | null {
  // 1. 检查嵌套更新（防止死循环）
  checkForNestedUpdates();

  // 2. 从当前 Fiber 向上收集 lanes 到根节点
  const root = markUpdateLaneFromFiberToRoot(fiber, lane);
  if (root === null) {
    return null;
  }

  // 3. 在根节点标记有待处理的更新
  markRootUpdated(root, lane, eventTime);

  // 4. 判断是否在渲染阶段触发的更新
  if (
    (executionContext & RenderContext) !== NoLanes &&
    root === workInProgressRoot
  ) {
    // 渲染阶段更新，特殊处理
    workInProgressRootRenderPhaseUpdatedLanes = mergeLanes(
      workInProgressRootRenderPhaseUpdatedLanes,
      lane,
    );
  } else {
    // 5. 正常更新，确保根节点被调度
    ensureRootIsScheduled(root, eventTime);

    // 6. 同步优先级且不在任何上下文中，立即执行
    if (
      lane === SyncLane &&
      executionContext === NoContext &&
      (fiber.mode & ConcurrentMode) === NoMode
    ) {
      resetRenderTimer();
      flushSyncCallbacksOnlyInLegacyMode();
    }
  }

  return root;
}
```

### 2.8 markUpdateLaneFromFiberToRoot 详解

```typescript
// 源码位置：react-reconciler/src/ReactFiberWorkLoop.new.js

function markUpdateLaneFromFiberToRoot(
  sourceFiber: Fiber,
  lane: Lane,
): FiberRoot | null {
  // 1. 更新当前 Fiber 的 lanes
  sourceFiber.lanes = mergeLanes(sourceFiber.lanes, lane);
  let alternate = sourceFiber.alternate;
  if (alternate !== null) {
    alternate.lanes = mergeLanes(alternate.lanes, lane);
  }

  // 2. 向上遍历，更新所有祖先节点的 childLanes
  let node = sourceFiber;
  let parent = sourceFiber.return;
  while (parent !== null) {
    parent.childLanes = mergeLanes(parent.childLanes, lane);
    alternate = parent.alternate;
    if (alternate !== null) {
      alternate.childLanes = mergeLanes(alternate.childLanes, lane);
    }
    node = parent;
    parent = parent.return;
  }

  // 3. 到达根节点，返回 FiberRoot
  if (node.tag === HostRoot) {
    const root: FiberRoot = node.stateNode;
    return root;
  } else {
    return null;
  }
}
```

### 2.9 ensureRootIsScheduled 详解

```typescript
// 源码位置：react-reconciler/src/ReactFiberWorkLoop.new.js

function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  const existingCallbackNode = root.callbackNode;

  // 1. 检查是否有任务饿死，标记为过期
  markStarvedLanesAsExpired(root, currentTime);

  // 2. 获取下一个要处理的 lanes
  const nextLanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );

  // 3. 没有待处理的 lanes，取消现有回调
  if (nextLanes === NoLanes) {
    if (existingCallbackNode !== null) {
      cancelCallback(existingCallbackNode);
    }
    root.callbackNode = null;
    root.callbackPriority = NoLane;
    return;
  }

  // 4. 获取最高优先级
  const newCallbackPriority = getHighestPriorityLane(nextLanes);
  const existingCallbackPriority = root.callbackPriority;

  // 5. 优先级相同，复用现有任务
  if (existingCallbackPriority === newCallbackPriority) {
    return;
  }

  // 6. 有更高优先级的任务，取消现有任务
  if (existingCallbackNode != null) {
    cancelCallback(existingCallbackNode);
  }

  // 7. 根据优先级选择调度方式
  let newCallbackNode;
  if (newCallbackPriority === SyncLane) {
    // 同步优先级：加入同步队列
    if (root.tag === LegacyRoot) {
      scheduleLegacySyncCallback(performSyncWorkOnRoot.bind(null, root));
    } else {
      scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root));
    }
    // 使用微任务调度
    scheduleMicrotask(() => {
      if (executionContext === NoContext) {
        flushSyncCallbacks();
      }
    });
    newCallbackNode = null;
  } else {
    // 非同步优先级：使用 Scheduler 调度
    let schedulerPriorityLevel;
    switch (lanesToEventPriority(nextLanes)) {
      case DiscreteEventPriority:
        schedulerPriorityLevel = ImmediatePriority;
        break;
      case ContinuousEventPriority:
        schedulerPriorityLevel = UserBlockingPriority;
        break;
      case DefaultEventPriority:
        schedulerPriorityLevel = NormalPriority;
        break;
      case IdleEventPriority:
        schedulerPriorityLevel = IdlePriority;
        break;
      default:
        schedulerPriorityLevel = NormalPriority;
        break;
    }
    // 调用 Scheduler 的 scheduleCallback
    newCallbackNode = scheduleCallback(
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root),
    );
  }

  // 8. 保存回调节点和优先级
  root.callbackPriority = newCallbackPriority;
  root.callbackNode = newCallbackNode;
}
```

### 2.10 初始化完成后的数据结构

初始化完成后，内存中的结构如下：

```
┌─────────────────────────────────────────────────────────────┐
│                        FiberRoot                             │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ containerInfo: <div id="root">                          ││
│  │ tag: ConcurrentRoot (1)                                 ││
│  │ current: ───────────────────────────────────────┐       ││
│  │ pendingLanes: DefaultLane                       │       ││
│  │ callbackNode: Scheduler Task                    │       ││
│  │ callbackPriority: DefaultLane                   │       ││
│  └─────────────────────────────────────────────────│───────┘│
└────────────────────────────────────────────────────│────────┘
                                                     │
                                                     ▼
                                    ┌───────────────────────────┐
                                    │     HostRoot Fiber        │
                                    │  ┌─────────────────────┐  │
                                    │  │ tag: HostRoot (3)   │  │
                                    │  │ stateNode: FiberRoot│  │
                                    │  │ memoizedState: {    │  │
                                    │  │   element: <App />  │  │
                                    │  │ }                   │  │
                                    │  │ updateQueue: {...}  │  │
                                    │  │ child: null (待构建)│  │
                                    │  └─────────────────────┘  │
                                    └───────────────────────────┘
```

---

## 第三部分：更新流程（setState）

当组件调用 `setState` 时，React 内部的执行流程如下：

### 3.1 整体流程图

```
setState(newValue)
    │
    └─> dispatchSetState(fiber, queue, action)
            │
            ├─> requestUpdateLane()              // 获取更新优先级
            │
            ├─> 创建 Update 对象
            │
            ├─> isRenderPhaseUpdate()?
            │       │
            │       ├─ YES ─> enqueueRenderPhaseUpdate()  // 渲染阶段更新
            │       │
            │       └─ NO ─> enqueueUpdate()     // 正常更新
            │               │
            │               ├─> eagerState 优化   // 尝试提前计算
            │               │       │
            │               │       └─ 相同则 return（不触发更新）
            │               │
            │               └─> scheduleUpdateOnFiber()  // 调度更新
            │                       │
            │                       ├─> markUpdateLaneFromFiberToRoot()
            │                       │
            │                       ├─> markRootUpdated()
            │                       │
            │                       └─> ensureRootIsScheduled()
            │                               │
            │                               └─> scheduleCallback() // Scheduler 调度
            │                                       │
            │                                       └─> performConcurrentWorkOnRoot()
            │                                               │
            │                                               ├─> renderRootSync/Concurrent()
            │                                               │       │
            │                                               │       └─> workLoopSync/Concurrent()
            │                                               │               │
            │                                               │               └─> performUnitOfWork()
            │                                               │                       │
            │                                               │                       ├─> beginWork()
            │                                               │                       │
            │                                               │                       └─> completeUnitOfWork()
            │                                               │
            │                                               └─> commitRoot()
            │                                                       │
            │                                                       ├─> commitBeforeMutationEffects()
            │                                                       │
            │                                                       ├─> commitMutationEffects()
            │                                                       │
            │                                                       └─> commitLayoutEffects()
```

### 3.2 useState 的 mount 阶段

```typescript
// 源码位置：react-reconciler/src/ReactFiberHooks.new.js

function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  // 1. 创建 Hook 对象并加入链表
  const hook = mountWorkInProgressHook();

  // 2. 处理初始值
  if (typeof initialState === 'function') {
    initialState = initialState();
  }
  hook.memoizedState = hook.baseState = initialState;

  // 3. 创建更新队列
  const queue: UpdateQueue<S, BasicStateAction<S>> = {
    pending: null,
    interleaved: null,
    lanes: NoLanes,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: initialState,
  };
  hook.queue = queue;

  // 4. 创建 dispatch 函数（绑定 fiber 和 queue）
  const dispatch: Dispatch<BasicStateAction<S>> = (queue.dispatch =
    dispatchSetState.bind(null, currentlyRenderingFiber, queue));

  return [hook.memoizedState, dispatch];
}

function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null,
  };

  if (workInProgressHook === null) {
    // 第一个 Hook，挂到 Fiber 的 memoizedState 上
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // 后续 Hook，追加到链表末尾
    workInProgressHook = workInProgressHook.next = hook;
  }
  return workInProgressHook;
}
```

### 3.3 dispatchSetState 详解

```typescript
// 源码位置：react-reconciler/src/ReactFiberHooks.new.js

function dispatchSetState<S, A>(
  fiber: Fiber,
  queue: UpdateQueue<S, A>,
  action: A,
) {
  // 1. 获取更新优先级
  const lane = requestUpdateLane(fiber);

  // 2. 创建更新对象
  const update: Update<S, A> = {
    lane,
    action,
    hasEagerState: false,
    eagerState: null,
    next: null,
  };

  // 3. 渲染阶段更新（特殊处理）
  if (isRenderPhaseUpdate(fiber)) {
    enqueueRenderPhaseUpdate(queue, update);
  } else {
    // 4. 正常更新
    enqueueUpdate(fiber, queue, update, lane);

    const alternate = fiber.alternate;

    // 5. eagerState 优化
    // 当 fiber 没有待处理的更新时，可以提前计算新 state
    if (
      fiber.lanes === NoLanes &&
      (alternate === null || alternate.lanes === NoLanes)
    ) {
      const lastRenderedReducer = queue.lastRenderedReducer;
      if (lastRenderedReducer !== null) {
        try {
          const currentState: S = queue.lastRenderedState;
          const eagerState = lastRenderedReducer(currentState, action);
          update.hasEagerState = true;
          update.eagerState = eagerState;

          // 如果新 state 和旧 state 相同，跳过更新
          if (is(eagerState, currentState)) {
            return;
          }
        } catch (error) {
          // 忽略错误，让正常渲染流程处理
        }
      }
    }

    // 6. 调度更新
    const eventTime = requestEventTime();
    const root = scheduleUpdateOnFiber(fiber, lane, eventTime);

    if (root !== null) {
      entangleTransitionUpdate(root, queue, lane);
    }
  }
}
```

### 3.4 useState 的 update 阶段

```typescript
// 源码位置：react-reconciler/src/ReactFiberHooks.new.js

function updateState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  return updateReducer(basicStateReducer, initialState);
}

function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: (I) => S,
): [S, Dispatch<A>] {
  // 1. 获取当前 Hook
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;

  queue.lastRenderedReducer = reducer;

  const current: Hook = currentHook;
  let baseQueue = current.baseQueue;
  const pendingQueue = queue.pending;

  // 2. 合并 pending 队列到 baseQueue
  if (pendingQueue !== null) {
    if (baseQueue !== null) {
      // 合并两个环形链表
      const baseFirst = baseQueue.next;
      const pendingFirst = pendingQueue.next;
      baseQueue.next = pendingFirst;
      pendingQueue.next = baseFirst;
    }
    current.baseQueue = baseQueue = pendingQueue;
    queue.pending = null;
  }

  // 3. 处理更新队列
  if (baseQueue !== null) {
    const first = baseQueue.next;
    let newState = current.baseState;
    let newBaseState = null;
    let newBaseQueueFirst = null;
    let newBaseQueueLast = null;
    let update = first;

    do {
      const updateLane = update.lane;

      if (!isSubsetOfLanes(renderLanes, updateLane)) {
        // 优先级不够，跳过此更新，保留到下次
        const clone: Update<S, A> = {
          lane: updateLane,
          action: update.action,
          hasEagerState: update.hasEagerState,
          eagerState: update.eagerState,
          next: null,
        };
        if (newBaseQueueLast === null) {
          newBaseQueueFirst = newBaseQueueLast = clone;
          newBaseState = newState;
        } else {
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }
        currentlyRenderingFiber.lanes = mergeLanes(
          currentlyRenderingFiber.lanes,
          updateLane,
        );
        markSkippedUpdateLanes(updateLane);
      } else {
        // 优先级足够，处理此更新
        if (newBaseQueueLast !== null) {
          const clone: Update<S, A> = {
            lane: NoLane,
            action: update.action,
            hasEagerState: update.hasEagerState,
            eagerState: update.eagerState,
            next: null,
          };
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }

        // 计算新 state
        if (update.hasEagerState) {
          newState = update.eagerState;
        } else {
          const action = update.action;
          newState = reducer(newState, action);
        }
      }
      update = update.next;
    } while (update !== null && update !== first);

    if (newBaseQueueLast === null) {
      newBaseState = newState;
    } else {
      newBaseQueueLast.next = newBaseQueueFirst;
    }

    // 4. 检查 state 是否变化
    if (!is(newState, hook.memoizedState)) {
      markWorkInProgressReceivedUpdate();
    }

    // 5. 更新 Hook 状态
    hook.memoizedState = newState;
    hook.baseState = newBaseState;
    hook.baseQueue = newBaseQueueLast;
    queue.lastRenderedState = newState;
  }

  const dispatch: Dispatch<A> = queue.dispatch;
  return [hook.memoizedState, dispatch];
}
```

### 3.5 渲染阶段详解

#### 3.5.1 performConcurrentWorkOnRoot

```typescript
// 源码位置：react-reconciler/src/ReactFiberWorkLoop.new.js

function performConcurrentWorkOnRoot(root, didTimeout) {
  // 1. 重置相关状态
  currentEventTime = NoTimestamp;
  currentEventTransitionLane = NoLanes;

  const originalCallbackNode = root.callbackNode;

  // 2. 执行 passive effects（useEffect）
  const didFlushPassiveEffects = flushPassiveEffects();
  if (didFlushPassiveEffects) {
    if (root.callbackNode !== originalCallbackNode) {
      return null;
    }
  }

  // 3. 获取要处理的 lanes
  let lanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );
  if (lanes === NoLanes) {
    return null;
  }

  // 4. 判断是否需要时间切片
  const shouldTimeSlice =
    !includesBlockingLane(root, lanes) &&
    !includesExpiredLane(root, lanes) &&
    !didTimeout;

  // 5. 执行渲染
  let exitStatus = shouldTimeSlice
    ? renderRootConcurrent(root, lanes)  // 并发渲染（可中断）
    : renderRootSync(root, lanes);       // 同步渲染

  // 6. 处理渲染结果
  if (exitStatus !== RootInProgress) {
    // 渲染完成，准备提交
    const finishedWork: Fiber = root.current.alternate;
    root.finishedWork = finishedWork;
    root.finishedLanes = lanes;

    // 7. 提交阶段
    finishConcurrentRender(root, exitStatus, lanes);
  }

  // 8. 检查是否还有待处理的工作
  ensureRootIsScheduled(root, now());

  // 9. 返回值决定是否继续调度
  if (root.callbackNode === originalCallbackNode) {
    // 任务被中断，返回函数继续执行
    return performConcurrentWorkOnRoot.bind(null, root);
  }
  return null;
}
```

#### 3.5.2 renderRootSync

```typescript
// 源码位置：react-reconciler/src/ReactFiberWorkLoop.new.js

function renderRootSync(root: FiberRoot, lanes: Lanes) {
  const prevExecutionContext = executionContext;
  executionContext |= RenderContext;
  const prevDispatcher = pushDispatcher();

  // 如果 root 或 lanes 改变，重新准备
  if (workInProgressRoot !== root || workInProgressRootRenderLanes !== lanes) {
    workInProgressTransitions = getTransitionsForLanes(root, lanes);
    prepareFreshStack(root, lanes);  // 创建 workInProgress 树
  }

  // 工作循环
  do {
    try {
      workLoopSync();
      break;
    } catch (thrownValue) {
      handleError(root, thrownValue);
    }
  } while (true);

  // 重置状态
  resetContextDependencies();
  executionContext = prevExecutionContext;
  popDispatcher(prevDispatcher);

  workInProgressRoot = null;
  workInProgressRootRenderLanes = NoLanes;

  return workInProgressRootExitStatus;
}

function workLoopSync() {
  // 同步渲染，不检查是否需要让出控制权
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}
```

#### 3.5.3 renderRootConcurrent（并发渲染）

```typescript
// 源码位置：react-reconciler/src/ReactFiberWorkLoop.new.js

function renderRootConcurrent(root: FiberRoot, lanes: Lanes) {
  const prevExecutionContext = executionContext;
  executionContext |= RenderContext;
  const prevDispatcher = pushDispatcher();

  if (workInProgressRoot !== root || workInProgressRootRenderLanes !== lanes) {
    workInProgressTransitions = getTransitionsForLanes(root, lanes);
    resetRenderTimer();
    prepareFreshStack(root, lanes);
  }

  // 并发工作循环
  do {
    try {
      workLoopConcurrent();
      break;
    } catch (thrownValue) {
      handleError(root, thrownValue);
    }
  } while (true);

  // ... 重置状态

  return workInProgressRootExitStatus;
}

function workLoopConcurrent() {
  // 并发渲染，每次循环都检查是否需要让出控制权
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

#### 3.5.4 performUnitOfWork

```typescript
// 源码位置：react-reconciler/src/ReactFiberWorkLoop.new.js

function performUnitOfWork(unitOfWork: Fiber): void {
  const current = unitOfWork.alternate;

  // 1. beginWork：处理当前节点，返回子节点
  let next = beginWork(current, unitOfWork, subtreeRenderLanes);

  // 保存已处理的 props
  unitOfWork.memoizedProps = unitOfWork.pendingProps;

  if (next === null) {
    // 2. 没有子节点，完成当前节点
    completeUnitOfWork(unitOfWork);
  } else {
    // 3. 有子节点，继续处理子节点
    workInProgress = next;
  }
}
```

#### 3.5.5 beginWork

```typescript
// 源码位置：react-reconciler/src/ReactFiberBeginWork.new.js

function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  // 1. 更新时的优化路径
  if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;

    if (oldProps !== newProps || hasLegacyContextChanged()) {
      didReceiveUpdate = true;
    } else {
      // 检查是否有待处理的更新
      const hasScheduledUpdateOrContext = checkScheduledUpdateOrContext(
        current,
        renderLanes,
      );
      if (!hasScheduledUpdateOrContext) {
        didReceiveUpdate = false;
        // 可以 bailout（复用）
        return attemptEarlyBailoutIfNoScheduledUpdate(
          current,
          workInProgress,
          renderLanes,
        );
      }
    }
  } else {
    didReceiveUpdate = false;
  }

  // 2. 清除 lanes
  workInProgress.lanes = NoLanes;

  // 3. 根据节点类型处理
  switch (workInProgress.tag) {
    case IndeterminateComponent:
      return mountIndeterminateComponent(
        current, workInProgress, workInProgress.type, renderLanes,
      );
    case FunctionComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      return updateFunctionComponent(
        current, workInProgress, Component, unresolvedProps, renderLanes,
      );
    }
    case ClassComponent:
      return updateClassComponent(
        current, workInProgress, workInProgress.type,
        workInProgress.pendingProps, renderLanes,
      );
    case HostRoot:
      return updateHostRoot(current, workInProgress, renderLanes);
    case HostComponent:
      return updateHostComponent(current, workInProgress, renderLanes);
    case HostText:
      return updateHostText(current, workInProgress);
    // ... 其他类型
  }
}
```

#### 3.5.6 completeUnitOfWork

```typescript
// 源码位置：react-reconciler/src/ReactFiberWorkLoop.new.js

function completeUnitOfWork(unitOfWork: Fiber): void {
  let completedWork = unitOfWork;

  do {
    const current = completedWork.alternate;
    const returnFiber = completedWork.return;

    if ((completedWork.flags & Incomplete) === NoFlags) {
      // 1. 正常完成，调用 completeWork
      let next = completeWork(current, completedWork, subtreeRenderLanes);

      if (next !== null) {
        // 产生了新的工作
        workInProgress = next;
        return;
      }
    } else {
      // 2. 出错，进行错误处理
      const next = unwindWork(current, completedWork, subtreeRenderLanes);
      // ...
    }

    // 3. 检查兄弟节点
    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      workInProgress = siblingFiber;
      return;
    }

    // 4. 返回父节点
    completedWork = returnFiber;
    workInProgress = completedWork;
  } while (completedWork !== null);

  // 5. 到达根节点，标记完成
  if (workInProgressRootExitStatus === RootInProgress) {
    workInProgressRootExitStatus = RootCompleted;
  }
}
```

### 3.6 提交阶段详解

#### 3.6.1 commitRoot

```typescript
// 源码位置：react-reconciler/src/ReactFiberWorkLoop.new.js

function commitRoot(root: FiberRoot, recoverableErrors: null | Array<mixed>) {
  const previousUpdateLanePriority = getCurrentUpdatePriority();
  const prevTransition = ReactCurrentBatchConfig.transition;

  try {
    ReactCurrentBatchConfig.transition = null;
    setCurrentUpdatePriority(DiscreteEventPriority);
    commitRootImpl(root, recoverableErrors, previousUpdateLanePriority);
  } finally {
    ReactCurrentBatchConfig.transition = prevTransition;
    setCurrentUpdatePriority(previousUpdateLanePriority);
  }
}
```

#### 3.6.2 commitRootImpl

```typescript
// 源码位置：react-reconciler/src/ReactFiberWorkLoop.new.js

function commitRootImpl(
  root: FiberRoot,
  recoverableErrors: null | Array<mixed>,
  renderPriorityLevel: EventPriority,
) {
  // 1. 执行之前的 passive effects
  do {
    flushPassiveEffects();
  } while (rootWithPendingPassiveEffects !== null);

  const finishedWork = root.finishedWork;
  const lanes = root.finishedLanes;

  // 2. 重置状态
  root.finishedWork = null;
  root.finishedLanes = NoLanes;
  root.callbackNode = null;
  root.callbackPriority = NoLane;

  // 3. 调度 passive effects（useEffect）
  if (
    (finishedWork.subtreeFlags & PassiveMask) !== NoFlags ||
    (finishedWork.flags & PassiveMask) !== NoFlags
  ) {
    if (!rootDoesHavePassiveEffects) {
      rootDoesHavePassiveEffects = true;
      scheduleCallback(NormalSchedulerPriority, () => {
        flushPassiveEffects();
        return null;
      });
    }
  }

  // 4. 检查是否有副作用
  const subtreeHasEffects = (finishedWork.subtreeFlags &
    (BeforeMutationMask | MutationMask | LayoutMask | PassiveMask)) !== NoFlags;
  const rootHasEffect = (finishedWork.flags &
    (BeforeMutationMask | MutationMask | LayoutMask | PassiveMask)) !== NoFlags;

  if (subtreeHasEffects || rootHasEffect) {
    const prevExecutionContext = executionContext;
    executionContext |= CommitContext;
    ReactCurrentOwner.current = null;

    // ====== 三个提交子阶段 ======

    // 阶段1: Before Mutation（DOM 修改前）
    // - 调用 getSnapshotBeforeUpdate
    // - 调度 useEffect
    commitBeforeMutationEffects(root, finishedWork);

    // 阶段2: Mutation（DOM 修改）
    // - 执行 DOM 操作（插入、更新、删除）
    // - 执行 useLayoutEffect 的销毁函数
    // - 更新 ref
    commitMutationEffects(root, finishedWork, lanes);

    // 切换 current 指针（双缓存切换）
    root.current = finishedWork;

    // 阶段3: Layout（DOM 修改后）
    // - 执行 useLayoutEffect 的创建函数
    // - 执行 componentDidMount/componentDidUpdate
    // - 执行 ref 回调
    commitLayoutEffects(finishedWork, root, lanes);

    executionContext = prevExecutionContext;
  } else {
    // 无副作用，直接切换 current
    root.current = finishedWork;
  }

  // 5. 处理剩余的 lanes
  const remainingLanes = mergeLanes(root.pendingLanes, root.suspendedLanes);
  markRootFinished(root, remainingLanes);

  // 6. 确保后续调度
  ensureRootIsScheduled(root, now());

  // 7. 同步刷新 layout effects 中触发的更新
  flushSyncCallbacks();
}
```

---

## 第四部分：调度机制（Scheduler）

Scheduler 是 React 的任务调度器，负责管理任务的优先级和执行时机。

### 4.1 Scheduler 核心概念

#### 4.1.1 优先级

```typescript
// 源码位置：scheduler/src/SchedulerPriorities.js

export const ImmediatePriority = 1;      // 立即执行（最高优先级）
export const UserBlockingPriority = 2;   // 用户交互（如点击）
export const NormalPriority = 3;         // 普通优先级（默认）
export const LowPriority = 4;            // 低优先级
export const IdlePriority = 5;           // 空闲优先级（最低）
```

#### 4.1.2 超时时间

```typescript
// 源码位置：scheduler/src/forks/Scheduler.js

var IMMEDIATE_PRIORITY_TIMEOUT = -1;          // 立即过期
var USER_BLOCKING_PRIORITY_TIMEOUT = 250;     // 250ms
var NORMAL_PRIORITY_TIMEOUT = 5000;           // 5s
var LOW_PRIORITY_TIMEOUT = 10000;             // 10s
var IDLE_PRIORITY_TIMEOUT = maxSigned31BitInt; // 永不过期
```

#### 4.1.3 任务队列

Scheduler 维护两个队列：

```typescript
// 源码位置：scheduler/src/forks/Scheduler.js

var taskQueue = [];    // 已过期/需要立即执行的任务
var timerQueue = [];   // 延迟任务

// 任务结构
type Task = {
  id: number,              // 任务 ID
  callback: Function,      // 任务回调
  priorityLevel: number,   // 优先级
  startTime: number,       // 开始时间
  expirationTime: number,  // 过期时间
  sortIndex: number,       // 排序索引
};
```

### 4.2 小顶堆实现

Scheduler 使用小顶堆管理任务，确保最高优先级（最小过期时间）的任务在堆顶。

```typescript
// 源码位置：scheduler/src/SchedulerMinHeap.js

type Heap = Array<Node>;
type Node = {
  id: number,
  sortIndex: number,
};

// 插入节点
export function push(heap: Heap, node: Node): void {
  const index = heap.length;
  heap.push(node);
  siftUp(heap, node, index);  // 上浮
}

// 获取堆顶（不删除）
export function peek(heap: Heap): Node | null {
  return heap.length === 0 ? null : heap[0];
}

// 弹出堆顶
export function pop(heap: Heap): Node | null {
  if (heap.length === 0) {
    return null;
  }
  const first = heap[0];
  const last = heap.pop();
  if (last !== first) {
    heap[0] = last;
    siftDown(heap, last, 0);  // 下沉
  }
  return first;
}

// 上浮操作
function siftUp(heap, node, i) {
  let index = i;
  while (index > 0) {
    const parentIndex = (index - 1) >>> 1;  // 父节点索引
    const parent = heap[parentIndex];
    if (compare(parent, node) > 0) {
      // 父节点更大，交换
      heap[parentIndex] = node;
      heap[index] = parent;
      index = parentIndex;
    } else {
      return;
    }
  }
}

// 下沉操作
function siftDown(heap, node, i) {
  let index = i;
  const length = heap.length;
  const halfLength = length >>> 1;
  while (index < halfLength) {
    const leftIndex = (index + 1) * 2 - 1;
    const left = heap[leftIndex];
    const rightIndex = leftIndex + 1;
    const right = heap[rightIndex];

    if (compare(left, node) < 0) {
      if (rightIndex < length && compare(right, left) < 0) {
        heap[index] = right;
        heap[rightIndex] = node;
        index = rightIndex;
      } else {
        heap[index] = left;
        heap[leftIndex] = node;
        index = leftIndex;
      }
    } else if (rightIndex < length && compare(right, node) < 0) {
      heap[index] = right;
      heap[rightIndex] = node;
      index = rightIndex;
    } else {
      return;
    }
  }
}

// 比较函数
function compare(a, b) {
  const diff = a.sortIndex - b.sortIndex;
  return diff !== 0 ? diff : a.id - b.id;
}
```

### 4.3 scheduleCallback 详解

这是 React 调用 Scheduler 的主要入口。

```typescript
// 源码位置：scheduler/src/forks/Scheduler.js

function unstable_scheduleCallback(priorityLevel, callback, options) {
  var currentTime = getCurrentTime();

  // 1. 计算开始时间
  var startTime;
  if (typeof options === 'object' && options !== null) {
    var delay = options.delay;
    if (typeof delay === 'number' && delay > 0) {
      startTime = currentTime + delay;  // 延迟任务
    } else {
      startTime = currentTime;
    }
  } else {
    startTime = currentTime;
  }

  // 2. 根据优先级计算超时时间
  var timeout;
  switch (priorityLevel) {
    case ImmediatePriority:
      timeout = IMMEDIATE_PRIORITY_TIMEOUT;    // -1
      break;
    case UserBlockingPriority:
      timeout = USER_BLOCKING_PRIORITY_TIMEOUT; // 250ms
      break;
    case IdlePriority:
      timeout = IDLE_PRIORITY_TIMEOUT;          // 永不过期
      break;
    case LowPriority:
      timeout = LOW_PRIORITY_TIMEOUT;           // 10s
      break;
    case NormalPriority:
    default:
      timeout = NORMAL_PRIORITY_TIMEOUT;        // 5s
      break;
  }

  // 3. 计算过期时间
  var expirationTime = startTime + timeout;

  // 4. 创建任务
  var newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    sortIndex: -1,
  };

  // 5. 根据开始时间决定放入哪个队列
  if (startTime > currentTime) {
    // 延迟任务，放入 timerQueue
    newTask.sortIndex = startTime;
    push(timerQueue, newTask);

    // 如果 taskQueue 为空且当前任务是最早的延迟任务
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      if (isHostTimeoutScheduled) {
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      // 设置定时器
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    // 非延迟任务，放入 taskQueue
    newTask.sortIndex = expirationTime;
    push(taskQueue, newTask);

    // 开启调度
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    }
  }

  return newTask;
}
```

### 4.4 requestHostCallback 详解

选择合适的宏任务 API 来调度任务执行。

```typescript
// 源码位置：scheduler/src/forks/Scheduler.js

let schedulePerformWorkUntilDeadline;

if (typeof localSetImmediate === 'function') {
  // Node.js 和旧版 IE
  schedulePerformWorkUntilDeadline = () => {
    localSetImmediate(performWorkUntilDeadline);
  };
} else if (typeof MessageChannel !== 'undefined') {
  // 浏览器环境（优先使用，避免 setTimeout 的 4ms 延迟）
  const channel = new MessageChannel();
  const port = channel.port2;
  channel.port1.onmessage = performWorkUntilDeadline;
  schedulePerformWorkUntilDeadline = () => {
    port.postMessage(null);
  };
} else {
  // 兜底方案
  schedulePerformWorkUntilDeadline = () => {
    localSetTimeout(performWorkUntilDeadline, 0);
  };
}

function requestHostCallback(callback) {
  scheduledHostCallback = callback;
  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true;
    schedulePerformWorkUntilDeadline();
  }
}
```

### 4.5 performWorkUntilDeadline 详解

调度执行的入口函数。

```typescript
// 源码位置：scheduler/src/forks/Scheduler.js

const performWorkUntilDeadline = () => {
  if (scheduledHostCallback !== null) {
    const currentTime = getCurrentTime();
    startTime = currentTime;
    const hasTimeRemaining = true;
    let hasMoreWork = true;

    try {
      // 执行 flushWork
      hasMoreWork = scheduledHostCallback(hasTimeRemaining, currentTime);
    } finally {
      if (hasMoreWork) {
        // 还有任务，继续调度
        schedulePerformWorkUntilDeadline();
      } else {
        isMessageLoopRunning = false;
        scheduledHostCallback = null;
      }
    }
  } else {
    isMessageLoopRunning = false;
  }
  needsPaint = false;
};
```

### 4.6 workLoop 详解

Scheduler 的核心工作循环。

```typescript
// 源码位置：scheduler/src/forks/Scheduler.js

function flushWork(hasTimeRemaining, initialTime) {
  isHostCallbackScheduled = false;
  if (isHostTimeoutScheduled) {
    isHostTimeoutScheduled = false;
    cancelHostTimeout();
  }

  isPerformingWork = true;
  const previousPriorityLevel = currentPriorityLevel;

  try {
    return workLoop(hasTimeRemaining, initialTime);
  } finally {
    currentTask = null;
    currentPriorityLevel = previousPriorityLevel;
    isPerformingWork = false;
  }
}

function workLoop(hasTimeRemaining, initialTime) {
  let currentTime = initialTime;

  // 1. 将到期的 timerQueue 任务转移到 taskQueue
  advanceTimers(currentTime);

  // 2. 获取最高优先级任务
  currentTask = peek(taskQueue);

  while (currentTask !== null) {
    // 3. 检查是否需要让出控制权
    if (
      currentTask.expirationTime > currentTime &&
      (!hasTimeRemaining || shouldYieldToHost())
    ) {
      // 任务未过期，但时间片用完或需要让出
      break;
    }

    // 4. 执行任务
    const callback = currentTask.callback;
    if (typeof callback === 'function') {
      currentTask.callback = null;
      currentPriorityLevel = currentTask.priorityLevel;
      const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;

      // 执行回调（React 的 performConcurrentWorkOnRoot）
      const continuationCallback = callback(didUserCallbackTimeout);
      currentTime = getCurrentTime();

      if (typeof continuationCallback === 'function') {
        // 任务被中断，保留回调继续执行
        currentTask.callback = continuationCallback;
      } else {
        // 任务完成，从队列移除
        if (currentTask === peek(taskQueue)) {
          pop(taskQueue);
        }
      }

      advanceTimers(currentTime);
    } else {
      pop(taskQueue);
    }

    currentTask = peek(taskQueue);
  }

  // 5. 返回是否还有任务
  if (currentTask !== null) {
    return true;
  } else {
    // 检查 timerQueue
    const firstTimer = peek(timerQueue);
    if (firstTimer !== null) {
      requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
    }
    return false;
  }
}
```

### 4.7 shouldYieldToHost 详解

判断是否需要让出控制权给浏览器。

```typescript
// 源码位置：scheduler/src/forks/Scheduler.js

let frameInterval = 5; // 默认 5ms（约 200fps）
let startTime = -1;

function shouldYieldToHost() {
  const timeElapsed = getCurrentTime() - startTime;

  // 1. 时间片未用完
  if (timeElapsed < frameInterval) {
    return false;
  }

  // 2. 检查是否有用户输入等待处理
  if (enableIsInputPending) {
    if (needsPaint) {
      return true;
    }
    if (timeElapsed < continuousInputInterval) {
      // 短时间内只检查离散输入（如点击）
      if (isInputPending !== null) {
        return isInputPending();
      }
    } else if (timeElapsed < maxInterval) {
      // 较长时间检查连续输入（如鼠标移动）
      if (isInputPending !== null) {
        return isInputPending(continuousOptions);
      }
    } else {
      // 时间过长，强制让出
      return true;
    }
  }

  return true;
}
```

### 4.8 Scheduler 与 React 的连接

React 通过 Lane 优先级映射到 Scheduler 优先级：

```typescript
// 源码位置：react-reconciler/src/ReactFiberWorkLoop.new.js

function ensureRootIsScheduled(root, currentTime) {
  // ...

  // Lane 优先级转换为 Scheduler 优先级
  let schedulerPriorityLevel;
  switch (lanesToEventPriority(nextLanes)) {
    case DiscreteEventPriority:
      schedulerPriorityLevel = ImmediatePriority;    // 1
      break;
    case ContinuousEventPriority:
      schedulerPriorityLevel = UserBlockingPriority; // 2
      break;
    case DefaultEventPriority:
      schedulerPriorityLevel = NormalPriority;       // 3
      break;
    case IdleEventPriority:
      schedulerPriorityLevel = IdlePriority;         // 5
      break;
    default:
      schedulerPriorityLevel = NormalPriority;
      break;
  }

  // 调用 Scheduler
  newCallbackNode = scheduleCallback(
    schedulerPriorityLevel,
    performConcurrentWorkOnRoot.bind(null, root),
  );
}
```

---

## 第五部分：完整调用链总结

### 5.1 初始化流程调用链

```
ReactDOM.createRoot(container)
    │
    ├── createRoot(container, options)
    │       │
    │       ├── isValidContainer(container)              // 验证容器
    │       │
    │       ├── createContainer(container, ConcurrentRoot, ...)
    │       │       │
    │       │       └── createFiberRoot(container, tag, ...)
    │       │               │
    │       │               ├── new FiberRootNode(...)   // 创建 FiberRoot
    │       │               │
    │       │               ├── createHostRootFiber(tag, ...) // 创建根 Fiber
    │       │               │       │
    │       │               │       └── createFiber(HostRoot, null, null, mode)
    │       │               │
    │       │               ├── root.current = uninitializedFiber
    │       │               ├── uninitializedFiber.stateNode = root
    │       │               │
    │       │               └── initializeUpdateQueue(uninitializedFiber)
    │       │
    │       ├── markContainerAsRoot(root.current, container)
    │       │
    │       ├── listenToAllSupportedEvents(rootContainerElement)
    │       │
    │       └── return new ReactDOMRoot(root)
    │
    └── root.render(<App />)
            │
            └── updateContainer(children, root, null, null)
                    │
                    ├── requestEventTime()               // 获取事件时间
                    ├── requestUpdateLane(current)       // 获取更新优先级
                    │
                    ├── createUpdate(eventTime, lane)    // 创建更新
                    ├── update.payload = {element: <App />}
                    │
                    ├── enqueueUpdate(current, update, lane) // 入队更新
                    │
                    └── scheduleUpdateOnFiber(current, lane, eventTime)
                            │
                            ├── markUpdateLaneFromFiberToRoot(fiber, lane)
                            ├── markRootUpdated(root, lane, eventTime)
                            │
                            └── ensureRootIsScheduled(root, eventTime)
                                    │
                                    └── scheduleCallback(priority, performConcurrentWorkOnRoot)
```

### 5.2 更新流程调用链

```
setState(newValue)
    │
    └── dispatchSetState(fiber, queue, action)
            │
            ├── requestUpdateLane(fiber)
            │
            ├── 创建 Update: {lane, action, ...}
            │
            ├── enqueueUpdate(fiber, queue, update, lane)
            │
            ├── [eagerState 优化]
            │       │
            │       └── 如果新旧 state 相同，提前返回
            │
            └── scheduleUpdateOnFiber(fiber, lane, eventTime)
                    │
                    ├── markUpdateLaneFromFiberToRoot(fiber, lane)
                    │       │
                    │       └── 向上遍历，更新所有祖先的 childLanes
                    │
                    ├── markRootUpdated(root, lane, eventTime)
                    │
                    └── ensureRootIsScheduled(root, eventTime)
                            │
                            ├── markStarvedLanesAsExpired(root, currentTime)
                            │
                            ├── getNextLanes(root, wipLanes)
                            │
                            ├── [如果是 SyncLane]
                            │       │
                            │       ├── scheduleSyncCallback(performSyncWorkOnRoot)
                            │       └── scheduleMicrotask(flushSyncCallbacks)
                            │
                            └── [如果是其他优先级]
                                    │
                                    └── scheduleCallback(priority, performConcurrentWorkOnRoot)
                                            │
                                            └── [Scheduler 调度]
                                                    │
                                                    └── workLoop()
                                                            │
                                                            └── performConcurrentWorkOnRoot(root)
                                                                    │
                                                                    ├── renderRootSync/Concurrent(root, lanes)
                                                                    │       │
                                                                    │       └── workLoopSync/Concurrent()
                                                                    │               │
                                                                    │               └── performUnitOfWork(workInProgress)
                                                                    │                       │
                                                                    │                       ├── beginWork(current, wip, lanes)
                                                                    │                       │       │
                                                                    │                       │       ├── updateFunctionComponent()
                                                                    │                       │       │       │
                                                                    │                       │       │       └── renderWithHooks()
                                                                    │                       │       │               │
                                                                    │                       │       │               └── Component(props)
                                                                    │                       │       │                       │
                                                                    │                       │       │                       └── useState() / updateReducer()
                                                                    │                       │       │
                                                                    │                       │       └── reconcileChildren()
                                                                    │                       │
                                                                    │                       └── completeUnitOfWork(unitOfWork)
                                                                    │                               │
                                                                    │                               └── completeWork()
                                                                    │
                                                                    └── commitRoot(root)
                                                                            │
                                                                            └── commitRootImpl(root, ...)
                                                                                    │
                                                                                    ├── commitBeforeMutationEffects()
                                                                                    │
                                                                                    ├── commitMutationEffects()
                                                                                    │       │
                                                                                    │       └── [执行 DOM 操作]
                                                                                    │
                                                                                    ├── root.current = finishedWork
                                                                                    │
                                                                                    └── commitLayoutEffects()
                                                                                            │
                                                                                            └── [执行 useLayoutEffect]
```

### 5.3 调度流程调用链

```
scheduleCallback(priority, callback)
    │
    ├── getCurrentTime()
    │
    ├── 计算 startTime, timeout, expirationTime
    │
    ├── 创建 Task: {id, callback, priority, startTime, expirationTime, sortIndex}
    │
    ├── [如果是延迟任务]
    │       │
    │       ├── push(timerQueue, newTask)
    │       │
    │       └── requestHostTimeout(handleTimeout, delay)
    │               │
    │               └── setTimeout(handleTimeout, delay)
    │
    └── [如果是立即任务]
            │
            ├── push(taskQueue, newTask)
            │
            └── requestHostCallback(flushWork)
                    │
                    └── schedulePerformWorkUntilDeadline()
                            │
                            ├── [Node.js] setImmediate(performWorkUntilDeadline)
                            │
                            ├── [Browser] MessageChannel.postMessage()
                            │
                            └── [Fallback] setTimeout(performWorkUntilDeadline, 0)

performWorkUntilDeadline()
    │
    └── flushWork(hasTimeRemaining, currentTime)
            │
            └── workLoop(hasTimeRemaining, initialTime)
                    │
                    ├── advanceTimers(currentTime)      // 转移到期的 timer
                    │
                    ├── currentTask = peek(taskQueue)
                    │
                    └── while (currentTask !== null) {
                            │
                            ├── [检查是否需要让出]
                            │       │
                            │       └── shouldYieldToHost()
                            │               │
                            │               ├── 时间片检查 (5ms)
                            │               │
                            │               └── isInputPending() // 用户输入检查
                            │
                            ├── callback(didUserCallbackTimeout)
                            │       │
                            │       └── performConcurrentWorkOnRoot(root)
                            │               │
                            │               ├── [完成] return null
                            │               │
                            │               └── [中断] return performConcurrentWorkOnRoot.bind(null, root)
                            │
                            ├── [如果返回函数]
                            │       │
                            │       └── currentTask.callback = continuationCallback  // 保留继续执行
                            │
                            └── [如果返回 null]
                                    │
                                    └── pop(taskQueue)  // 移除已完成任务
                        }
```

---

## 附录：关键函数速查表

| 函数名 | 文件位置 | 作用 |
|--------|----------|------|
| `createRoot` | react-dom/src/client/ReactDOMRoot.js | 创建 React 根节点 |
| `createFiberRoot` | react-reconciler/src/ReactFiberRoot.new.js | 创建 FiberRoot |
| `createHostRootFiber` | react-reconciler/src/ReactFiber.new.js | 创建根 Fiber |
| `updateContainer` | react-reconciler/src/ReactFiberReconciler.new.js | 触发更新 |
| `scheduleUpdateOnFiber` | react-reconciler/src/ReactFiberWorkLoop.new.js | 调度更新入口 |
| `ensureRootIsScheduled` | react-reconciler/src/ReactFiberWorkLoop.new.js | 确保根被调度 |
| `performConcurrentWorkOnRoot` | react-reconciler/src/ReactFiberWorkLoop.new.js | 并发渲染入口 |
| `performSyncWorkOnRoot` | react-reconciler/src/ReactFiberWorkLoop.new.js | 同步渲染入口 |
| `renderRootSync` | react-reconciler/src/ReactFiberWorkLoop.new.js | 同步渲染 |
| `renderRootConcurrent` | react-reconciler/src/ReactFiberWorkLoop.new.js | 并发渲染 |
| `workLoopSync` | react-reconciler/src/ReactFiberWorkLoop.new.js | 同步工作循环 |
| `workLoopConcurrent` | react-reconciler/src/ReactFiberWorkLoop.new.js | 并发工作循环 |
| `performUnitOfWork` | react-reconciler/src/ReactFiberWorkLoop.new.js | 处理单个 Fiber |
| `beginWork` | react-reconciler/src/ReactFiberBeginWork.new.js | 递阶段（向下） |
| `completeUnitOfWork` | react-reconciler/src/ReactFiberWorkLoop.new.js | 归阶段（向上） |
| `completeWork` | react-reconciler/src/ReactFiberCompleteWork.new.js | 完成单个 Fiber |
| `commitRoot` | react-reconciler/src/ReactFiberWorkLoop.new.js | 提交阶段入口 |
| `commitRootImpl` | react-reconciler/src/ReactFiberWorkLoop.new.js | 提交阶段实现 |
| `dispatchSetState` | react-reconciler/src/ReactFiberHooks.new.js | setState 实现 |
| `mountState` | react-reconciler/src/ReactFiberHooks.new.js | useState mount |
| `updateReducer` | react-reconciler/src/ReactFiberHooks.new.js | useState update |
| `scheduleCallback` | scheduler/src/forks/Scheduler.js | Scheduler 调度入口 |
| `workLoop` | scheduler/src/forks/Scheduler.js | Scheduler 工作循环 |
| `shouldYieldToHost` | scheduler/src/forks/Scheduler.js | 判断是否让出控制权 |

---

## 结语

本文档详细分析了 React 18 的三大核心流程：

1. **初始化流程**：从 `createRoot` 到 `render`，创建 FiberRoot 和根 Fiber，绑定事件委托，触发首次渲染调度。

2. **更新流程**：从 `setState` 到界面更新，创建 Update 对象，通过 Lane 优先级系统调度更新，经过 render（递归构建 Fiber 树）和 commit（提交 DOM 变更）两个阶段。

3. **调度机制**：Scheduler 使用小顶堆管理任务队列，通过时间切片实现可中断渲染，配合 `shouldYieldToHost` 实现优先级调度和用户交互响应。

理解这三条主线，基本就掌握了 React 的核心工作原理。
