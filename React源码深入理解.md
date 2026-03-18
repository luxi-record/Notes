# React 源码深度剖析：从初始化到更新的完整流程

本文档以 React 18 源码为基础，通过三条主线深入剖析 React 内部机制：
1. **初始化流程**：`ReactDOM.createRoot(root).render(<App/>)` 如何启动整个应用
2. **更新流程**：`setState` 触发更新时 React 内部如何处理
3. **调度机制**：Scheduler 如何实现任务调度和时间切片

---

## 目录

- [第一部分：核心数据结构](#第一部分核心数据结构)
  - [1.1 FiberRoot](#11-fiberroot应用根节点)
  - [1.2 Fiber](#12-fiberfiber-节点)
  - [1.3 Hook](#13-hookhook-数据结构)
  - [1.4 Lane（基础）](#14-lane优先级模型)
  - [1.5 Lane 优先级系统详解](#15-lane-优先级系统详解)
- [第二部分：初始化流程（createRoot + render）](#第二部分初始化流程createroot--render)
  - [2.1 整体流程图](#21-整体流程图)
  - [2.2 createRoot 详解](#22-createroot-详解)
  - [2.3 事件系统详解](#23-事件系统详解listentoallsupportedevents)
    - [2.3.9 JSX 事件属性的存储与获取机制](#239-jsx-事件属性的存储与获取机制)
    - [2.3.10 useTransition 与事件优先级降级机制](#2310-usetransition-与事件优先级降级机制)
  - [2.4 createFiberRoot 详解](#24-createfiberroot-详解)
  - [2.5 createHostRootFiber 详解](#25-createhostrootfiber-详解)
  - [2.6 render 方法详解](#26-render-方法详解)
  - [2.7 Diff 算法详解](#27-diff-算法详解reconcilechildren)
    - [2.7.1 Diff 算法的基本策略](#271-diff-算法的基本策略)
    - [2.7.2 reconcileChildren 入口](#272-reconcilechildren-入口)
    - [2.7.3 reconcileChildFibers 核心逻辑](#273-reconcilechildfibers-核心逻辑)
    - [2.7.4 单节点 Diff](#274-单节点-diffreconcilesinglement)
    - [2.7.5 多节点 Diff](#275-多节点-diffreconcilechildrenarray)
    - [2.7.6 节点移动判断](#276-节点移动判断placechild)
    - [2.7.7 多节点 Diff 完整示例](#277-多节点-diff-完整示例)
    - [2.7.8 不同类型组件的 Diff 处理](#278-不同类型组件的-diff-处理)
    - [2.7.9 Diff 算法流程图总结](#279-diff-算法流程图总结)
    - [2.7.10 Diff 算法的注意事项与最佳实践](#2710-diff-算法的注意事项与最佳实践)
- [第三部分：更新流程（setState）](#第三部分更新流程setstate)
- [第四部分：调度机制（Scheduler）](#第四部分调度机制scheduler)
  - [4.0 Scheduler 与 React 的关系](#40-scheduler-与-react-的关系)
  - [4.1 Scheduler 核心概念](#41-scheduler-核心概念)
  - [4.2 小顶堆实现](#42-小顶堆实现)
  - [4.3 scheduleCallback 详解](#43-schedulecallback-详解)
  - [4.4 requestHostCallback 详解](#44-requesthostcallback-详解)
  - [4.5 performWorkUntilDeadline 详解](#45-performworkuntildeadline-详解)
  - [4.6 workLoop 详解](#46-workloop-详解)
  - [4.7 shouldYieldToHost 详解](#47-shouldyieldtohost-详解)
  - [4.8 Scheduler 与 React 的连接](#48-scheduler-与-react-的连接)
  - [4.9 React 如何调度多个 setState](#49-react-如何调度多个-setstate)
    - [4.9.1 同步场景下的批量更新](#491-同步场景下的批量更新)
    - [4.9.2 ensureRootIsScheduled 的去重机制](#492-ensurerootisscheduled-的去重机制)
    - [4.9.3 不同优先级的 setState](#493-不同优先级的-setstate)
    - [4.9.4 高优先级打断低优先级](#494-高优先级打断低优先级)
    - [4.9.5 为什么要从头开始而不是从中断点恢复](#495-为什么要从头开始而不是从中断点恢复)
  - [4.10 时间切片的实现细节](#410-时间切片的实现细节)
  - [4.11 任务饿死的防止机制](#411-任务饿死的防止机制)
  - [4.12 调度流程完整图解](#412-调度流程完整图解)
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

### 1.5 Lane 优先级系统详解

Lane 是 React 18 中最核心的优先级系统，使用 31 位二进制数（受 JavaScript 31 位有符号整数限制）来表示优先级，**数值越小优先级越高**（位越靠右优先级越高）。

#### 1.5.1 完整的 Lane 常量定义

```typescript
// 源码位置：react-reconciler/src/ReactFiberLane.new.js

export type Lanes = number;  // 多个车道的集合（位掩码）
export type Lane = number;   // 单个车道
export type LaneMap<T> = Array<T>;  // Lane 到数据的映射数组

export const TotalLanes = 31;  // 总共 31 条车道

export const NoLanes: Lanes = /*                        */ 0b0000000000000000000000000000000;
export const NoLane: Lane = /*                          */ 0b0000000000000000000000000000000;

// ============ 同步优先级（最高）============
export const SyncLane: Lane = /*                        */ 0b0000000000000000000000000000001;

// ============ 连续输入优先级 ============
export const InputContinuousHydrationLane: Lane = /*    */ 0b0000000000000000000000000000010;
export const InputContinuousLane: Lane = /*             */ 0b0000000000000000000000000000100;

// ============ 默认优先级 ============
export const DefaultHydrationLane: Lane = /*            */ 0b0000000000000000000000000001000;
export const DefaultLane: Lane = /*                     */ 0b0000000000000000000000000010000;

// ============ Transition 优先级（共 16 条 lane）============
const TransitionHydrationLane: Lane = /*                */ 0b0000000000000000000000000100000;
const TransitionLanes: Lanes = /*                       */ 0b0000000001111111111111111000000;
const TransitionLane1: Lane = /*                        */ 0b0000000000000000000000001000000;
const TransitionLane2: Lane = /*                        */ 0b0000000000000000000000010000000;
// ... TransitionLane3 ~ TransitionLane16

// ============ Retry 优先级（共 5 条 lane）============
const RetryLanes: Lanes = /*                            */ 0b0000111110000000000000000000000;
const RetryLane1: Lane = /*                             */ 0b0000000010000000000000000000000;
// ... RetryLane2 ~ RetryLane5

// ============ 选择性 Hydration ============
export const SelectiveHydrationLane: Lane = /*          */ 0b0001000000000000000000000000000;

// ============ 空闲优先级（最低）============
export const IdleHydrationLane: Lane = /*               */ 0b0010000000000000000000000000000;
export const IdleLane: Lane = /*                        */ 0b0100000000000000000000000000000;

// ============ Offscreen（屏幕外渲染）============
export const OffscreenLane: Lane = /*                   */ 0b1000000000000000000000000000000;
```

#### 1.5.2 Lane 位运算操作

Lane 的设计精髓在于使用位运算实现高效的集合操作：

```typescript
// 源码位置：react-reconciler/src/ReactFiberLane.new.js

// 合并 lanes（位或操作）
export function mergeLanes(a: Lanes | Lane, b: Lanes | Lane): Lanes {
  return a | b;
}

// 从 set 中移除 subset（位与非操作）
export function removeLanes(set: Lanes, subset: Lanes | Lane): Lanes {
  return set & ~subset;
}

// 取交集（位与操作）
export function intersectLanes(a: Lanes | Lane, b: Lanes | Lane): Lanes {
  return a & b;
}

// 检查是否包含某些 lanes
export function includesSomeLane(a: Lanes | Lane, b: Lanes | Lane): boolean {
  return (a & b) !== NoLanes;
}

// 检查是否是 subset 的子集
export function isSubsetOfLanes(set: Lanes, subset: Lanes | Lane): boolean {
  return (set & subset) === subset;
}

// 获取最高优先级的 Lane（利用补码特性，获取最右边的 1）
export function getHighestPriorityLane(lanes: Lanes): Lane {
  return lanes & -lanes;
}
```

**`lanes & -lanes` 的原理：**
```
假设 lanes = 0b101000
-lanes = ~lanes + 1 = 0b010111 + 1 = 0b011000

lanes & -lanes = 0b101000 & 0b011000 = 0b001000
```
这个技巧利用了二进制补码的特性，能够快速获取最右边的那个 1（即最高优先级的 lane）。

#### 1.5.3 requestUpdateLane - 获取更新的 Lane

当触发更新时，React 会根据触发源决定使用哪个 Lane：

```typescript
// 源码位置：react-reconciler/src/ReactFiberWorkLoop.new.js:437
export function requestUpdateLane(fiber: Fiber): Lane {
  const mode = fiber.mode;

  // 1. Legacy 模式：总是返回同步 Lane
  if ((mode & ConcurrentMode) === NoMode) {
    return SyncLane;
  }

  // 2. 渲染阶段更新：复用当前渲染的 lane（实现批量更新）
  if ((executionContext & RenderContext) !== NoContext &&
      workInProgressRootRenderLanes !== NoLanes) {
    return pickArbitraryLane(workInProgressRootRenderLanes);
  }

  // 3. Transition 更新：分配 transition lane
  const isTransition = requestCurrentTransition() !== NoTransition;
  if (isTransition) {
    if (currentEventTransitionLane === NoLane) {
      // 分配一个新的 transition lane
      currentEventTransitionLane = claimNextTransitionLane();
    }
    return currentEventTransitionLane;
  }

  // 4. React 内部事件触发的优先级
  const updateLane = getCurrentUpdatePriority();
  if (updateLane !== NoLane) {
    return updateLane;
  }

  // 5. React 外部事件（原生事件）触发的优先级
  const eventLane = getCurrentEventPriority();
  return eventLane;
}
```

#### 1.5.4 getNextLanes - 获取下一个要处理的 Lanes

这是调度系统的核心函数，决定接下来要处理哪些优先级的任务：

```typescript
// 源码位置：react-reconciler/src/ReactFiberLane.new.js:187

export function getNextLanes(root: FiberRoot, wipLanes: Lanes): Lanes {
  const pendingLanes = root.pendingLanes;
  if (pendingLanes === NoLanes) {
    return NoLanes;
  }

  let nextLanes = NoLanes;
  const suspendedLanes = root.suspendedLanes;
  const pingedLanes = root.pingedLanes;

  // 1. 优先处理非 Idle 任务
  const nonIdlePendingLanes = pendingLanes & NonIdleLanes;
  if (nonIdlePendingLanes !== NoLanes) {
    // 优先处理未被挂起的任务
    const nonIdleUnblockedLanes = nonIdlePendingLanes & ~suspendedLanes;
    if (nonIdleUnblockedLanes !== NoLanes) {
      nextLanes = getHighestPriorityLanes(nonIdleUnblockedLanes);
    } else {
      // 否则处理已被 ping 的任务（Suspense 恢复）
      const nonIdlePingedLanes = nonIdlePendingLanes & pingedLanes;
      if (nonIdlePingedLanes !== NoLanes) {
        nextLanes = getHighestPriorityLanes(nonIdlePingedLanes);
      }
    }
  } else {
    // 2. 只剩 Idle 任务
    const unblockedLanes = pendingLanes & ~suspendedLanes;
    if (unblockedLanes !== NoLanes) {
      nextLanes = getHighestPriorityLanes(unblockedLanes);
    } else if (pingedLanes !== NoLanes) {
      nextLanes = getHighestPriorityLanes(pingedLanes);
    }
  }

  // 3. 判断是否需要中断当前渲染（高优先级插队）
  if (wipLanes !== NoLanes && wipLanes !== nextLanes &&
      (wipLanes & suspendedLanes) === NoLanes) {
    const nextLane = getHighestPriorityLane(nextLanes);
    const wipLane = getHighestPriorityLane(wipLanes);
    // 如果新任务优先级不高于当前任务，继续当前渲染
    if (nextLane >= wipLane ||
        (nextLane === DefaultLane && (wipLane & TransitionLanes) !== NoLanes)) {
      return wipLanes;
    }
  }

  // 4. 处理 entangled lanes（纠缠的 lane，必须一起处理）
  const entangledLanes = root.entangledLanes;
  if (entangledLanes !== NoLanes) {
    const entanglements = root.entanglements;
    let lanes = nextLanes & entangledLanes;
    while (lanes > 0) {
      const index = pickArbitraryLaneIndex(lanes);
      const lane = 1 << index;
      nextLanes |= entanglements[index];
      lanes &= ~lane;
    }
  }

  return nextLanes;
}
```

#### 1.5.5 Lane 与 Scheduler 优先级的映射

React 内部使用 Lane，但 Scheduler 使用数字优先级。它们之间需要转换：

```typescript
// 源码位置：react-reconciler/src/ReactEventPriorities.new.js

// EventPriority 实际上就是对应的 Lane 值
export const DiscreteEventPriority: EventPriority = SyncLane;           // 离散事件
export const ContinuousEventPriority: EventPriority = InputContinuousLane;  // 连续事件
export const DefaultEventPriority: EventPriority = DefaultLane;         // 默认事件
export const IdleEventPriority: EventPriority = IdleLane;               // 空闲事件

// Lane 转 EventPriority
export function lanesToEventPriority(lanes: Lanes): EventPriority {
  const lane = getHighestPriorityLane(lanes);
  if (!isHigherEventPriority(DiscreteEventPriority, lane)) {
    return DiscreteEventPriority;  // SyncLane
  }
  if (!isHigherEventPriority(ContinuousEventPriority, lane)) {
    return ContinuousEventPriority;  // InputContinuousLane
  }
  if (includesNonIdleWork(lane)) {
    return DefaultEventPriority;  // DefaultLane
  }
  return IdleEventPriority;  // IdleLane
}
```

```typescript
// 源码位置：react-reconciler/src/ReactFiberWorkLoop.new.js
// 在 ensureRootIsScheduled 中的转换

let schedulerPriorityLevel;
switch (lanesToEventPriority(nextLanes)) {
  case DiscreteEventPriority:
    schedulerPriorityLevel = ImmediateSchedulerPriority;  // 1
    break;
  case ContinuousEventPriority:
    schedulerPriorityLevel = UserBlockingSchedulerPriority;  // 2
    break;
  case DefaultEventPriority:
    schedulerPriorityLevel = NormalSchedulerPriority;  // 3
    break;
  case IdleEventPriority:
    schedulerPriorityLevel = IdleSchedulerPriority;  // 5
    break;
  default:
    schedulerPriorityLevel = NormalSchedulerPriority;
}

// Scheduler 优先级定义
// 源码位置：scheduler/src/SchedulerPriorities.js
export const NoPriority = 0;
export const ImmediatePriority = 1;     // 立即执行，timeout = -1
export const UserBlockingPriority = 2;  // 用户阻塞，timeout = 250ms
export const NormalPriority = 3;        // 普通，timeout = 5000ms
export const LowPriority = 4;           // 低优先级，timeout = 10000ms
export const IdlePriority = 5;          // 空闲，timeout = maxSigned31BitInt（永不过期）
```

#### 1.5.6 完整的优先级映射表

```
┌─────────────────────────┬─────────────────────────┬─────────────────────┬──────────────────────────┬────────────┐
│ DOM 事件类型            │ EventPriority           │ Lane                │ Scheduler Priority       │ Timeout    │
├─────────────────────────┼─────────────────────────┼─────────────────────┼──────────────────────────┼────────────┤
│ click, keydown, input   │ DiscreteEventPriority   │ SyncLane            │ ImmediatePriority (1)    │ -1 (立即)  │
├─────────────────────────┼─────────────────────────┼─────────────────────┼──────────────────────────┼────────────┤
│ mousemove, scroll, drag │ ContinuousEventPriority │ InputContinuousLane │ UserBlockingPriority (2) │ 250ms      │
├─────────────────────────┼─────────────────────────┼─────────────────────┼──────────────────────────┼────────────┤
│ 默认 / useTransition    │ DefaultEventPriority    │ DefaultLane /       │ NormalPriority (3)       │ 5000ms     │
│                         │                         │ TransitionLanes     │                          │            │
├─────────────────────────┼─────────────────────────┼─────────────────────┼──────────────────────────┼────────────┤
│ requestIdleCallback     │ IdleEventPriority       │ IdleLane            │ IdlePriority (5)         │ 永不过期   │
└─────────────────────────┴─────────────────────────┴─────────────────────┴──────────────────────────┴────────────┘
```

#### 1.5.7 FiberRoot 中 Lane 相关字段

```typescript
type FiberRoot = {
  // ... 其他字段

  // Lane 优先级管理
  pendingLanes: Lanes,      // 待处理的 lanes（所有未完成的更新）
  suspendedLanes: Lanes,    // 被挂起的 lanes（Suspense 造成）
  pingedLanes: Lanes,       // 被 ping 的 lanes（Suspense 数据加载完成）
  expiredLanes: Lanes,      // 已过期的 lanes（需要同步执行，防止饥饿）
  mutableReadLanes: Lanes,  // 可变读取的 lanes
  finishedLanes: Lanes,     // 已完成的 lanes（render 阶段完成，等待 commit）

  // Lane 纠缠（entanglement）
  entangledLanes: Lanes,          // 有纠缠关系的 lanes
  entanglements: LaneMap<Lanes>,  // 每个 lane 纠缠的其他 lanes（必须一起处理）

  // 时间记录
  eventTimes: LaneMap<number>,       // 每个 lane 的事件触发时间
  expirationTimes: LaneMap<number>,  // 每个 lane 的过期时间
};
```

#### 1.5.8 防饥饿机制

低优先级任务可能会被高优先级任务不断打断，导致"饥饿"。React 通过过期时间机制防止这种情况：

```typescript
// 源码位置：react-reconciler/src/ReactFiberLane.new.js

export function markStarvedLanesAsExpired(
  root: FiberRoot,
  currentTime: number,
): void {
  const pendingLanes = root.pendingLanes;
  const suspendedLanes = root.suspendedLanes;
  const pingedLanes = root.pingedLanes;
  const expirationTimes = root.expirationTimes;

  let lanes = pendingLanes;
  while (lanes > 0) {
    const index = pickArbitraryLaneIndex(lanes);
    const lane = 1 << index;
    const expirationTime = expirationTimes[index];

    if (expirationTime === NoTimestamp) {
      // 首次看到这个 lane，计算过期时间
      if ((lane & suspendedLanes) === NoLanes ||
          (lane & pingedLanes) !== NoLanes) {
        expirationTimes[index] = computeExpirationTime(lane, currentTime);
      }
    } else if (expirationTime <= currentTime) {
      // 已过期，标记为 expired（必须同步执行）
      root.expiredLanes |= lane;
    }

    lanes &= ~lane;
  }
}
```

当 lane 被标记为 `expiredLanes` 后，即使有更高优先级的任务，也必须先处理过期的任务，从而防止饥饿。

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

### 2.3 事件系统详解（listenToAllSupportedEvents）

React 的事件系统是一个完整的事件委托机制，在 `createRoot` 时通过 `listenToAllSupportedEvents` 将所有支持的事件绑定到根容器上。

#### 2.3.1 事件委托机制

**核心思想：** React 不会在每个 DOM 元素上绑定事件，而是将所有事件统一绑定到根容器（container）上，通过事件冒泡/捕获机制处理所有子元素的事件。

```typescript
// 源码位置：react-dom/src/events/DOMPluginEventSystem.js

const listeningMarker = '_reactListening' + Math.random().toString(36).slice(2);

export function listenToAllSupportedEvents(rootContainerElement: EventTarget) {
  // 1. 避免重复绑定（通过标记判断）
  if (!(rootContainerElement: any)[listeningMarker]) {
    (rootContainerElement: any)[listeningMarker] = true;

    // 2. 遍历所有原生事件，绑定到 container 上
    allNativeEvents.forEach(domEventName => {
      // selectionchange 特殊处理，需要绑定到 document
      if (domEventName !== 'selectionchange') {
        // 非委托事件（scroll、load 等不冒泡的事件）
        if (!nonDelegatedEvents.has(domEventName)) {
          // 绑定冒泡阶段
          listenToNativeEvent(domEventName, false, rootContainerElement);
        }
        // 绑定捕获阶段
        listenToNativeEvent(domEventName, true, rootContainerElement);
      }
    });

    // 3. selectionchange 绑定到 document
    const ownerDocument = rootContainerElement.ownerDocument;
    if (ownerDocument !== null) {
      if (!(ownerDocument: any)[listeningMarker]) {
        (ownerDocument: any)[listeningMarker] = true;
        listenToNativeEvent('selectionchange', false, ownerDocument);
      }
    }
  }
}

// 不委托的事件（这些事件不冒泡）
export const nonDelegatedEvents: Set<DOMEventName> = new Set([
  'cancel',
  'close',
  'invalid',
  'load',
  'scroll',
  'toggle',
  // 媒体事件
  'abort', 'canplay', 'canplaythrough', 'durationchange', 'emptied',
  'encrypted', 'ended', 'error', 'loadeddata', 'loadedmetadata',
  'loadstart', 'pause', 'play', 'playing', 'progress', 'ratechange',
  'resize', 'seeked', 'seeking', 'stalled', 'suspend', 'timeupdate',
  'volumechange', 'waiting',
]);
```

#### 2.3.2 事件优先级绑定

React 根据事件类型分配不同的优先级，不同优先级使用不同的 listener 包装函数：

```typescript
// 源码位置：react-dom/src/events/ReactDOMEventListener.js

export function createEventListenerWrapperWithPriority(
  targetContainer: EventTarget,
  domEventName: DOMEventName,
  eventSystemFlags: EventSystemFlags,
): Function {
  // 根据事件名获取优先级
  const eventPriority = getEventPriority(domEventName);

  let listenerWrapper;
  switch (eventPriority) {
    case DiscreteEventPriority:      // 最高优先级（SyncLane）
      listenerWrapper = dispatchDiscreteEvent;
      break;
    case ContinuousEventPriority:    // 连续事件优先级
      listenerWrapper = dispatchContinuousEvent;
      break;
    case DefaultEventPriority:       // 默认优先级
    default:
      listenerWrapper = dispatchEvent;
      break;
  }

  return listenerWrapper.bind(null, domEventName, eventSystemFlags, targetContainer);
}

// 获取事件优先级
export function getEventPriority(domEventName: DOMEventName): * {
  switch (domEventName) {
    // 离散事件 - 最高优先级（DiscreteEventPriority = SyncLane）
    case 'click':
    case 'keydown':
    case 'keyup':
    case 'mousedown':
    case 'mouseup':
    case 'input':
    case 'change':
    case 'focus':
    case 'blur':
    case 'submit':
    // ... 等等
      return DiscreteEventPriority;

    // 连续事件 - 中等优先级（ContinuousEventPriority = InputContinuousLane）
    case 'drag':
    case 'mousemove':
    case 'mouseout':
    case 'mouseover':
    case 'scroll':
    case 'touchmove':
    case 'wheel':
    // ... 等等
      return ContinuousEventPriority;

    // 其他事件 - 默认优先级
    default:
      return DefaultEventPriority;
  }
}
```

**事件优先级映射表：**

```
┌────────────────────────┬───────────────────────────┬──────────────────────┐
│ 事件类型               │ 事件名示例                │ 优先级               │
├────────────────────────┼───────────────────────────┼──────────────────────┤
│ 离散事件（Discrete）   │ click, keydown, input,    │ DiscreteEventPriority│
│                        │ change, focus, blur,      │ = SyncLane (1)       │
│                        │ submit, mousedown...      │                      │
├────────────────────────┼───────────────────────────┼──────────────────────┤
│ 连续事件（Continuous） │ mousemove, scroll, drag,  │ ContinuousEventPriority│
│                        │ touchmove, wheel,         │ = InputContinuousLane│
│                        │ pointerover...            │ (4)                  │
├────────────────────────┼───────────────────────────┼──────────────────────┤
│ 默认事件               │ message, 其他未分类事件   │ DefaultEventPriority │
│                        │                           │ = DefaultLane (16)   │
└────────────────────────┴───────────────────────────┴──────────────────────┘
```

#### 2.3.3 合成事件（SyntheticEvent）

React 使用合成事件包装原生事件，提供跨浏览器一致的 API：

```typescript
// 源码位置：react-dom/src/events/SyntheticEvent.js

function createSyntheticEvent(Interface: EventInterfaceType) {
  function SyntheticBaseEvent(
    reactName: string | null,      // React 事件名，如 'onClick'
    reactEventType: string,        // 事件类型，如 'click'
    targetInst: Fiber,             // 目标 Fiber 节点
    nativeEvent: Object,           // 原生事件对象
    nativeEventTarget: EventTarget // 原生事件目标
  ) {
    this._reactName = reactName;
    this._targetInst = targetInst;
    this.type = reactEventType;
    this.nativeEvent = nativeEvent;      // 保留原生事件的引用
    this.target = nativeEventTarget;
    this.currentTarget = null;

    // 从 Interface 复制属性（标准化不同浏览器的差异）
    for (const propName in Interface) {
      const normalize = Interface[propName];
      if (normalize) {
        this[propName] = normalize(nativeEvent);  // 标准化处理
      } else {
        this[propName] = nativeEvent[propName];   // 直接复制
      }
    }

    // 处理 defaultPrevented
    this.isDefaultPrevented = nativeEvent.defaultPrevented
      ? functionThatReturnsTrue
      : functionThatReturnsFalse;
    this.isPropagationStopped = functionThatReturnsFalse;

    return this;
  }

  // 添加方法
  assign(SyntheticBaseEvent.prototype, {
    preventDefault: function() {
      this.defaultPrevented = true;
      const event = this.nativeEvent;
      if (event.preventDefault) {
        event.preventDefault();
      }
      this.isDefaultPrevented = functionThatReturnsTrue;
    },

    stopPropagation: function() {
      const event = this.nativeEvent;
      if (event.stopPropagation) {
        event.stopPropagation();
      }
      this.isPropagationStopped = functionThatReturnsTrue;
    },
  });

  return SyntheticBaseEvent;
}

// 不同类型事件的合成事件构造函数
export const SyntheticEvent = createSyntheticEvent(EventInterface);
export const SyntheticMouseEvent = createSyntheticEvent(MouseEventInterface);
export const SyntheticKeyboardEvent = createSyntheticEvent(KeyboardEventInterface);
export const SyntheticFocusEvent = createSyntheticEvent(FocusEventInterface);
export const SyntheticTouchEvent = createSyntheticEvent(TouchEventInterface);
export const SyntheticWheelEvent = createSyntheticEvent(WheelEventInterface);
// ... 等等
```

**合成事件与原生事件的区别：**

| 特性 | 原生事件 | React 合成事件 |
|------|---------|---------------|
| 事件绑定 | 绑定在具体 DOM 元素上 | 委托到根容器 |
| 事件命名 | 全小写（onclick） | 驼峰命名（onClick） |
| 阻止默认 | event.preventDefault() | 相同，但也可以 return false |
| 事件对象 | 浏览器原生 Event | SyntheticEvent（包装原生事件） |
| 跨浏览器 | 存在差异 | 统一标准化 |
| this 指向 | DOM 元素 | undefined（需要手动绑定） |

#### 2.3.4 事件触发与收集流程

当用户触发事件时，React 的处理流程如下：

```typescript
// 1. 触发绑定在 container 上的 listener（dispatchDiscreteEvent/dispatchContinuousEvent/dispatchEvent）

// 离散事件触发函数
function dispatchDiscreteEvent(
  domEventName,
  eventSystemFlags,
  container,
  nativeEvent,
) {
  // 保存之前的优先级
  const previousPriority = getCurrentUpdatePriority();
  const prevTransition = ReactCurrentBatchConfig.transition;
  ReactCurrentBatchConfig.transition = null;

  try {
    // 设置当前更新优先级为离散事件优先级
    setCurrentUpdatePriority(DiscreteEventPriority);
    // 调用通用的 dispatchEvent
    dispatchEvent(domEventName, eventSystemFlags, container, nativeEvent);
  } finally {
    // 恢复之前的优先级
    setCurrentUpdatePriority(previousPriority);
    ReactCurrentBatchConfig.transition = prevTransition;
  }
}

// 2. dispatchEvent 找到触发事件的 Fiber 节点
function dispatchEvent(
  domEventName: DOMEventName,
  eventSystemFlags: EventSystemFlags,
  targetContainer: EventTarget,
  nativeEvent: AnyNativeEvent,
) {
  // 找到触发事件的 DOM 和对应的 Fiber
  // 如果是hydration，点击之前未hydration完成会阻塞，当hydration完成后再重启
  let blockedOn = findInstanceBlockingEvent(
    domEventName,
    eventSystemFlags,
    targetContainer,
    nativeEvent,
  );

  if (blockedOn === null) {
    // 通过插件系统派发事件
    dispatchEventForPluginEventSystem(
      domEventName,
      eventSystemFlags,
      nativeEvent,
      return_targetInst,  // 目标 Fiber
      targetContainer,
    );
    return;
  }
  // ... 处理 hydration 等特殊情况
}

// 3. 通过插件系统派发事件
export function dispatchEventForPluginEventSystem(
  domEventName: DOMEventName,
  eventSystemFlags: EventSystemFlags,
  nativeEvent: AnyNativeEvent,
  targetInst: null | Fiber,
  targetContainer: EventTarget,
): void {
  // ... 查找祖先节点

  // 使用批量更新包装事件处理
  batchedUpdates(() =>
    dispatchEventsForPlugins(
      domEventName,
      eventSystemFlags,
      nativeEvent,
      ancestorInst,
      targetContainer,
    ),
  );
}

// 4. 收集事件监听器
function dispatchEventsForPlugins(
  domEventName: DOMEventName,
  eventSystemFlags: EventSystemFlags,
  nativeEvent: AnyNativeEvent,
  targetInst: null | Fiber,
  targetContainer: EventTarget,
): void {
  const nativeEventTarget = getEventTarget(nativeEvent);
  const dispatchQueue: DispatchQueue = [];

  // 收集事件（通过各种插件）
  extractEvents(
    dispatchQueue,
    domEventName,
    targetInst,
    nativeEvent,
    nativeEventTarget,
    eventSystemFlags,
    targetContainer,
  );

  // 处理收集到的事件
  processDispatchQueue(dispatchQueue, eventSystemFlags);
}
```

#### 2.3.5 事件收集（从目标到根）

```typescript
// 源码位置：react-dom/src/events/DOMPluginEventSystem.js

export function accumulateSinglePhaseListeners(
  targetFiber: Fiber | null,
  reactName: string | null,       // 'onClick'
  nativeEventType: string,        // 'click'
  inCapturePhase: boolean,
  accumulateTargetOnly: boolean,
  nativeEvent: AnyNativeEvent,
): Array<DispatchListener> {
  const captureName = reactName !== null ? reactName + 'Capture' : null;
  const reactEventName = inCapturePhase ? captureName : reactName;
  let listeners: Array<DispatchListener> = [];

  let instance = targetFiber;

  // 从触发事件的 Fiber 向上遍历到根节点，收集所有同名事件
  while (instance !== null) {
    const {stateNode, tag} = instance;

    // 只处理 HostComponent（DOM 元素）
    if (tag === HostComponent && stateNode !== null) {
      // 从 Fiber 的 props 中获取事件监听器
      if (reactEventName !== null) {
        const listener = getListener(instance, reactEventName);
        if (listener != null) {
          listeners.push(
            createDispatchListener(instance, listener, stateNode),
          );
        }
      }
    }

    // 如果只需要目标节点的事件（如 scroll），不继续向上收集
    if (accumulateTargetOnly) {
      break;
    }

    // 继续向上遍历
    instance = instance.return;
  }

  return listeners;
}

// 从 Fiber 的 props 中获取事件监听器
export default function getListener(
  inst: Fiber,
  registrationName: string,  // 'onClick'
): Function | null {
  const stateNode = inst.stateNode;
  // 获取当前 Fiber 对应 DOM 节点的 props
  const props = getFiberCurrentPropsFromNode(stateNode);
  // 返回对应的事件处理函数
  const listener = props[registrationName];  // props.onClick
  return listener;
}
```

#### 2.3.6 事件执行

```typescript
// 源码位置：react-dom/src/events/DOMPluginEventSystem.js

export function processDispatchQueue(
  dispatchQueue: DispatchQueue,
  eventSystemFlags: EventSystemFlags,
): void {
  const inCapturePhase = (eventSystemFlags & IS_CAPTURE_PHASE) !== 0;

  for (let i = 0; i < dispatchQueue.length; i++) {
    const {event, listeners} = dispatchQueue[i];
    processDispatchQueueItemsInOrder(event, listeners, inCapturePhase);
  }

  rethrowCaughtError();
}

function processDispatchQueueItemsInOrder(
  event: ReactSyntheticEvent,
  dispatchListeners: Array<DispatchListener>,
  inCapturePhase: boolean,
): void {
  let previousInstance;

  if (inCapturePhase) {
    // 捕获阶段：从后往前执行（从根到目标）
    for (let i = dispatchListeners.length - 1; i >= 0; i--) {
      const {instance, currentTarget, listener} = dispatchListeners[i];
      if (instance !== previousInstance && event.isPropagationStopped()) {
        return;  // 如果调用了 stopPropagation，停止执行
      }
      executeDispatch(event, listener, currentTarget);
      previousInstance = instance;
    }
  } else {
    // 冒泡阶段：从前往后执行（从目标到根）
    for (let i = 0; i < dispatchListeners.length; i++) {
      const {instance, currentTarget, listener} = dispatchListeners[i];
      if (instance !== previousInstance && event.isPropagationStopped()) {
        return;
      }
      executeDispatch(event, listener, currentTarget);
      previousInstance = instance;
    }
  }
}

function executeDispatch(
  event: ReactSyntheticEvent,
  listener: Function,
  currentTarget: EventTarget,
): void {
  event.currentTarget = currentTarget;
  // 执行事件处理函数，this 为 undefined
  invokeGuardedCallbackAndCatchFirstError(
    event.type,
    listener,
    undefined,  // this 为 undefined，这就是为什么 React 事件中 this 为 undefined
    event,
  );
  event.currentTarget = null;
}
```

#### 2.3.7 原生事件与 React 事件同时绑定

当一个元素同时绑定原生事件和 React 事件时，**原生事件先于 React 事件执行**。

```jsx
function App() {
  const divRef = useRef(null);

  useEffect(() => {
    // 原生事件
    divRef.current.addEventListener('click', () => {
      console.log('1. 原生事件 - 冒泡阶段');
    });
    divRef.current.addEventListener('click', () => {
      console.log('0. 原生事件 - 捕获阶段');
    }, true);
  }, []);

  return (
    <div
      ref={divRef}
      onClick={() => console.log('3. React 事件 - 冒泡阶段')}
      onClickCapture={() => console.log('2. React 事件 - 捕获阶段')}
    >
      点击我
    </div>
  );
}

// 点击后输出顺序：
// 0. 原生事件 - 捕获阶段
// 1. 原生事件 - 冒泡阶段
// 2. React 事件 - 捕获阶段
// 3. React 事件 - 冒泡阶段
```

**执行顺序原理：**

```
事件流程：
                    捕获阶段                冒泡阶段
                        ↓                     ↑
    document  ─────────────────────────────────────
                        ↓                     ↑
    container ─────────────────────────────────────  ← React 事件绑定在这里
                        ↓                     ↑
    父元素    ─────────────────────────────────────
                        ↓                     ↑
    目标元素  ─────────────────────────────────────  ← 原生事件绑定在这里

1. 捕获阶段从 document 向下传播，到达目标元素
2. 原生事件在目标元素上触发
3. 冒泡阶段从目标元素向上传播
4. 到达 container 时触发 React 的事件处理
5. React 在内部模拟捕获/冒泡阶段
```

**注意事项：**
- 原生事件中调用 `e.stopPropagation()` 会阻止 React 事件触发（因为事件无法冒泡到 container）
- React 事件中调用 `e.stopPropagation()` 只能阻止 React 事件的传播，不影响原生事件
- 如果需要在 React 事件中阻止原生事件，需要使用 `e.nativeEvent.stopImmediatePropagation()`

#### 2.3.8 事件系统流程总结

```
用户点击 DOM 元素
        │
        ├─ [捕获阶段] 事件从 document 向下传播
        │         │
        │         └─ 经过 container（React 捕获监听器触发点）
        │                   │
        │                   └─ 到达目标元素
        │
        ├─ [目标阶段] 原生事件在目标元素触发
        │
        └─ [冒泡阶段] 事件从目标元素向上传播
                  │
                  └─ 到达 container
                           │
                           └─ 触发 React 的事件 listener
                                    │
                                    ├─ dispatchDiscreteEvent/dispatchContinuousEvent
                                    │        │
                                    │        └─ 设置当前更新优先级
                                    │
                                    ├─ dispatchEvent
                                    │        │
                                    │        └─ findInstanceBlockingEvent（找到目标 Fiber）
                                    │
                                    ├─ dispatchEventForPluginEventSystem
                                    │        │
                                    │        └─ batchedUpdates（批量更新包装）
                                    │
                                    ├─ extractEvents
                                    │        │
                                    │        └─ accumulateSinglePhaseListeners
                                    │                 │
                                    │                 └─ 从目标 Fiber 向上收集所有同名事件监听器
                                    │
                                    └─ processDispatchQueue
                                             │
                                             └─ 按顺序执行收集到的监听器
                                                      │
                                                      └─ executeDispatch（this = undefined）
```

#### 2.3.9 JSX 事件属性的存储与获取机制

当我们在 JSX 中写 `<div onClick={handleClick} />` 时，React 并不会把事件直接绑定到这个 DOM 元素上。事件是通过事件委托绑定在 container 上的。那 `onClick` 这个属性存储在哪里？事件触发时又是如何找到它的？

**1. JSX 到 createElement 的转换**

```jsx
// 你写的 JSX
<div onClick={handleClick} className="box">Hello</div>

// Babel 编译后
React.createElement('div', {
  onClick: handleClick,
  className: 'box'
}, 'Hello');

// 返回的 React Element 结构
{
  $$typeof: Symbol(react.element),
  type: 'div',
  props: {
    onClick: handleClick,    // 事件处理函数存在 props 中
    className: 'box',
    children: 'Hello'
  },
  key: null,
  ref: null,
}
```

**2. Fiber 节点创建时 props 的处理**

在 render 阶段，React 会为每个 React Element 创建对应的 Fiber 节点：

```typescript
// 源码位置：react-reconciler/src/ReactFiber.new.js

function createFiberFromElement(element, mode, lanes) {
  const type = element.type;
  const key = element.key;
  const pendingProps = element.props;  // props（包含 onClick）存储在 Fiber.pendingProps

  let fiber = createFiberFromTypeAndProps(
    type,
    key,
    pendingProps,  // 传递给 Fiber
    owner,
    mode,
    lanes,
  );
  return fiber;
}
```

**3. DOM 节点创建时 props 的存储**

在 commit 阶段，当 DOM 节点被创建时，React 会将 props 存储到 DOM 节点上：

```typescript
// 源码位置：react-dom/src/client/ReactDOMComponentTree.js

// 生成随机 key 避免冲突
const randomKey = Math.random().toString(36).slice(2);
const internalInstanceKey = '__reactFiber$' + randomKey;
const internalPropsKey = '__reactProps$' + randomKey;  // 用于存储 props

// 将 Fiber 节点存储到 DOM 上
export function precacheFiberNode(hostInst: Fiber, node: Instance): void {
  (node: any)[internalInstanceKey] = hostInst;  // DOM.__reactFiber$ = Fiber
}

// 将 props 存储到 DOM 上
export function updateFiberProps(node: Instance, props: Props): void {
  (node: any)[internalPropsKey] = props;  // DOM.__reactProps$ = props
}

// 从 DOM 上获取 props
export function getFiberCurrentPropsFromNode(node: Instance): Props {
  return (node: any)[internalPropsKey] || null;  // 返回 DOM.__reactProps$
}
```

**4. updateFiberProps 的调用时机**

在 commit 阶段创建或更新 DOM 时调用：

```typescript
// 源码位置：react-dom/src/client/ReactDOMHostConfig.js

export function createInstance(
  type: string,
  props: Props,
  rootContainerInstance: Container,
  hostContext: HostContext,
  internalInstanceHandle: Object,  // 对应的 Fiber
): Instance {
  // 1. 创建 DOM 元素
  const domElement: Instance = createElement(type, props, rootContainerInstance);

  // 2. 将 Fiber 存储到 DOM 上
  precacheFiberNode(internalInstanceHandle, domElement);

  // 3. 将 props 存储到 DOM 上（包含 onClick 等事件处理函数）
  updateFiberProps(domElement, props);

  return domElement;
}
```

**5. 事件触发时获取处理函数的流程**

```typescript
// 源码位置：react-dom/src/events/getListener.js

export default function getListener(
  inst: Fiber,           // 目标 Fiber 节点
  registrationName: string,  // 如 'onClick'
): Function | null {
  const stateNode = inst.stateNode;  // 获取 DOM 节点
  if (stateNode === null) {
    return null;
  }

  // 从 DOM 节点上获取 props
  const props = getFiberCurrentPropsFromNode(stateNode);
  if (props === null) {
    return null;
  }

  // 从 props 中获取事件处理函数
  const listener = props[registrationName];  // props.onClick

  // 对于 disabled 的交互元素，阻止鼠标事件
  if (shouldPreventMouseEvent(registrationName, inst.type, props)) {
    return null;
  }

  return listener;
}
```

**6. 完整的数据流向图**

```
JSX: <div onClick={handleClick} />
              │
              ▼
createElement → React Element { props: { onClick: handleClick } }
              │
              ▼
createFiberFromElement → Fiber { pendingProps: { onClick: handleClick } }
              │
              ▼
commit 阶段 createInstance
              │
              ├─ 创建 DOM: <div>
              │
              ├─ precacheFiberNode(fiber, dom)
              │     └─ dom.__reactFiber$ = fiber
              │
              └─ updateFiberProps(dom, props)
                    └─ dom.__reactProps$ = { onClick: handleClick }

事件触发时：
              │
              ▼
dispatchEvent → 获取 event.target (DOM)
              │
              ▼
getClosestInstanceFromNode(dom) → dom.__reactFiber$ → Fiber
              │
              ▼
accumulateSinglePhaseListeners → 向上遍历 Fiber 树
              │
              ▼
getListener(fiber, 'onClick')
   └─ getFiberCurrentPropsFromNode(fiber.stateNode)
         └─ dom.__reactProps$.onClick → handleClick
```

**7. 为什么不直接存在 Fiber 上？**

你可能会问：为什么要把 props 存在 DOM 节点上，而不是直接从 Fiber.memoizedProps 获取？

原因是 React 使用双缓冲技术（current/workInProgress），在渲染过程中会有两棵 Fiber 树。事件触发时，React 需要确保获取的是"当前已渲染完成"的 props，而不是"正在构建中"的 props。通过 DOM 节点作为桥梁，可以确保获取到稳定的、已提交到屏幕上的 props。

```typescript
// 源码位置：react-dom/src/events/DOMPluginEventSystem.js

function accumulateSinglePhaseListeners(
  targetFiber: Fiber | null,
  reactName: string | null,    // 如 'onClick'
  nativeEventType: string,
  inCapturePhase: boolean,
  accumulateTargetOnly: boolean,
  nativeEvent: AnyNativeEvent,
): Array<DispatchListener> {
  const listeners: Array<DispatchListener> = [];
  let instance = targetFiber;

  // 从目标 Fiber 向上遍历到 HostRoot
  while (instance !== null) {
    const {stateNode, tag} = instance;
    if (tag === HostComponent && stateNode !== null) {
      // 获取事件处理函数
      const listener = getListener(instance, reactName);
      if (listener != null) {
        listeners.push({
          instance,
          listener,
          currentTarget: stateNode,
        });
      }
    }

    if (accumulateTargetOnly) {
      break;
    }
    instance = instance.return;  // 继续向上遍历
  }

  return listeners;
}
```

#### 2.3.10 useTransition 与事件优先级降级机制

当在点击事件中使用 `useTransition` 包裹 `setState` 时，React 会将更新优先级从高优先级（SyncLane）降级为低优先级（TransitionLane）。

**1. useTransition 的基本用法**

```jsx
function SearchComponent() {
  const [isPending, startTransition] = useTransition();
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  const handleChange = (e) => {
    // 高优先级更新：立即更新输入框
    setQuery(e.target.value);

    // 低优先级更新：搜索结果可以延迟
    startTransition(() => {
      setResults(search(e.target.value));
    });
  };

  return (
    <div>
      <input value={query} onChange={handleChange} />
      {isPending && <span>加载中...</span>}
      <ResultList results={results} />
    </div>
  );
}
```

**2. useTransition 的内部实现**

```typescript
// 源码位置：react-reconciler/src/ReactFiberHooks.new.js

function mountTransition(): [boolean, (callback: () => void) => void] {
  // 创建一个 state 来追踪 pending 状态
  const [isPending, setPending] = mountState(false);

  // startTransition 函数绑定 setPending
  const start = startTransition.bind(null, setPending);

  // 将 start 函数存储在 hook 上
  const hook = mountWorkInProgressHook();
  hook.memoizedState = start;

  return [isPending, start];
}
```

**3. startTransition 的核心逻辑**

```typescript
// 源码位置：react-reconciler/src/ReactFiberHooks.new.js

function startTransition(setPending, callback, options) {
  // 1. 保存当前的更新优先级
  const previousPriority = getCurrentUpdatePriority();

  // 2. 设置更新优先级为 ContinuousEventPriority（不是最高，但也不低）
  setCurrentUpdatePriority(
    higherEventPriority(previousPriority, ContinuousEventPriority),
  );

  // 3. 高优先级：设置 isPending = true
  setPending(true);

  // 4. 设置 transition 标记（关键！）
  const prevTransition = ReactCurrentBatchConfig.transition;
  ReactCurrentBatchConfig.transition = {};  // 标记当前处于 transition 中

  try {
    // 5. 高优先级：设置 isPending = false（会被批量处理）
    setPending(false);

    // 6. 执行用户回调（此时 transition 标记存在）
    callback();
  } finally {
    // 7. 恢复之前的优先级和 transition 状态
    setCurrentUpdatePriority(previousPriority);
    ReactCurrentBatchConfig.transition = prevTransition;
  }
}
```

**4. requestUpdateLane 如何判断 Transition 优先级**

当 `setState` 被调用时，会通过 `requestUpdateLane` 获取更新优先级：

```typescript
// 源码位置：react-reconciler/src/ReactFiberWorkLoop.new.js

export function requestUpdateLane(fiber: Fiber): Lane {
  const mode = fiber.mode;

  // Legacy 模式始终返回 SyncLane
  if ((mode & ConcurrentMode) === NoMode) {
    return SyncLane;
  }

  // 🔥 关键：检查是否处于 transition 中
  const isTransition = requestCurrentTransition() !== NoTransition;

  if (isTransition) {
    // 如果在 transition 中，分配一个 TransitionLane
    // TransitionLane 的优先级比 SyncLane 和 DefaultLane 都低
    return claimNextTransitionLane();
  }

  // 获取当前事件优先级
  const updateLane: Lane = getCurrentUpdatePriority();
  if (updateLane !== NoLane) {
    return updateLane;
  }

  // 获取当前事件类型对应的优先级
  const eventLane: Lane = getCurrentEventPriority();
  return eventLane;
}

// 检查是否在 transition 中
export function requestCurrentTransition(): Transition | null {
  return ReactCurrentBatchConfig.transition;
}
```

**5. claimNextTransitionLane 的实现**

```typescript
// 源码位置：react-reconciler/src/ReactFiberLane.new.js

// 16 个 Transition Lane，循环使用
const TransitionLanes: Lanes = 0b0000000001111111111111111000000;
const TransitionLane1: Lane =  0b0000000000000000000000001000000;
// ... TransitionLane2 ~ TransitionLane16

let nextTransitionLane: Lane = TransitionLane1;

export function claimNextTransitionLane(): Lane {
  // 获取当前的 transition lane
  const lane = nextTransitionLane;

  // 移动到下一个 lane
  nextTransitionLane <<= 1;

  // 如果超出 TransitionLanes 范围，循环回到 TransitionLane1
  if ((nextTransitionLane & TransitionLanes) === 0) {
    nextTransitionLane = TransitionLane1;
  }

  return lane;
}
```

**6. 优先级对比图**

```
Lane 优先级（数值越小，优先级越高）：

SyncLane                    0b0000000000000000000000000000001  (最高)
InputContinuousLane         0b0000000000000000000000000000100
DefaultLane                 0b0000000000000000000000000010000
TransitionLane1~16          0b0000000000000000000000001000000  (较低)
                            ~
                            0b0000000001000000000000000000000
RetryLanes                  0b0000111110000000000000000000000
IdleLane                    0b0100000000000000000000000000000  (最低)
```

**7. 完整的优先级降级流程**

```
用户点击按钮
    │
    └─ click 事件触发
         │
         └─ dispatchDiscreteEvent
              │
              └─ 设置 currentUpdatePriority = DiscreteEventPriority (对应 SyncLane)
                   │
                   └─ 执行事件处理函数 handleClick
                        │
                        ├─ setQuery(value)  // 直接调用
                        │    │
                        │    └─ requestUpdateLane(fiber)
                        │         └─ isTransition = false
                        │         └─ 返回 SyncLane (高优先级)
                        │
                        └─ startTransition(() => { setResults(...) })
                             │
                             ├─ ReactCurrentBatchConfig.transition = {}
                             │
                             └─ 执行 callback: setResults(...)
                                  │
                                  └─ requestUpdateLane(fiber)
                                       └─ isTransition = true ✓
                                       └─ 返回 TransitionLane (低优先级)

结果：
- setQuery 的更新使用 SyncLane → 立即同步执行
- setResults 的更新使用 TransitionLane → 可被中断，延迟执行
```

**8. 实际效果示例**

```jsx
function ExpensiveList({ items }) {
  // 假设这是一个渲染很慢的列表
  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>{/* 复杂计算 */}</li>
      ))}
    </ul>
  );
}

function App() {
  const [text, setText] = useState('');
  const [list, setList] = useState([]);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e) => {
    // 输入框立即响应（SyncLane）
    setText(e.target.value);

    // 列表更新可以等待（TransitionLane）
    startTransition(() => {
      setList(generateItems(e.target.value));
    });
  };

  return (
    <div>
      <input value={text} onChange={handleChange} />
      {isPending ? <p>加载中...</p> : <ExpensiveList items={list} />}
    </div>
  );
}

// 调度过程：
// 1. 输入 "a"，产生两个更新：
//    - setText("a")  → SyncLane
//    - setList(...)  → TransitionLane
//
// 2. React 调度器首先处理高优先级的 SyncLane 更新
//    - 输入框立即显示 "a"
//    - isPending = true，显示"加载中..."
//
// 3. 当用户继续输入 "ab" 时：
//    - 新的 SyncLane 更新打断 TransitionLane 的渲染
//    - 输入框立即显示 "ab"
//    - 之前的 TransitionLane 更新被丢弃，使用新的更新
//
// 4. 当用户停止输入后，TransitionLane 更新完成
//    - isPending = false
//    - 显示最终的列表
```

---

### 2.4 createFiberRoot 详解

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

### 2.5 createHostRootFiber 详解

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

### 2.6 render 方法详解

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

### 2.7 updateContainer 详解

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

### 2.8 scheduleUpdateOnFiber 详解（调度入口）

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

### 2.9 markUpdateLaneFromFiberToRoot 详解

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

### 2.10 ensureRootIsScheduled 详解

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

### 2.11 初始化完成后的数据结构

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

### 2.7 Diff 算法详解（reconcileChildren）

React 的 Diff 算法是其高效更新 DOM 的核心。当组件状态更新时，React 需要对比新旧虚拟 DOM 树，找出最小的变更集。

#### 2.7.1 Diff 算法的基本策略

React 的 Diff 算法基于三个假设进行优化，将 O(n³) 的复杂度降低到 O(n)：

```
传统 Diff 算法：O(n³) - 无法接受
React Diff 算法：O(n)  - 基于以下三个假设

假设一：跨层级的 DOM 移动操作很少
        → 只对同层级节点进行比较

假设二：不同类型的组件产生不同的树结构
        → type 不同直接删除重建

假设三：通过 key 可以标识哪些子元素在不同渲染间保持稳定
        → key 相同的节点可以复用
```

#### 2.7.2 reconcileChildren 入口

```typescript
// 源码位置：react-reconciler/src/ReactFiberBeginWork.new.js

export function reconcileChildren(
  current: Fiber | null,
  workInProgress: Fiber,
  nextChildren: any,      // 新的子元素（render 返回值）
  renderLanes: Lanes,
) {
  if (current === null) {
    // 首次渲染：没有旧节点需要对比
    // mountChildFibers 不会标记 Placement（因为整个树都是新的）
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,           // 旧的子 Fiber 为 null
      nextChildren,
      renderLanes,
    );
  } else {
    // 更新渲染：需要进行 diff
    // reconcileChildFibers 会标记副作用（Placement/Deletion 等）
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,  // 旧的子 Fiber
      nextChildren,   // 新的子元素
      renderLanes,
    );
  }
}

// mountChildFibers 和 reconcileChildFibers 都是 ChildReconciler 的实例
// 唯一区别是 shouldTrackSideEffects 参数
export const reconcileChildFibers = ChildReconciler(true);   // 追踪副作用
export const mountChildFibers = ChildReconciler(false);      // 不追踪副作用
```

#### 2.7.3 reconcileChildFibers 核心逻辑

```typescript
// 源码位置：react-reconciler/src/ReactChildFiber.new.js

function reconcileChildFibers(
  returnFiber: Fiber,         // 父 Fiber
  currentFirstChild: Fiber | null,  // 旧的第一个子 Fiber
  newChild: any,              // 新的子元素
  lanes: Lanes,
): Fiber | null {

  // 1. 处理顶层无 key 的 Fragment，展开其 children
  const isUnkeyedTopLevelFragment =
    typeof newChild === 'object' &&
    newChild !== null &&
    newChild.type === REACT_FRAGMENT_TYPE &&
    newChild.key === null;
  if (isUnkeyedTopLevelFragment) {
    newChild = newChild.props.children;
  }

  // 2. 根据 newChild 的类型分发处理
  if (typeof newChild === 'object' && newChild !== null) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE:
        // 单个 React 元素
        return placeSingleChild(
          reconcileSingleElement(returnFiber, currentFirstChild, newChild, lanes)
        );
      case REACT_PORTAL_TYPE:
        // Portal 类型
        return placeSingleChild(
          reconcileSinglePortal(returnFiber, currentFirstChild, newChild, lanes)
        );
      case REACT_LAZY_TYPE:
        // Lazy 组件
        const payload = newChild._payload;
        const init = newChild._init;
        return reconcileChildFibers(returnFiber, currentFirstChild, init(payload), lanes);
    }

    if (isArray(newChild)) {
      // 数组类型的子元素（最常见的多子节点情况）
      return reconcileChildrenArray(returnFiber, currentFirstChild, newChild, lanes);
    }

    if (getIteratorFn(newChild)) {
      // 可迭代对象
      return reconcileChildrenIterator(returnFiber, currentFirstChild, newChild, lanes);
    }
  }

  // 3. 文本节点
  if ((typeof newChild === 'string' && newChild !== '') || typeof newChild === 'number') {
    return placeSingleChild(
      reconcileSingleTextNode(returnFiber, currentFirstChild, '' + newChild, lanes)
    );
  }

  // 4. 其他情况（null, undefined, boolean）- 删除所有旧子节点
  return deleteRemainingChildren(returnFiber, currentFirstChild);
}
```

#### 2.7.4 单节点 Diff（reconcileSingleElement）

当新的子元素是单个 React Element 时：

```typescript
// 源码位置：react-reconciler/src/ReactChildFiber.new.js

function reconcileSingleElement(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  element: ReactElement,
  lanes: Lanes,
): Fiber {
  const key = element.key;
  let child = currentFirstChild;

  // 遍历旧的子 Fiber 链表，尝试找到可复用的节点
  while (child !== null) {
    // 1. 首先比较 key
    if (child.key === key) {
      const elementType = element.type;

      // 2. key 相同，再比较 type
      if (elementType === REACT_FRAGMENT_TYPE) {
        // Fragment 类型
        if (child.tag === Fragment) {
          // key 和 type 都匹配，可以复用
          deleteRemainingChildren(returnFiber, child.sibling);  // 删除多余的兄弟节点
          const existing = useFiber(child, element.props.children);
          existing.return = returnFiber;
          return existing;
        }
      } else {
        // 普通元素
        if (child.elementType === elementType) {
          // key 和 type 都匹配，可以复用！
          deleteRemainingChildren(returnFiber, child.sibling);  // 删除多余的兄弟节点
          const existing = useFiber(child, element.props);       // 复用旧 Fiber
          existing.ref = coerceRef(returnFiber, child, element);
          existing.return = returnFiber;
          return existing;
        }
      }
      // key 相同但 type 不同，删除所有旧节点，跳出循环
      deleteRemainingChildren(returnFiber, child);
      break;
    } else {
      // key 不同，标记删除当前节点，继续查找
      deleteChild(returnFiber, child);
    }
    child = child.sibling;
  }

  // 没有找到可复用的节点，创建新的 Fiber
  if (element.type === REACT_FRAGMENT_TYPE) {
    const created = createFiberFromFragment(
      element.props.children,
      returnFiber.mode,
      lanes,
      element.key,
    );
    created.return = returnFiber;
    return created;
  } else {
    const created = createFiberFromElement(element, returnFiber.mode, lanes);
    created.ref = coerceRef(returnFiber, currentFirstChild, element);
    created.return = returnFiber;
    return created;
  }
}
```

**单节点 Diff 流程图：**

```
新元素: <div key="a">Hello</div>
旧子节点: A(key="a") -> B(key="b") -> C(key="c")

Step 1: 比较 A
        ├─ key 相同 ("a" === "a") ✓
        └─ type 相同 (div === div) ✓
            └─ 复用 A，删除 B 和 C

结果: 复用 A

---

新元素: <span key="a">Hello</span>
旧子节点: A(key="a", div) -> B(key="b") -> C(key="c")

Step 1: 比较 A
        ├─ key 相同 ("a" === "a") ✓
        └─ type 不同 (span !== div) ✗
            └─ 删除所有旧节点，创建新节点

结果: 删除 A, B, C，创建新的 span

---

新元素: <div key="b">Hello</div>
旧子节点: A(key="a") -> B(key="b") -> C(key="c")

Step 1: 比较 A
        └─ key 不同 ("b" !== "a") ✗
            └─ 标记删除 A，继续查找

Step 2: 比较 B
        ├─ key 相同 ("b" === "b") ✓
        └─ type 相同 (div === div) ✓
            └─ 复用 B，删除 C

结果: 删除 A 和 C，复用 B
```

#### 2.7.5 多节点 Diff（reconcileChildrenArray）

当子元素是数组时，React 使用更复杂的算法来处理可能的增、删、移动操作：

```typescript
// 源码位置：react-reconciler/src/ReactChildFiber.new.js

function reconcileChildrenArray(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChildren: Array<*>,
  lanes: Lanes,
): Fiber | null {
  let resultingFirstChild: Fiber | null = null;  // 新 Fiber 链表的头
  let previousNewFiber: Fiber | null = null;     // 上一个新 Fiber

  let oldFiber = currentFirstChild;
  let lastPlacedIndex = 0;  // 最后一个不需要移动的旧节点索引
  let newIdx = 0;
  let nextOldFiber = null;

  // ==================== 第一轮遍历 ====================
  // 从左到右同时遍历新旧数组，尝试复用
  for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
    if (oldFiber.index > newIdx) {
      // 旧 Fiber 的索引大于新索引，说明有节点被删除了
      nextOldFiber = oldFiber;
      oldFiber = null;
    } else {
      nextOldFiber = oldFiber.sibling;
    }

    // 尝试复用或创建新 Fiber
    const newFiber = updateSlot(returnFiber, oldFiber, newChildren[newIdx], lanes);

    if (newFiber === null) {
      // key 不匹配，无法继续复用，跳出第一轮循环
      if (oldFiber === null) {
        oldFiber = nextOldFiber;
      }
      break;
    }

    if (shouldTrackSideEffects) {
      if (oldFiber && newFiber.alternate === null) {
        // 匹配到 slot 但没有复用（type 变了），删除旧节点
        deleteChild(returnFiber, oldFiber);
      }
    }

    // 判断节点是否需要移动
    lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);

    // 构建新 Fiber 链表
    if (previousNewFiber === null) {
      resultingFirstChild = newFiber;
    } else {
      previousNewFiber.sibling = newFiber;
    }
    previousNewFiber = newFiber;
    oldFiber = nextOldFiber;
  }

  // ==================== 第一轮结束后的情况 ====================

  // 情况1: 新数组遍历完了，删除剩余的旧节点
  if (newIdx === newChildren.length) {
    deleteRemainingChildren(returnFiber, oldFiber);
    return resultingFirstChild;
  }

  // 情况2: 旧链表遍历完了，剩余的新节点都是插入
  if (oldFiber === null) {
    for (; newIdx < newChildren.length; newIdx++) {
      const newFiber = createChild(returnFiber, newChildren[newIdx], lanes);
      if (newFiber === null) continue;
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
    }
    return resultingFirstChild;
  }

  // ==================== 第二轮遍历 ====================
  // 情况3: 新旧数组都没遍历完，需要处理移动的情况

  // 将剩余的旧节点放入 Map 中，以 key（或 index）为键
  const existingChildren = mapRemainingChildren(returnFiber, oldFiber);

  // 遍历剩余的新节点
  for (; newIdx < newChildren.length; newIdx++) {
    // 从 Map 中查找可复用的旧节点
    const newFiber = updateFromMap(
      existingChildren,
      returnFiber,
      newIdx,
      newChildren[newIdx],
      lanes,
    );

    if (newFiber !== null) {
      if (shouldTrackSideEffects) {
        if (newFiber.alternate !== null) {
          // 复用了旧节点，从 Map 中移除
          existingChildren.delete(newFiber.key === null ? newIdx : newFiber.key);
        }
      }
      // 判断是否需要移动
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
    }
  }

  // 删除 Map 中剩余的旧节点（它们在新数组中不存在）
  if (shouldTrackSideEffects) {
    existingChildren.forEach(child => deleteChild(returnFiber, child));
  }

  return resultingFirstChild;
}
```

#### 2.7.6 节点移动判断（placeChild）

```typescript
function placeChild(
  newFiber: Fiber,
  lastPlacedIndex: number,
  newIndex: number,
): number {
  newFiber.index = newIndex;

  if (!shouldTrackSideEffects) {
    newFiber.flags |= Forked;
    return lastPlacedIndex;
  }

  const current = newFiber.alternate;
  if (current !== null) {
    const oldIndex = current.index;
    if (oldIndex < lastPlacedIndex) {
      // 旧索引 < lastPlacedIndex，说明这个节点需要向右移动
      // 标记为需要 Placement
      newFiber.flags |= Placement;
      return lastPlacedIndex;
    } else {
      // 不需要移动，更新 lastPlacedIndex
      return oldIndex;
    }
  } else {
    // 新节点（没有 alternate），需要插入
    newFiber.flags |= Placement;
    return lastPlacedIndex;
  }
}
```

**移动判断的核心思想：**

```
核心原则：参照物是最后一个不需要移动的节点的旧索引（lastPlacedIndex）

如果当前节点的旧索引 < lastPlacedIndex，说明它原本在参照物的左边，
但现在需要移动到参照物的右边，所以标记为需要移动。

示例：
旧: A(0) B(1) C(2) D(3)
新: A    C    D    B

遍历过程：
1. A: oldIndex=0, lastPlacedIndex=0
      oldIndex(0) >= lastPlacedIndex(0) ✓ 不移动
      lastPlacedIndex = 0

2. C: oldIndex=2, lastPlacedIndex=0
      oldIndex(2) >= lastPlacedIndex(0) ✓ 不移动
      lastPlacedIndex = 2

3. D: oldIndex=3, lastPlacedIndex=2
      oldIndex(3) >= lastPlacedIndex(2) ✓ 不移动
      lastPlacedIndex = 3

4. B: oldIndex=1, lastPlacedIndex=3
      oldIndex(1) < lastPlacedIndex(3) ✗ 需要移动！
      标记 B 为 Placement

结果：只需要移动 B 到最后
```

#### 2.7.7 多节点 Diff 完整示例

```
示例1：节点更新（无增删移动）
旧: A(key=a) -> B(key=b) -> C(key=c)
新: [A', B', C']  (内容更新，key不变)

第一轮遍历：
  A: key 匹配，type 匹配 → 复用
  B: key 匹配，type 匹配 → 复用
  C: key 匹配，type 匹配 → 复用
结果: 全部复用，只更新 props

---

示例2：节点删除
旧: A -> B -> C -> D
新: [A, C]

第一轮遍历：
  A: 复用
  C: key="c" !== oldFiber(B).key="b" → 跳出循环
第二轮遍历（Map）：
  C: 从 Map 中找到 C → 复用
剩余旧节点 B, D 被删除

---

示例3：节点新增
旧: A -> B
新: [A, B, C, D]

第一轮遍历：
  A: 复用
  B: 复用
oldFiber === null，进入情况2
剩余新节点 C, D 全部创建插入

---

示例4：节点移动
旧: A(0) -> B(1) -> C(2) -> D(3)
新: [D, A, B, C]

第一轮遍历：
  D: key="d" !== oldFiber(A).key="a" → 跳出循环

第二轮遍历（Map）：
  existingChildren = {a: A, b: B, c: C, d: D}

  D: oldIndex=3, lastPlacedIndex=0 → 3 >= 0 → 不移动, lastPlacedIndex=3
  A: oldIndex=0, lastPlacedIndex=3 → 0 < 3  → 移动！
  B: oldIndex=1, lastPlacedIndex=3 → 1 < 3  → 移动！
  C: oldIndex=2, lastPlacedIndex=3 → 2 < 3  → 移动！

结果: D 不动，A、B、C 依次移动到 D 后面
实际 DOM 操作: 3 次移动
```

#### 2.7.8 不同类型组件的 Diff 处理

在 `beginWork` 阶段，不同类型的组件有不同的处理方式：

**1. 函数组件（FunctionComponent）**

```typescript
function updateFunctionComponent(
  current, workInProgress, Component, nextProps, renderLanes
) {
  // 1. 执行函数组件，获取新的子元素
  let nextChildren = renderWithHooks(
    current,
    workInProgress,
    Component,
    nextProps,
    context,
    renderLanes,
  );

  // 2. 检查是否可以 bailout（跳过更新）
  if (current !== null && !didReceiveUpdate) {
    bailoutHooks(current, workInProgress, renderLanes);
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
  }

  // 3. 协调子节点
  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  return workInProgress.child;
}
```

**2. 类组件（ClassComponent）**

```typescript
function updateClassComponent(
  current, workInProgress, Component, nextProps, renderLanes
) {
  // 1. 获取或创建实例
  const instance = workInProgress.stateNode;

  // 2. 调用生命周期方法，判断是否需要更新
  const shouldUpdate = checkShouldComponentUpdate(
    workInProgress,
    Component,
    oldProps,
    newProps,
    oldState,
    newState,
    nextContext,
  );

  if (shouldUpdate) {
    // 3. 调用 render 方法获取新的子元素
    nextChildren = instance.render();
  }

  // 4. 协调子节点
  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  return workInProgress.child;
}
```

**3. 原生 DOM 组件（HostComponent）**

```typescript
function updateHostComponent(current, workInProgress, renderLanes) {
  const type = workInProgress.type;
  const nextProps = workInProgress.pendingProps;

  // 1. 获取子元素
  let nextChildren = nextProps.children;

  // 2. 特殊处理：如果只有文本子节点，直接设置文本内容
  const isDirectTextChild = shouldSetTextContent(type, nextProps);
  if (isDirectTextChild) {
    // 不创建 HostText Fiber，直接在 DOM 节点上设置文本
    nextChildren = null;
  }

  // 3. 标记 ref
  markRef(current, workInProgress);

  // 4. 协调子节点
  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  return workInProgress.child;
}
```

**4. Memo 组件（MemoComponent）**

```typescript
function updateMemoComponent(
  current, workInProgress, Component, nextProps, renderLanes
) {
  if (current !== null) {
    const prevProps = current.memoizedProps;

    // 使用 compare 函数（默认是 shallowEqual）比较 props
    if (shallowEqual(prevProps, nextProps) && current.ref === workInProgress.ref) {
      // props 没变，跳过更新
      didReceiveUpdate = false;
      // ... bailout 逻辑
    }
  }

  // props 变了或首次渲染，走正常更新流程
  return updateSimpleMemoComponent(/*...*/);
}
```

#### 2.7.9 Diff 算法流程图总结

```
reconcileChildFibers(returnFiber, currentFirstChild, newChild, lanes)
                │
                ├─ newChild 是单个元素？
                │       │
                │       └─ reconcileSingleElement
                │              │
                │              └─ 遍历旧 Fiber 链表
                │                    │
                │                    ├─ key 相同 && type 相同 → 复用
                │                    ├─ key 相同 && type 不同 → 删除所有，创建新的
                │                    └─ key 不同 → 删除当前，继续查找
                │
                ├─ newChild 是数组？
                │       │
                │       └─ reconcileChildrenArray
                │              │
                │              ├─ 第一轮：从左到右遍历
                │              │     ├─ key 匹配 → 复用/更新
                │              │     └─ key 不匹配 → 跳出循环
                │              │
                │              ├─ 新数组遍历完 → 删除剩余旧节点
                │              │
                │              ├─ 旧链表遍历完 → 插入剩余新节点
                │              │
                │              └─ 第二轮：Map 查找
                │                    ├─ 构建 existingChildren Map
                │                    ├─ 遍历剩余新节点
                │                    │     └─ 从 Map 中查找可复用节点
                │                    └─ 删除 Map 中剩余的旧节点
                │
                └─ newChild 是文本？
                        │
                        └─ reconcileSingleTextNode
```

#### 2.7.10 Diff 算法的注意事项与最佳实践

**1. key 的重要性**

```jsx
// ❌ 错误：使用 index 作为 key
{items.map((item, index) => (
  <Item key={index} data={item} />
))}
// 问题：当列表顺序变化时，React 会错误地复用节点，导致状态混乱

// ✅ 正确：使用稳定且唯一的 id
{items.map(item => (
  <Item key={item.id} data={item} />
))}
```

**2. 避免不必要的节点类型变化**

```jsx
// ❌ 类型变化会导致整个子树重建
{isLoggedIn ? <UserPanel /> : <LoginForm />}

// ✅ 如果结构相似，考虑使用同一组件
<AuthPanel isLoggedIn={isLoggedIn} />
```

**3. 列表操作的优化**

```
在列表头部插入：
旧: B -> C -> D
新: A -> B -> C -> D

React 的处理：
第一轮：A.key !== B.key → 跳出
第二轮：A 是新增，B/C/D 从 Map 中找到并复用
       但 B/C/D 的 oldIndex 都 < lastPlacedIndex(新插入的A没有oldIndex)
       所以 B/C/D 都被标记为移动！

优化建议：
- 频繁在头部插入的场景，考虑使用反转数组
- 或者使用虚拟列表只渲染可见项
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

Scheduler 是一个**独立于 React 的通用任务调度库**，React 通过它来实现任务的优先级调度和时间切片。

### 4.0 Scheduler 与 React 的关系

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Scheduler（独立的包）                              │
│                                                                         │
│   • 不依赖 React，可以单独使用                                            │
│   • 负责任务的调度、优先级管理、时间切片                                    │
│   • 使用小顶堆管理任务队列                                                │
│   • 通过 MessageChannel/setTimeout 实现宏任务调度                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    ▲
                                    │ scheduleCallback(priority, callback)
                                    │
┌─────────────────────────────────────────────────────────────────────────┐
│                        React Reconciler                                  │
│                                                                         │
│   • 将 Lane 优先级转换为 Scheduler 优先级                                 │
│   • 将 performConcurrentWorkOnRoot 作为 callback 传给 Scheduler          │
│   • Scheduler 回调执行时，React 开始 render/commit 工作                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**为什么要独立？**
1. 关注点分离：Scheduler 专注于"何时执行"，React 专注于"执行什么"
2. 可复用性：其他框架也可以使用 Scheduler
3. 可测试性：可以独立测试调度逻辑

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

### 4.9 React 如何调度多个 setState

当多个 `setState` 同时触发时，React 的调度逻辑如下：

#### 4.9.1 同步场景下的批量更新

```jsx
function handleClick() {
  setState1(1);  // 触发更新
  setState2(2);  // 触发更新
  setState3(3);  // 触发更新
}
// 这三个 setState 会被合并为一次更新
```

**流程解析：**

```
handleClick 执行
         │
         ├─ setState1(1)
         │       │
         │       ├─ dispatchSetState(fiber, queue, action)
         │       │       │
         │       │       ├─ requestUpdateLane() → SyncLane
         │       │       ├─ 创建 Update 对象，加入 queue
         │       │       └─ scheduleUpdateOnFiber(fiber, SyncLane)
         │       │               │
         │       │               ├─ markUpdateLaneFromFiberToRoot()
         │       │               │     // 向上标记 lane 到 root
         │       │               │
         │       │               └─ ensureRootIsScheduled(root)
         │       │                       │
         │       │                       ├─ 首次：注册 flushSyncCallbacks 到微任务
         │       │                       │        scheduleMicrotask(flushSyncCallbacks)
         │       │                       │
         │       │                       └─ root.callbackPriority = SyncLane
         │       │
         ├─ setState2(2)
         │       │
         │       └─ ... 同上流程 ...
         │               │
         │               └─ ensureRootIsScheduled(root)
         │                       │
         │                       └─ 🔥 关键判断：
         │                          existingCallbackPriority === newCallbackPriority
         │                          (SyncLane === SyncLane) → true
         │                          直接 return，复用之前的任务！
         │
         └─ setState3(3) → 同上，也复用之前的任务

handleClick 执行完毕
         │
         ▼
微任务执行：flushSyncCallbacks()
         │
         └─ 执行 performSyncWorkOnRoot(root)
                 │
                 └─ 一次性处理所有 3 个 Update
```

#### 4.9.2 ensureRootIsScheduled 的去重机制

```typescript
// 源码位置：react-reconciler/src/ReactFiberWorkLoop.new.js

function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  const existingCallbackNode = root.callbackNode;

  // ... 省略过期任务处理 ...

  const nextLanes = getNextLanes(root, workInProgressRootRenderLanes);

  if (nextLanes === NoLanes) {
    // 没有任务了
    root.callbackNode = null;
    root.callbackPriority = NoLane;
    return;
  }

  const newCallbackPriority = getHighestPriorityLane(nextLanes);
  const existingCallbackPriority = root.callbackPriority;

  // 🔥🔥🔥 关键：优先级相同时，复用现有任务
  if (existingCallbackPriority === newCallbackPriority) {
    // 优先级没变，不需要重新调度，直接复用现有任务
    return;
  }

  // 优先级变了（新的更高）
  if (existingCallbackNode != null) {
    // 取消旧任务
    cancelCallback(existingCallbackNode);
  }

  // 注册新任务
  let newCallbackNode;
  if (newCallbackPriority === SyncLane) {
    // 同步优先级：通过微任务调度
    scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root));
    scheduleMicrotask(flushSyncCallbacks);
    newCallbackNode = null;
  } else {
    // 其他优先级：通过 Scheduler 调度
    newCallbackNode = scheduleCallback(
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root),
    );
  }

  root.callbackPriority = newCallbackPriority;
  root.callbackNode = newCallbackNode;
}
```

#### 4.9.3 不同优先级的 setState

```jsx
function handleClick() {
  // 高优先级（点击事件中，SyncLane）
  setCount(1);

  // 低优先级（useTransition 包裹）
  startTransition(() => {
    setList(newList);
  });
}
```

**流程解析：**

```
handleClick 执行
         │
         ├─ setCount(1)
         │       │
         │       └─ requestUpdateLane() → SyncLane (高优先级)
         │               │
         │               └─ scheduleUpdateOnFiber → ensureRootIsScheduled
         │                       │
         │                       └─ 注册 SyncLane 任务
         │                          root.callbackPriority = SyncLane
         │
         └─ startTransition(() => setList(newList))
                 │
                 ├─ ReactCurrentBatchConfig.transition = {}
                 │
                 └─ setList(newList)
                         │
                         └─ requestUpdateLane() → TransitionLane (低优先级)
                                 │
                                 └─ scheduleUpdateOnFiber → ensureRootIsScheduled
                                         │
                                         └─ existingCallbackPriority(Sync) !== newCallbackPriority(Transition)
                                            由于 Sync > Transition，不取消现有任务
                                            Transition 更新会在 Sync 任务完成后再处理

微任务执行
         │
         └─ flushSyncCallbacks() → performSyncWorkOnRoot
                 │
                 ├─ 处理 SyncLane 的 setCount 更新
                 │
                 └─ render + commit 完成后
                         │
                         └─ ensureRootIsScheduled(root)
                                 │
                                 └─ 检测到还有 TransitionLane 待处理
                                    注册新的 Scheduler 任务
```

#### 4.9.4 高优先级打断低优先级

```jsx
// 场景：低优先级任务正在执行时，用户触发高优先级更新

function App() {
  const [list, setList] = useState([]);
  const [count, setCount] = useState(0);

  // 低优先级更新（渲染大列表）
  useEffect(() => {
    startTransition(() => {
      setList(generateLargeList());  // TransitionLane
    });
  }, []);

  // 高优先级更新（用户点击）
  const handleClick = () => {
    setCount(c => c + 1);  // SyncLane
  };
}
```

**打断与恢复流程：**

```
1. Transition 更新开始执行
         │
         └─ performConcurrentWorkOnRoot(root)
                 │
                 └─ renderRootConcurrent(root, TransitionLanes)
                         │
                         └─ workLoopConcurrent()
                                 │
                                 ├─ beginWork(fiber1) ✓
                                 ├─ beginWork(fiber2) ✓
                                 ├─ beginWork(fiber3) ✓
                                 │
                                 └─ 用户点击！触发高优先级更新

2. 高优先级更新调度
         │
         └─ dispatchSetState → scheduleUpdateOnFiber
                 │
                 └─ ensureRootIsScheduled(root)
                         │
                         ├─ newCallbackPriority = SyncLane
                         ├─ existingCallbackPriority = TransitionLane
                         │
                         └─ SyncLane > TransitionLane
                            取消现有的 Transition 任务
                            注册新的 Sync 任务

3. Scheduler workLoop 检测到时间片用完或被打断
         │
         └─ shouldYieldToHost() → true
                 │
                 └─ workLoopConcurrent 跳出循环
                         │
                         └─ performConcurrentWorkOnRoot 返回 continuation
                            return performConcurrentWorkOnRoot.bind(null, root)
                            // 但这个 continuation 不会被执行，因为任务被取消了

4. 高优先级任务执行
         │
         └─ flushSyncCallbacks() → performSyncWorkOnRoot
                 │
                 └─ renderRootSync() + commitRoot()
                         │
                         └─ 高优先级更新完成

5. 低优先级任务恢复
         │
         └─ ensureRootIsScheduled(root) [在 commit 结束后调用]
                 │
                 └─ 检测到还有 TransitionLanes 待处理
                         │
                         └─ 重新注册 Scheduler 任务
                            performConcurrentWorkOnRoot.bind(null, root)

6. Transition 更新重新开始
         │
         └─ performConcurrentWorkOnRoot(root)
                 │
                 └─ 🔥 从头开始 render（不是从中断点恢复）
                    prepareFreshStack(root, lanes) // 重置 workInProgress
```

#### 4.9.5 为什么要从头开始而不是从中断点恢复？

```
原因：高优先级更新可能改变了状态

假设场景：
  - Transition 更新基于 state = {count: 0} 渲染
  - 高优先级更新把 state 改成 {count: 1}
  - Transition 已经渲染了一部分节点（基于旧 state）

如果从中断点恢复：
  - 前半部分基于 count: 0 渲染 ❌
  - 后半部分基于 count: 1 渲染 ❌
  - 结果不一致！

所以必须从头开始：
  - 整个 Transition 更新基于最新的 count: 1 渲染 ✓
  - 结果一致！

这就是为什么 React 使用 workInProgress 树而不是直接修改 current 树
workInProgress 树可以被丢弃，从头重建
```

### 4.10 时间切片的实现细节

#### 4.10.1 为什么使用 MessageChannel 而不是 setTimeout？

```typescript
// setTimeout 的问题：最小延迟 4ms（嵌套调用时）
setTimeout(callback, 0);  // 实际延迟 4ms+

// MessageChannel 没有这个限制
const channel = new MessageChannel();
channel.port1.onmessage = callback;
channel.port2.postMessage(null);  // 几乎立即触发

// 对于 60fps 的目标（16.6ms 一帧）
// 4ms 的延迟意味着丢失 25% 的时间！
```

#### 4.10.2 时间切片的默认时长

```typescript
let frameInterval = 5;  // 默认 5ms

// 每个时间切片执行 5ms 的工作，然后让出控制权
// 这样浏览器有足够时间处理：
// - 用户输入
// - 动画
// - 布局和绘制
```

#### 4.10.3 workLoopConcurrent vs workLoopSync

```typescript
// 同步版本：不可中断
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
    // 不检查是否需要让出，一直执行到完成
  }
}

// 并发版本：可中断
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYieldToHost()) {
    performUnitOfWork(workInProgress);
    // 每次循环都检查是否需要让出
  }
}

// shouldYieldToHost 的判断
function shouldYieldToHost() {
  const timeElapsed = getCurrentTime() - startTime;

  if (timeElapsed < frameInterval) {  // 5ms
    // 时间片没用完，继续执行
    return false;
  }

  // 时间片用完了
  // 检查是否有用户输入等待处理
  if (isInputPending !== null) {
    return isInputPending();  // 使用 navigator.scheduling.isInputPending
  }

  return true;  // 让出控制权
}
```

### 4.11 任务饿死的防止机制

#### 4.11.1 过期时间机制

```typescript
// 每个优先级对应不同的超时时间
var IMMEDIATE_PRIORITY_TIMEOUT = -1;          // 立即过期
var USER_BLOCKING_PRIORITY_TIMEOUT = 250;     // 250ms 后过期
var NORMAL_PRIORITY_TIMEOUT = 5000;           // 5s 后过期
var LOW_PRIORITY_TIMEOUT = 10000;             // 10s 后过期
var IDLE_PRIORITY_TIMEOUT = maxSigned31BitInt; // 永不过期

// 任务创建时计算过期时间
expirationTime = startTime + timeout;
```

#### 4.11.2 过期任务的同步执行

```typescript
// 源码位置：react-reconciler/src/ReactFiberWorkLoop.new.js

function ensureRootIsScheduled(root, currentTime) {
  // 检查是否有任务已过期（饿死）
  markStarvedLanesAsExpired(root, currentTime);
  // ...
}

// 源码位置：react-reconciler/src/ReactFiberLane.new.js
export function markStarvedLanesAsExpired(root, currentTime) {
  const pendingLanes = root.pendingLanes;
  const expirationTimes = root.expirationTimes;

  let lanes = pendingLanes;
  while (lanes > 0) {
    const index = pickArbitraryLaneIndex(lanes);
    const lane = 1 << index;
    const expirationTime = expirationTimes[index];

    if (expirationTime === NoTimestamp) {
      // 还没设置过期时间，设置一个
      expirationTimes[index] = computeExpirationTime(lane, currentTime);
    } else if (expirationTime <= currentTime) {
      // 🔥 已过期！标记到 expiredLanes
      root.expiredLanes |= lane;
    }

    lanes &= ~lane;
  }
}

// performConcurrentWorkOnRoot 中的判断
const shouldTimeSlice =
  !includesBlockingLane(root, lanes) &&
  !includesExpiredLane(root, lanes) &&  // 🔥 如果有过期任务
  !didTimeout;

if (shouldTimeSlice) {
  renderRootConcurrent(root, lanes);  // 可中断
} else {
  renderRootSync(root, lanes);         // 🔥 同步执行，不可中断
}
```

### 4.12 调度流程完整图解

```
setState/dispatchSetState
         │
         ▼
requestUpdateLane(fiber) ────────────────────────────────┐
         │                                                │
         │  判断优先级：                                    │
         │  - 事件上下文？ → DiscreteEventPriority         │
         │  - Transition？ → TransitionLane               │
         │  - 默认 → DefaultLane                          │
         │                                                │
         ▼                                                │
scheduleUpdateOnFiber(fiber, lane)                        │
         │                                                │
         ├─ markUpdateLaneFromFiberToRoot()               │
         │     // 向上遍历，在每个 fiber.lanes 标记 lane   │
         │     // 在每个 fiber.childLanes 标记 lane       │
         │     // 直到 root.pendingLanes |= lane          │
         │                                                │
         └─ ensureRootIsScheduled(root, currentTime)      │
                 │                                        │
                 ├─ markStarvedLanesAsExpired() // 防饿死   │
                 │                                        │
                 ├─ getNextLanes() // 获取最高优先级 lanes  │
                 │                                        │
                 ├─ 去重检查：                             │
                 │   existingCallbackPriority === newPriority?
                 │   │                                    │
                 │   ├─ 是 → return（复用现有任务）        │
                 │   │                                    │
                 │   └─ 否 → 取消旧任务，创建新任务        │
                 │                                        │
                 └─ 创建调度任务：                         │
                     │                                    │
                     ├─ SyncLane?                         │
                     │   └─ scheduleMicrotask(flushSyncCallbacks)
                     │                                    │
                     └─ 其他 Lane?                        │
                         └─ scheduleCallback(             │
                              priority,                   │
                              performConcurrentWorkOnRoot │
                            )                             │
                                                          │
┌─────────────────────────────────────────────────────────┘
│  Scheduler 调度层
│
▼
scheduleCallback(priority, callback)
         │
         ├─ 计算 expirationTime = now + timeout
         │
         ├─ 创建 task = { callback, expirationTime, ... }
         │
         ├─ push(taskQueue, task)  // 小顶堆
         │
         └─ requestHostCallback(flushWork)
                 │
                 └─ schedulePerformWorkUntilDeadline()
                         │
                         └─ MessageChannel.postMessage()
                                 │
                                 ▼
                         [下一个宏任务]
                                 │
                                 ▼
                   performWorkUntilDeadline()
                                 │
                                 └─ flushWork()
                                         │
                                         └─ workLoop()
                                                 │
                                                 └─ while (task && !shouldYield)
                                                         │
                                                         ├─ task.callback()
                                                         │   // = performConcurrentWorkOnRoot
                                                         │
                                                         └─ 返回值？
                                                             │
                                                             ├─ function → task.callback = 返回值
                                                             │              (任务未完成，继续调度)
                                                             │
                                                             └─ null → pop(taskQueue)
                                                                        (任务完成)
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
