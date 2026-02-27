## React 内部执行机制：以一次 `setState` 更新为主线

在进入流程细节之前，先把 React 在这条链路中用到的几个**核心数据结构**捋一下，后文会不停地出现它们的身影：

- **Fiber（工作单元节点）**：  
  - 描述 UI 树上的一个节点（函数组件 / class 组件 / DOM 节点等）；  
  - 关键字段（节选，按「结构关系 / 渲染状态 / 副作用与优先级」三类来记）：  
    - **结构关系相关**：  
      - `tag`：节点类型（FunctionComponent、ClassComponent、HostComponent、HostRoot 等）；  
      - `key`：同一层级下用于 diff 的稳定标识；  
      - `type`：组件本身（函数 / class / 标签字符串等）；  
      - `elementType`：和 `type` 接近，更多用于区分包裹过的组件（memo / forwardRef 等）；  
      - `child` / `sibling` / `return`：指向第一个子节点、兄弟节点和父节点，组成 Fiber 树；  
      - `index`：在同一层兄弟节点中的位置（用于 children diff）；  
      - `ref`：当前节点关联的 ref（函数 ref 或对象 ref）。  
    - **渲染状态相关**：  
      - `stateNode`：宿主环境实例（如 DOM 节点）或 class 组件实例；  
      - `pendingProps`：本次 render 即将使用的 props；  
      - `memoizedProps`：上一次完成 render 后保存下来的 props；  
      - `memoizedState`：上一次完成 render 后的 state（函数组件这里挂 Hook 链表）；  
      - `updateQueue`：类组件或 HostRoot 上挂的更新队列 / 副作用队列；  
      - `dependencies`：对 context / subscriptions 等依赖的描述；  
      - `alternate`：指向当前 Fiber 的「另一棵树」上的对应节点（current <-> workInProgress 双缓存）；  
      - `mode`：当前 Fiber 所在树的运行模式标记（ConcurrentMode、StrictMode 等）。  
    - **副作用与优先级相关**：  
      - `flags` / `subtreeFlags`：本节点及子树本次渲染产生的副作用标记（Placement / Update / Deletion / Passive / Layout 等）；  
      - `lanes` / `childLanes`：该节点及子树上挂着的所有更新优先级集合；  
      - `deletions`：本节点要删除的子 Fiber 列表（在 commit 阶段一并处理）。

- **Hook（函数组件的状态链表节点）**：  
  - 只存在于函数组件 Fiber 的 `memoizedState` 上，以**单向链表**形式串联多个 Hook 调用；  
  - 关键字段：  
    - `memoizedState`：当前 Hook 的状态值（比如 `useState` 的 state、`useReducer` 的 state、effect 的依赖数组等）；  
    - `baseState` / `baseQueue`：在多次打断 / 重算时作为「基线」的状态和队列；  
    - `queue`：对于 `useState` / `useReducer`，这里存放所有待处理的 Update 链表；  
    - `next`：指向下一个 Hook，保证多次 `useXXX` 调用顺序一致。

- **UpdateQueue（更新队列，用于收集 setState）**：  
  - 挂在 Hook 的 `queue` 字段上，描述这一条 state 上所有待处理的更新；  
  - 关键字段（以 `useState` 为例）：  
    - `pending`：指向一个**环形链表**中的最后一个 Update，链表上存放这次渲染前尚未消费的所有更新；  
    - `interleaved`：并发模式下交错进来的更新；  
    - `lanes`：这条队列上所有未处理更新的优先级集合；  
    - `dispatch`：也就是我们在组件里拿到的 `setState`，本质是绑定了 `fiber + queue` 的 `dispatchSetState`；  
    - `lastRenderedReducer` / `lastRenderedState`：上一次渲染时真正用来计算 state 的 reducer 和 state。

- **Update（单次状态更新）**：  
  - 由 `dispatchSetState` 创建，挂在 UpdateQueue 的环形链表里；  
  - 关键字段：  
    - `lane`：这次更新所属的优先级；  
    - `action`：`setState` 传入的值或函数；  
    - `hasEagerState` / `eagerState`：是否已经在调度阶段预先算过一次 state；  
    - `next`：指向下一个 Update，组成环形链表。

- **FiberRoot（根节点容器）**：  
  - 对应一次 `createRoot` / `legacyRender` 创建出来的根；  
  - 关键字段（与本流程相关）：  
    - `current`：指向当前已挂载的 Fiber 树根节点（HostRoot Fiber）；  
    - `finishedWork` / `finishedLanes`：本次 render 完成后待提交的 Fiber 树根及其 lanes；  
    - `pendingLanes` / `suspendedLanes` 等：整棵应用上所有未完成更新的优先级信息；  
    - `callbackNode` / `callbackPriority`：当前通过 Scheduler 注册的调度任务及其优先级。

- **Lane / Lanes（优先级模型）**：  
  - React 18 内部用「位掩码」表示的优先级系统；  
  - 单个 lane 代表一种优先级（如 `SyncLane`、`DefaultLane` 等），`Lanes` 是 lane 的集合；  
  - 在本文主线中，`setState` -> `requestUpdateLane` 会为每次更新分配 lane，后续调度 / 渲染 / 提交都会以 lanes 为依据进行筛选和合并。

### 1. 从业务组件出发：一次 `setState` 调用长什么样？

以项目中的 `AppTest` 组件为例，其中有典型的函数组件 `useState` 调用和点击事件里触发的多次 `setState`：

```typescript
function AppTest () {
  const [state, setState] = useState('state1')
  const [state2, setState2] = useState('state2')
  useEffect(() => {
    console.log('我是没有依赖的useEffect')
    document.getElementById('but').addEventListener('click', navClick)
    return () => console.log('我是没有依赖的useEffect的清除函数')
  }, [])
  useLayoutEffect(() => {
    console.log('我是没有依赖的useLayoutEffect')
    return () => console.log('我是没有依赖的useLayoutEffect的清除函数')
  }, [])
  useEffect(() => {
    console.log('我是每次更新都要执行的useEffect')
    return () => console.log('我是每次更新都要执行的useEffect清除')
  })
  const click = () => {
    setState('change1')
    setState2('change2')
  }
  const navClick = () => {
    setState('change1')
    setState2('change2')
  }
  const time = () => {
    setTimeout(() => {
      setState('changeTime')
      setState2('changeTime2')
    }, 200)
  }
  return (
    <div>
      <span>state1：{state}</span>
      <span>state2：{state2}</span>
      <button onClick={click}>react事件</button>
      <button id='but'>原生事件</button>
      <button onClick={time}>延迟事件</button>
      <Component />
    </div>
  )
}
```

**这一节要记住两点：**
- **`useState` 返回的第二个值就是内部包装过的 `dispatchSetState`；**
- **事件回调中每一次 `setState` 调用，最后都会走到同一个调度入口 `scheduleUpdateOnFiber`。**

下面从 Hooks 的实现开始，一步步顺着这条调用链往下走。

---

### 2. Hook 层：从 `useState` 到 `dispatchSetState`

#### 2.1 React 导出的 `useState` 本体

业务代码里写的 `useState`，实际上是从 `react` 包里导出的函数，它内部会先解析当前的 `dispatcher`，再转调到 reconciler 里的实现：

```typescript
export function useState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  const dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}
```

`resolveDispatcher` 会从全局的 `ReactCurrentDispatcher.current` 上拿到当前渲染阶段对应的一套 Hook 实现（mount / update / rerender 三套）。

#### 2.2 mount 阶段的 `useState`：`mountState`

在首次渲染函数组件时，`dispatcher.useState` 实际会指向 `ReactFiberHooks.new.js` 中的 `mountState`：

```typescript
function mountWorkInProgressHook(): Hook { // 渲染时候的生成并链接hook
    const hook: Hook = {
      memoizedState: null,
      baseState: null,
      baseQueue: null,
      queue: null,
      next: null,
    };
    if (workInProgressHook === null) {
      // 第一个hooks
      currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
    } else {
      // 不是第一个就append到最后去
      workInProgressHook = workInProgressHook.next = hook;
    }
    return workInProgressHook;
}

function mountState<S>(// mount阶段调用的usestate
    initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
    const hook = mountWorkInProgressHook();// 创建hook并且和之前的hook形成链表，之前没有hook就挂载到workinprogress的memoizedState上
    if (typeof initialState === 'function') {
      initialState = initialState();
    }
    hook.memoizedState = hook.baseState = initialState;
    const queue: UpdateQueue<S, BasicStateAction<S>> = {//创建更新队列
      pending: null,
      interleaved: null,
      lanes: NoLanes,
      dispatch: null,
      lastRenderedReducer: basicStateReducer,
      lastRenderedState: (initialState: any),// 上次更新后的state
    };
    hook.queue = queue;
    const dispatch: Dispatch<
      BasicStateAction<S>,
    > = (queue.dispatch = (dispatchSetState.bind( //dispatch函数
      null,
      currentlyRenderingFiber,
      queue,
    ): any));
    return [hook.memoizedState, dispatch];
}
```

**这一段做了几件关键事情：**
- **通过 `mountWorkInProgressHook` 创建一个 Hook，并挂到当前函数组件 Fiber 的 `memoizedState` 链表上；**
- **把初始 state 写到 `hook.memoizedState / baseState`；**
- **为这个 Hook 创建一个 `UpdateQueue`，用于后续收集所有 setState 更新；**
- **最核心：构造 `dispatch`，它是 `dispatchSetState.bind(null, currentlyRenderingFiber, queue)`。

也就是说，`useState` 返回的 `setState` 实际就是一个「预先绑定了 fiber + queue」的 `dispatchSetState`。

#### 2.3 更新阶段的 `useState`：`updateState` / `rerenderState`

后续更新渲染时，`dispatcher.useState` 会变成 `updateState` 或 `rerenderState`，它们都复用了 `updateReducer` 这套通用逻辑。

`updateReducer` 的完整实现如下：

```typescript
function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
    const hook = updateWorkInProgressHook();// 拷贝之前的hook并返回
    const queue = hook.queue;
    if (queue === null) {
        throw new Error(
            'Should have a queue. This is likely a bug in React. Please file an issue.',
        );
    }
    queue.lastRenderedReducer = reducer;
    const current: Hook = (currentHook: any);
    let baseQueue = current.baseQueue;
    const pendingQueue = queue.pending;
    if (pendingQueue !== null) {
        if (baseQueue !== null) {
            const baseFirst = baseQueue.next;
            const pendingFirst = pendingQueue.next;
            baseQueue.next = pendingFirst;
            pendingQueue.next = baseFirst;
        }
        if (__DEV__) {
            if (current.baseQueue !== baseQueue) {
                console.error(
                    'Internal error: Expected work-in-progress queue to be a clone. ' +
                    'This is a bug in React.',
                );
            }
        }
        current.baseQueue = baseQueue = pendingQueue;
        queue.pending = null;
    }
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
                // 优先级不够高，这次更新先不处理
                const clone: Update<S, A> = {
                    lane: updateLane,
                    action: update.action,
                    hasEagerState: update.hasEagerState,
                    eagerState: update.eagerState,
                    next: (null: any),
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
                // 这条更新的优先级足够高，可以在这次渲染中被处理
                if (newBaseQueueLast !== null) {
                    const clone: Update<S, A> = {
                        lane: NoLane,
                        action: update.action,
                        hasEagerState: update.hasEagerState,
                        eagerState: update.eagerState,
                        next: (null: any),
                    };
                    newBaseQueueLast = newBaseQueueLast.next = clone;
                }
                if (update.hasEagerState) {
                    newState = ((update.eagerState: any): S);
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
            newBaseQueueLast.next = (newBaseQueueFirst: any);
        }
        // 比较状态是否不一样，不一样就标记需要更新
        if (!is(newState, hook.memoizedState)) {
            markWorkInProgressReceivedUpdate();
        }
        hook.memoizedState = newState;
        hook.baseState = newBaseState;
        hook.baseQueue = newBaseQueueLast;
        queue.lastRenderedState = newState;
    }
    // 在某次 Fiber 渲染「进行过程中」新进来的那批更新。它们不会马上并入当前这次渲染的计算，但会被单独记录起来，保证之后的调度不会漏掉
    const lastInterleaved = queue.interleaved;
    if (lastInterleaved !== null) {
        let interleaved = lastInterleaved;
        do {
            const interleavedLane = interleaved.lane;
            currentlyRenderingFiber.lanes = mergeLanes(
                currentlyRenderingFiber.lanes,
                interleavedLane,
            );
            markSkippedUpdateLanes(interleavedLane);
            interleaved = ((interleaved: any).next: Update < S, A >);
        } while (interleaved !== lastInterleaved);
    } else if (baseQueue === null) {
        queue.lanes = NoLanes;
    }
    const dispatch: Dispatch<A> = (queue.dispatch: any);
    return [hook.memoizedState, dispatch];
}
```

**结合这段源码，可以更精确地理解「更新阶段 useState 做了什么」：**
- **先把这次渲染前挂在 `queue.pending` 上的所有 Update「并」到 `baseQueue` 上，形成一条要处理的环形链表；**  
- **遍历这条链表：对于不在本次渲染优先级（`renderLanes`）里的 Update，只克隆到新的 `baseQueue`，把它们和对应的 lanes「保留下来」留待下次；**  
- **对于优先级足够的 Update，就用 reducer（`basicStateReducer` 对应 `useState`）一步步计算出 `newState`；**  
- **最终得到这次渲染要用的 `hook.memoizedState`，并更新 `baseState / baseQueue / lastRenderedState` 等字段，为下一轮渲染打好「基线」。**

- **简而言之：`updateReducer` 就是「在考虑优先级的前提下，把这条 Hook 上的所有 Update 合并成一个新的 state」的核心实现。**

---

### 3. 触发更新：`dispatchSetState`

当组件里调用 `setState` 时，实际上就是在调用 `dispatchSetState`：

```typescript
function dispatchSetState<S, A>(// 触发更新
    fiber: Fiber,
    queue: UpdateQueue<S, A>,
    action: A,
) {
    if (__DEV__) {
        if (typeof arguments[3] === 'function') {
            console.error(
                "State updates from the useState() and useReducer() Hooks don't support the " +
                'second callback argument. To execute a side effect after ' +
                'rendering, declare it in the component body with useEffect().',
            );
        }
    }
    const lane = requestUpdateLane(fiber);
    const update: Update<S, A> = {
        lane,
        action,
        hasEagerState: false,
        eagerState: null,
        next: (null: any),
    };
    // render阶段调用setState
    if (isRenderPhaseUpdate(fiber)) {
        enqueueRenderPhaseUpdate(queue, update);
    } else {
        //正常更新
        enqueueUpdate(fiber, queue, update, lane); // 把更新对象加入fiber的更新列表
        const alternate = fiber.alternate;
        if (
            fiber.lanes === NoLanes &&
            (alternate === null || alternate.lanes === NoLanes)
        ) {
            const lastRenderedReducer = queue.lastRenderedReducer;
            if (lastRenderedReducer !== null) {
                let prevDispatcher;
                if (__DEV__) {
                    prevDispatcher = ReactCurrentDispatcher.current;
                    ReactCurrentDispatcher.current = InvalidNestedHooksDispatcherOnUpdateInDEV;
                }
                try {
                    const currentState: S = (queue.lastRenderedState: any);
                    const eagerState = lastRenderedReducer(currentState, action);
                    update.hasEagerState = true;
                    update.eagerState = eagerState;
                    if (is(eagerState, currentState)) {
                        return;
                    }
                } catch (error) {
                    
                } finally {
                    if (__DEV__) {
                        ReactCurrentDispatcher.current = prevDispatcher;
                    }
                }
            }
        }
        const eventTime = requestEventTime();
        const root = scheduleUpdateOnFiber(fiber, lane, eventTime);
        if (root !== null) {
            entangleTransitionUpdate(root, queue, lane);
        }
    }
    markUpdateInDevTools(fiber, lane, action);
}
function enqueueRenderPhaseUpdate<S, A>(
    queue: UpdateQueue<S, A>,
    update: Update<S, A>,
) {
    // This is a render phase update. Stash it in a lazily-created map of
    // queue -> linked list of updates. After this render pass, we'll restart
    // and apply the stashed updates on top of the work-in-progress hook.
    didScheduleRenderPhaseUpdateDuringThisPass = didScheduleRenderPhaseUpdate = true;
    const pending = queue.pending;
    if (pending === null) {
        // This is the first update. Create a circular list.
        update.next = update;
    } else {
        update.next = pending.next;
        pending.next = update;
    }
    queue.pending = update;
}
```

**这里可以分为三个重要步骤：**

1. **确定优先级：**  
   - 通过 `requestUpdateLane(fiber)` 计算这次更新所在的 `lane`（对应不同事件类型优先级）。

2. **构造 Update 并入队：**  
   - 创建一个 `update` 对象 `{ lane, action, ... }`；  
   - 如果是渲染阶段更新（例如 reducer 里再次 `setState`），特殊处理放到 `render phase` 队列；  
   - 否则通过 `enqueueUpdate(fiber, queue, update, lane)` 把 Update 记录到 Fiber 对应 Hook 的更新队列中。

3. **调度：进入 Fiber 调度体系：**  
   - 计算事件时间 `eventTime = requestEventTime()`；  
   - 调用 **`scheduleUpdateOnFiber(fiber, lane, eventTime)`** 把这次更新上升到整个应用的调度入口；  
   - `entangleTransitionUpdate` 负责和 concurrent 模式下的 transition 关联，这里只要知道它是 transition 相关的附加处理即可。

**到这里为止，`setState` 做的事可以概括为：**
- 在当前 Hook 的更新队列里插入一个 Update；
- 把这次更新的「存在」和「优先级」通知给根节点；
- 触发一次新的渲染调度流程。

---

### 4. 调度入口：`scheduleUpdateOnFiber`

`dispatchSetState` 触发完本地的入队逻辑以后，会把控制权交给 `scheduleUpdateOnFiber`：

```typescript
export function scheduleUpdateOnFiber( //调度更新函数，判断是否需要调度构建fiber
  fiber: Fiber,
  lane: Lane,
  eventTime: number,
): FiberRoot | null {
  checkForNestedUpdates();// 检查是否有死循环，render中调用更新
  if (__DEV__) {
    if (isRunningInsertionEffect) {
      console.error('useInsertionEffect must not schedule updates.');
    }
  }
  //收集需要更新的子节点lane，放在父节点上，直至根节点
  const root = markUpdateLaneFromFiberToRoot(fiber, lane);
  if (root === null) {
    return null;
  }
  if (__DEV__) {
    if (isFlushingPassiveEffects) {
      didScheduleUpdateDuringPassiveEffects = true;
    }
  }
  // 把当前更新的lane放在hostroot上的pendingLanes上，然后将事件触发时间记录在eventtimes属性上
  markRootUpdated(root, lane, eventTime);
  if (
    (executionContext & RenderContext) !== NoLanes &&
    root === workInProgressRoot //正在运行的根节点fiber
  ) {
    // 命中render阶段触发setState
    // 通常我们不能在组件的render阶段去调用setState，有些内部特性（比如选择性 hydration）会把 ‘render 阶段更新
    warnAboutRenderPhaseUpdatesInDEV(fiber);
    // Track lanes that were updated during the render phase
    workInProgressRootRenderPhaseUpdatedLanes = mergeLanes(
      workInProgressRootRenderPhaseUpdatedLanes,
      lane,
    );
  } else {
    // 正常触发的更新。比如事件触发的。
    if (enableUpdaterTracking) {
      if (isDevToolsPresent) {
        addFiberToLanesMap(root, fiber, lane);
      }
    }
    warnIfUpdatesNotWrappedWithActDEV(fiber);
    if (enableProfilerTimer && enableProfilerNestedUpdateScheduledHook) {
      if (
        (executionContext & CommitContext) !== NoContext &&
        root === rootCommittingMutationOrLayoutEffects
      ) {
        if (fiber.mode & ProfileMode) {
          let current = fiber;
          while (current !== null) {
            if (current.tag === Profiler) {
              const {id, onNestedUpdateScheduled} = current.memoizedProps;
              if (typeof onNestedUpdateScheduled === 'function') {
                onNestedUpdateScheduled(id);
              }
            }
            current = current.return;
          }
        }
      }
    }
    if (enableTransitionTracing) {
      const transition = ReactCurrentBatchConfig.transition;
      if (transition !== null) {
        if (transition.startTime === -1) {
          transition.startTime = now();
        }
        addTransitionToLanesMap(root, transition, lane);
      }
    }
    if (root === workInProgressRoot) {
      // TODO: Consolidate with `isInterleavedUpdate` check
      // Received an update to a tree that's in the middle of rendering. Mark
      // that there was an interleaved update work on this root. Unless the
      // `deferRenderPhaseUpdateToNextBatch` flag is off and this is a render
      // phase update. In that case, we don't treat render phase updates as if
      // they were interleaved, for backwards compat reasons.
      if (
        deferRenderPhaseUpdateToNextBatch ||
        (executionContext & RenderContext) === NoContext
      ) {
        workInProgressRootInterleavedUpdatedLanes = mergeLanes(
          workInProgressRootInterleavedUpdatedLanes,
          lane,
        );
      }
      if (workInProgressRootExitStatus === RootSuspendedWithDelay) {
        // The root already suspended with a delay, which means this render
        // definitely won't finish. Since we have a new update, let's mark it as
        // suspended now, right before marking the incoming update. This has the
        // effect of interrupting the current render and switching to the update.
        // TODO: Make sure this doesn't override pings that happen while we've
        // already started rendering.
        markRootSuspended(root, workInProgressRootRenderLanes);
      }
    }
    ensureRootIsScheduled(root, eventTime); //调度创建fiber
    if (
      lane === SyncLane &&
      executionContext === NoContext &&
      (fiber.mode & ConcurrentMode) === NoMode &&
      // Treat `act` as if it's inside `batchedUpdates`, even in legacy mode.
      !(__DEV__ && ReactCurrentActQueue.isBatchingLegacy)
    ) {
      // Flush the synchronous work now, unless we're already working or inside
      // a batch. This is intentionally inside scheduleUpdateOnFiber instead of
      // scheduleCallbackForFiber to preserve the ability to schedule a callback
      // without immediately flushing it. We only do this for user-initiated
      // updates, to preserve historical behavior of legacy mode.
      resetRenderTimer();
      flushSyncCallbacksOnlyInLegacyMode();// 同步执行更新
    }
  }
  return root;
}

function markUpdateLaneFromFiberToRoot( //收集更新以及子节点的更新
  sourceFiber: Fiber,
  lane: Lane,
): FiberRoot | null {
  // Update the source fiber's lanes
  // 更新当前lane
  sourceFiber.lanes = mergeLanes(sourceFiber.lanes, lane);
  let alternate = sourceFiber.alternate;
  if (alternate !== null) {
    alternate.lanes = mergeLanes(alternate.lanes, lane);
  }
  if (__DEV__) {
    if (
      alternate === null &&
      (sourceFiber.flags & (Placement | Hydrating)) !== NoFlags
    ) {
      warnAboutUpdateOnNotYetMountedFiberInDEV(sourceFiber);
    }
  }
  // Walk the parent path to the root and update the child lanes.
  //一个向上收集的过程
  let node = sourceFiber;
  let parent = sourceFiber.return;
  while (parent !== null) {
    parent.childLanes = mergeLanes(parent.childLanes, lane);
    alternate = parent.alternate;
    if (alternate !== null) {
      alternate.childLanes = mergeLanes(alternate.childLanes, lane);
    } else {
      if (__DEV__) {
        if ((parent.flags & (Placement | Hydrating)) !== NoFlags) {
          warnAboutUpdateOnNotYetMountedFiberInDEV(sourceFiber);
        }
      }
    }
    node = parent;
    parent = parent.return;
  }
  if (node.tag === HostRoot) {
    const root: FiberRoot = node.stateNode;
    return root;
  } else {
    return null;
  }
}

function markRootUpdated(
  root: FiberRoot,
  updateLane: Lane,
  eventTime: number,
) {
  root.pendingLanes |= updateLane;

  // If there are any suspended transitions, it's possible this new update
  // could unblock them. Clear the suspended lanes so that we can try rendering
  // them again.
  //
  // TODO: We really only need to unsuspend only lanes that are in the
  // `subtreeLanes` of the updated fiber, or the update lanes of the return
  // path. This would exclude suspended updates in an unrelated sibling tree,
  // since there's no way for this update to unblock it.
  //
  // We don't do this if the incoming update is idle, because we never process
  // idle updates until after all the regular updates have finished; there's no
  // way it could unblock a transition.
  if (updateLane !== IdleLane) {
    root.suspendedLanes = NoLanes;
    root.pingedLanes = NoLanes;
  }

  const eventTimes = root.eventTimes;
  const index = laneToIndex(updateLane);
  // We can always overwrite an existing timestamp because we prefer the most
  // recent event, and we assume time is monotonically increasing.
  eventTimes[index] = eventTime;
}

function ensureRootIsScheduled(root: FiberRoot, currentTime: number) { //调度创建fiber函数,主要通过schedulecallback调度执行performConcurrentWorkperOnRoot
  const existingCallbackNode = root.callbackNode;

  // Check if any lanes are being starved by other work. If so, mark them as
  // expired so we know to work on those next.
  // 为当前任务根据优先级添加过期时间， 检查未执行任务中是否有过期任务，有的话就添加到expiedLanes中，在后续以同步方式执行，避免饿死
  markStarvedLanesAsExpired(root, currentTime);

  // Determine the next lanes to work on, and their priority.
  //获取优先级
  const nextLanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );
  // 如果nextLanes为空没任务执行了就中断
  if (nextLanes === NoLanes) {
    // Special case: There's nothing to work on.
    if (existingCallbackNode !== null) {
      cancelCallback(existingCallbackNode);
    }
    root.callbackNode = null;
    root.callbackPriority = NoLane;
    return;
  }

  // We use the highest priority lane to represent the priority of the callback.
  const newCallbackPriority = getHighestPriorityLane(nextLanes);

  // Check if there's an existing task. We may be able to reuse it.
  const existingCallbackPriority = root.callbackPriority;
  if (
    // 连续的两次setstate的话就会判断他们优先级是否相同，相同则不注册新的回调
    existingCallbackPriority === newCallbackPriority &&
    // Special case related to `act`. If the currently scheduled task is a
    // Scheduler task, rather than an `act` task, cancel it and re-scheduled
    // on the `act` queue.
    !(
      __DEV__ &&
      ReactCurrentActQueue.current !== null &&
      existingCallbackNode !== fakeActCallbackNode
    )
  ) {
    if (__DEV__) {
      // If we're going to re-use an existing task, it needs to exist.
      // Assume that discrete update microtasks are non-cancellable and null.
      // TODO: Temporary until we confirm this warning is not fired.
      if (
        existingCallbackNode == null &&
        existingCallbackPriority !== SyncLane
      ) {
        console.error(
          'Expected scheduled callback to exist. This error is likely caused by a bug in React. Please file an issue.',
        );
      }
    }
    // The priority hasn't changed. We can reuse the existing task. Exit.
    return;
  }
  ...
}
```

**这一段是将“某个组件上的更新”提升为“整个根上的一次调度”的关键：**
- 通过 `markUpdateLaneFromFiberToRoot` 自下而上把当前 `lane` 冒泡到根 `FiberRoot`，记录到 `pendingLanes`；  
- 通过 `markRootUpdated` 把根标记为「有这个优先级的更新待处理」，并记录事件时间；  
- 最后调用 `ensureRootIsScheduled(root, eventTime)` 确保对这个根安排一次「工作任务」。

对于同步优先级（`SyncLane`）并且当前不在任何渲染 / 提交上下文里，还会立即 `flushSyncCallbacksOnlyInLegacyMode`，直接在当前事件循环内完成整次更新。

> 辅助函数说明：  
> - `markUpdateLaneFromFiberToRoot`：从当前 fiber 一直向上，把此 lane 合并到每一层 fiber 的 `lanes` 字段，并最终返回根 `FiberRoot`；  
> - `markRootUpdated`：在根上记录本次更新的 lanes 和 eventTime，作为后续调度 & 过期检查依据；  
> - `ensureRootIsScheduled`：根据根上所有 `pendingLanes` 算出本轮要处理的优先级，并通过 Scheduler / 微任务队列安排真正的 render 入口函数。

---

### 5. 渲染入口：`performSyncWorkOnRoot` / `renderRootSync`

对于一次同步的 `setState`，最终会走到 `performSyncWorkOnRoot` 这个根级别的同步工作入口：

```typescript
// This is the entry point for synchronous tasks that don't go
// through Scheduler
function performSyncWorkOnRoot(root) {
  if (enableProfilerTimer && enableProfilerNestedUpdatePhase) {
    syncNestedUpdateFlag();
  }
  if ((executionContext & (RenderContext | CommitContext)) !== NoContext) {
    throw new Error('Should not already be working.');
  }
  flushPassiveEffects();
  let lanes = getNextLanes(root, NoLanes);
  if (!includesSomeLane(lanes, SyncLane)) {
    // There's no remaining sync work left.
    ensureRootIsScheduled(root, now());
    return null;
  }
  let exitStatus = renderRootSync(root, lanes);
  if (root.tag !== LegacyRoot && exitStatus === RootErrored) {
    // If something threw an error, try rendering one more time. We'll render
    // synchronously to block concurrent data mutations, and we'll includes
    // all pending updates are included. If it still fails after the second
    // attempt, we'll give up and commit the resulting tree.
    const errorRetryLanes = getLanesToRetrySynchronouslyOnError(root);
    if (errorRetryLanes !== NoLanes) {
      lanes = errorRetryLanes;
      exitStatus = recoverFromConcurrentError(root, errorRetryLanes);
    }
  }
  if (exitStatus === RootFatalErrored) {
    const fatalError = workInProgressRootFatalError;
    prepareFreshStack(root, NoLanes);
    markRootSuspended(root, lanes);
    ensureRootIsScheduled(root, now());
    throw fatalError;
  }
  if (exitStatus === RootDidNotComplete) {
    throw new Error('Root did not complete. This is a bug in React.');
  }
  // We now have a consistent tree. Because this is a sync render, we
  // will commit it even if something suspended.
  const finishedWork: Fiber = (root.current.alternate: any);
  root.finishedWork = finishedWork;
  root.finishedLanes = lanes;
  commitRoot(root, workInProgressRootRecoverableErrors);
  // Before exiting, make sure there's a callback scheduled for the next
  // pending level.
  ensureRootIsScheduled(root, now());

  return null;
}
```

**同步渲染流程可以拆成三大步：**

1. **准备阶段：**
   - `flushPassiveEffects()`：先把上一次渲染遗留的 effect 清空；  
   - 通过 `getNextLanes(root, NoLanes)` 选择这次要处理的 lanes，并确保其中包含 `SyncLane`。

2. **渲染阶段（render phase）：**
   - 调用 `renderRootSync(root, lanes)`，这一步会构建整棵新的 Fiber 树（`workInProgress` 树）。

3. **完成 & 提交阶段：**
   - 渲染成功后，从 `root.current.alternate` 拿到 `finishedWork`（即刚构造好的 `workInProgress` 根）；  
   - 将 `root.finishedWork` 和 `root.finishedLanes` 赋值；  
   - 调用 **`commitRoot(root, ...)` 进入 commit 阶段，把变更真正应用到宿主环境（如 DOM）。  

#### 5.1 `renderRootSync` 内部：如何跑完整个 Fiber 树？

`renderRootSync` 里最重要的一段就是内部调用 `workLoopSync` 的这一部分，完整源码如下：

```typescript
function renderRootSync(root: FiberRoot, lanes: Lanes) {
  const prevExecutionContext = executionContext;
  executionContext |= RenderContext;
  const prevDispatcher = pushDispatcher();
  // If the root or lanes have changed, throw out the existing stack
  // and prepare a fresh one. Otherwise we'll continue where we left off.
  if (workInProgressRoot !== root || workInProgressRootRenderLanes !== lanes) {
    if (enableUpdaterTracking) {
      if (isDevToolsPresent) {
        const memoizedUpdaters = root.memoizedUpdaters;
        if (memoizedUpdaters.size > 0) {
          restorePendingUpdaters(root, workInProgressRootRenderLanes);
          memoizedUpdaters.clear();
        }
        // At this point, move Fibers that scheduled the upcoming work from the Map to the Set.
        // If we bailout on this work, we'll move them back (like above).
        // It's important to move them now in case the work spawns more work at the same priority with different updaters.
        // That way we can keep the current update and future updates separate.
        movePendingFibersToMemoized(root, lanes);
      }
    }
    workInProgressTransitions = getTransitionsForLanes(root, lanes);
    prepareFreshStack(root, lanes);
  }
  if (__DEV__) {
    if (enableDebugTracing) {
      logRenderStarted(lanes);
    }
  }
  if (enableSchedulingProfiler) {
    markRenderStarted(lanes);
  }
  do {
    try {
      workLoopSync();
      break;
    } catch (thrownValue) {
      handleError(root, thrownValue);
    }
  } while (true);
  resetContextDependencies();
  executionContext = prevExecutionContext;
  popDispatcher(prevDispatcher);
  if (workInProgress !== null) {
    // This is a sync render, so we should have finished the whole tree.
    throw new Error(
      'Cannot commit an incomplete root. This error is likely caused by a ' +
        'bug in React. Please file an issue.',
    );
  }
  if (__DEV__) {
    if (enableDebugTracing) {
      logRenderStopped();
    }
  }

  if (enableSchedulingProfiler) {
    markRenderStopped();
  }

  // Set this to null to indicate there's no in-progress render.
  workInProgressRoot = null;
  workInProgressRootRenderLanes = NoLanes;

  return workInProgressRootExitStatus;
}

// The work loop is an extremely hot path. Tell Closure not to inline it.
/** @noinline */
function workLoopSync() {
  // Already timed out, so perform work without checking if we need to yield.
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}
```

可以看到，**整个 render 阶段其实就是一个 while 循环，不断调用 `performUnitOfWork`，直到 `workInProgress` 为空为止。**

---

### 6. 单个 Fiber 的执行：`performUnitOfWork`、`beginWork`、`completeUnitOfWork`

#### 6.1 `performUnitOfWork`：一个节点的完整「工作单元」

```typescript
function performUnitOfWork(unitOfWork: Fiber): void {
  const current = unitOfWork.alternate;
  setCurrentDebugFiberInDEV(unitOfWork);

  let next;// 用于移动workInProgress指针
  if (enableProfilerTimer && (unitOfWork.mode & ProfileMode) !== NoMode) {
    startProfilerTimer(unitOfWork);
    next = beginWork(current, unitOfWork, subtreeRenderLanes);
    stopProfilerTimerIfRunningAndRecordDelta(unitOfWork, true);
  } else {
    next = beginWork(current, unitOfWork, subtreeRenderLanes);
  }

  resetCurrentDebugFiberInDEV();
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }

  ReactCurrentOwner.current = null;
}
```

这里可以理解为「处理一个 Fiber 节点的完整生命周期」：
- 先调用 `beginWork(current, unitOfWork, subtreeRenderLanes)` 向下「打开」这个节点；  
- 如果 `beginWork` 返回了一个子节点 `next`，说明还有子树要继续向下走，于是把 `workInProgress` 指向这个子节点；  
- 如果返回 `null`，说明当前节点已经是叶子节点或子树已经处理完，接下来进入 **`completeUnitOfWork`** 向上回溯。

#### 6.2 `beginWork`：根据 Tag 创建 / 更新当前节点

`beginWork` 是 render 阶段「向下」的主函数：

```typescript
function beginWork( //performUnitOfWork的向下执行的函数，主要是创建或者更新当前节点属性返回下一节点
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  if (__DEV__) {
    if (workInProgress._debugNeedsRemount && current !== null) {
      // This will restart the begin phase with a new fiber.
      return remountFiber(
        current,
        workInProgress,
        createFiberFromTypeAndProps(
          workInProgress.type,
          workInProgress.key,
          workInProgress.pendingProps,
          workInProgress._debugOwner || null,
          workInProgress.mode,
          workInProgress.lanes,
        ),
      );
    }
  }

  if (current !== null) { // 初次渲染时候current为空
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;

    if (
      oldProps !== newProps ||
      hasLegacyContextChanged() ||
      // Force a re-render if the implementation changed due to hot reload:
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      // If props or context changed, mark the fiber as having performed work.
      // This may be unset if the props are determined to be equal later (memo).
      didReceiveUpdate = true;
    } else {
      // Neither props nor legacy context changes. Check if there's a pending
      // update or context change.
      const hasScheduledUpdateOrContext = checkScheduledUpdateOrContext(
        current,
        renderLanes,
      );
      if (
        !hasScheduledUpdateOrContext &&
        // If this is the second pass of an error or suspense boundary, there
        // may not be work scheduled on `current`, so we check for this flag.
        (workInProgress.flags & DidCapture) === NoFlags
      ) {
        // No pending updates or context. Bail out now.
        didReceiveUpdate = false;
        return attemptEarlyBailoutIfNoScheduledUpdate( // 复用之前的节点
          current,
          workInProgress,
          renderLanes,
        );
      }
      if ((current.flags & ForceUpdateForLegacySuspense) !== NoFlags) {
        // This is a special case that only exists for legacy mode.
        // See https://github.com/facebook/react/pull/19216.
        didReceiveUpdate = true;
      } else {
        // An update was scheduled on this fiber, but there are no new props
        // nor legacy context. Set this to false. If an update queue or context
        // consumer produces a changed value, it will set this to true. Otherwise,
        // the component will assume the children have not changed and bail out.
        didReceiveUpdate = false;
      }
    }
  } else {
    didReceiveUpdate = false;

    if (getIsHydrating() && isForkedChild(workInProgress)) {
      // Check if this child belongs to a list of muliple children in
      // its parent.
      //
      // In a true multi-threaded implementation, we would render children on
      // parallel threads. This would represent the beginning of a new render
      // thread for this subtree.
      //
      // We only use this for id generation during hydration, which is why the
      // logic is located in this special branch.
      const slotIndex = workInProgress.index;
      const numberOfForks = getForksAtLevel(workInProgress);
      pushTreeId(workInProgress, numberOfForks, slotIndex);
    }
  }

  // Before entering the begin phase, clear pending update priority.
  // TODO: This assumes that we're about to evaluate the component and process
  // the update queue. However, there's an exception: SimpleMemoComponent
  // sometimes bails out later in the begin phase. This indicates that we should
  // move this assignment out of the common path and into each branch.
  workInProgress.lanes = NoLanes;

  switch (workInProgress.tag) {
    case IndeterminateComponent: {
      return mountIndeterminateComponent(
        current,
        workInProgress,
        workInProgress.type,
        renderLanes,
      );
    }
    case LazyComponent: {
      const elementType = workInProgress.elementType;
      return mountLazyComponent(
        current,
        workInProgress,
        elementType,
        renderLanes,
      );
    }
    case FunctionComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateFunctionComponent( //函数组件
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes,
      );
    }
    // 下面还有 ClassComponent / HostRoot / HostComponent / HostText 等分支，这里略去不再重复贴出。
  }
}
```

对于我们关注的函数组件：
- `tag === FunctionComponent` 分支会调用 `updateFunctionComponent`，内部会执行组件函数体，并在执行过程中驱动前面提到的 Hook 系统（`mountState/updateState` 等）；  
- 在这个过程中，会读取每个 Hook 队列里的 Update，合并成新的 state，决定本组件是否需要更新，并构造新的子节点 Fiber。

> 其他 Tag（ClassComponent / HostComponent / Suspense 等）都是同一套模式：根据 Tag 调用对应的 updateXXX 函数，决定如何构建子树。

#### 6.3 `completeUnitOfWork` + `completeWork`：向上归并 & 标记副作用

当某个 Fiber 子树已经「向下」走完（即 `beginWork` 返回 `null`），就会进入 `completeUnitOfWork`，自下而上地做收尾工作并构建 effect 链：

```typescript
function completeUnitOfWork(unitOfWork: Fiber): void {
  // Attempt to complete the current unit of work, then move to the next
  // sibling. If there are no more siblings, return to the parent fiber.
  let completedWork = unitOfWork;
  do {
    // The current, flushed, state of this fiber is the alternate. Ideally
    // nothing should rely on this, but relying on it here means that we don't
    // need an additional field on the work in progress.
    const current = completedWork.alternate;
    const returnFiber = completedWork.return;

    // Check if the work completed or if something threw.
    if ((completedWork.flags & Incomplete) === NoFlags) {
      setCurrentDebugFiberInDEV(completedWork);
      let next;
      if (
        !enableProfilerTimer ||
        (completedWork.mode & ProfileMode) === NoMode
      ) {
        next = completeWork(current, completedWork, subtreeRenderLanes);
      } else {
        startProfilerTimer(completedWork);
        next = completeWork(current, completedWork, subtreeRenderLanes);
        // Update render duration assuming we didn't error.
        stopProfilerTimerIfRunningAndRecordDelta(completedWork, false);
      }
      resetCurrentDebugFiberInDEV();

      if (next !== null) {
        // Completing this fiber spawned new work. Work on that next.
        workInProgress = next;
        return;
      }
    } else {
      ...
    }

    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      // If there is more work to do in this returnFiber, do that next.
      workInProgress = siblingFiber;
      return;
    }
    // Otherwise, return to the parent
    completedWork = returnFiber;
    // Update the next thing we're working on in case something throws.
...
```

真正完成每个节点「收尾工作」的是 `ReactFiberCompleteWork.new.js` 里的 `completeWork`：

```typescript
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const newProps = workInProgress.pendingProps;
  // Note: This intentionally doesn't check if we're hydrating because comparing
  // to the current tree provider fiber is just as fast and less error-prone.
  // Ideally we would have a special version of the work loop only
  // for hydration.
  popTreeContext(workInProgress);
  switch (workInProgress.tag) {
    case IndeterminateComponent:
    case LazyComponent:
    case SimpleMemoComponent:
    case FunctionComponent:
    case ForwardRef:
    case Fragment:
    case Mode:
    case Profiler:
    case ContextConsumer:
    case MemoComponent:
      bubbleProperties(workInProgress);
      return null;
    case ClassComponent: {
      const Component = workInProgress.type;
      if (isLegacyContextProvider(Component)) {
        popLegacyContext(workInProgress);
      }
      bubbleProperties(workInProgress);
      return null;
    }
    case HostRoot: {
      const fiberRoot = (workInProgress.stateNode: FiberRoot);
      ...
      updateHostContainer(current, workInProgress);
      bubbleProperties(workInProgress);
      ...
      return null;
    }
    case HostComponent: {
      popHostContext(workInProgress);
      const rootContainerInstance = getRootHostContainer();
      const type = workInProgress.type;
      if (current !== null && workInProgress.stateNode != null) {
        updateHostComponent(
          current,
          workInProgress,
          type,
          newProps,
          rootContainerInstance,
        );

        if (current.ref !== workInProgress.ref) {
          markRef(workInProgress);
        }
      } else {
        ...
        const instance = createInstance(
          type,
          newProps,
          rootContainerInstance,
          currentHostContext,
          workInProgress,
        );

        appendAllChildren(instance, workInProgress, false, false);

        workInProgress.stateNode = instance;
        ...
      }
      bubbleProperties(workInProgress);
      return null;
    }
    case HostText: {
      const newText = newProps;
      if (current && workInProgress.stateNode != null) {
        const oldText = current.memoizedProps;
        // If we have an alternate, that means this is an update and we need
        // to schedule a side-effect to do the updates.
        updateHostText(current, workInProgress, oldText, newText);
      } else {
        ...
        const wasHydrated = popHydrationState(workInProgress);
        if (wasHydrated) {
          if (prepareToHydrateHostTextInstance(workInProgress)) {
            markUpdate(workInProgress);
          }
        } else {
          workInProgress.stateNode = createTextInstance(
            newText,
            rootContainerInstance,
            currentHostContext,
            workInProgress,
          );
        }
      }
      bubbleProperties(workInProgress);
      return null;
    }
    ...
}
```

在这里，React 会：
- 为 HostComponent / HostText 等宿主节点创建或更新对应的宿主实例（比如 DOM 节点）；  
- 根据新旧 props 的差异，通过 `markUpdate` / `updateHostComponent` 等方式在 Fiber 上打上「需要在 commit 阶段执行 DOM 更新」的 flag；  
- 通过 `bubbleProperties` 向父节点汇总 subtree 的 flags（例如 Placement / Update / Deletion 等）。

**到此为止，整个 render 阶段完成：**
- 新的 Fiber 树已经构造好；  
- 哪些地方需要新增 / 更新 / 删除 / 调整属性 / 触发 ref / 执行生命周期 / effect，都已经标记在 Fiber 的 flags 和 subtreeFlags 上；  
- 接下来就进入 **commit 阶段**，真正「动手」修改 DOM 和触发副作用。

---

### 7. 提交阶段：`commitRoot` -> `commitRootImpl`

渲染结束后，`performSyncWorkOnRoot` 会调用 `commitRoot`：

```typescript
function commitRoot(root: FiberRoot, recoverableErrors: null | Array<mixed>) {// 提交阶段
  // TODO: This no longer makes any sense. We already wrap the mutation and
  // layout phases. Should be able to remove.
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

  return null;
}
```

真正的提交逻辑在 `commitRootImpl` 中：

```typescript
function commitRootImpl(// commit执行函数
  root: FiberRoot,
  recoverableErrors: null | Array<mixed>,
  renderPriorityLevel: EventPriority,
) {
  do {
    // `flushPassiveEffects` will call `flushSyncUpdateQueue` at the end, which
    // means `flushPassiveEffects` will sometimes result in additional
    // passive effects. So we need to keep flushing in a loop until there are
    // no more pending effects.
    // TODO: Might be better if `flushPassiveEffects` did not automatically
    // flush synchronous work at the end, to avoid factoring hazards like this.
    flushPassiveEffects();
  } while (rootWithPendingPassiveEffects !== null);
  flushRenderPhaseStrictModeWarningsInDEV();

  if ((executionContext & (RenderContext | CommitContext)) !== NoContext) {
    throw new Error('Should not already be working.');
  }

  const finishedWork = root.finishedWork;
  const lanes = root.finishedLanes;
  ...
  root.finishedWork = null;
  root.finishedLanes = NoLanes;
  ...
  // commitRoot never returns a continuation; it always finishes synchronously.
  // So we can clear these now to allow a new callback to be scheduled.
  root.callbackNode = null;
  root.callbackPriority = NoLane;
  ...
  // If there are pending passive effects, schedule a callback to process them.
  // Do this as early as possible, so it is queued before anything else that
  // might get scheduled in the commit phase. (See #16714.)
  // TODO: Delete all other places that schedule the passive effect callback
  // They're redundant.
  if (
    (finishedWork.subtreeFlags & PassiveMask) !== NoFlags ||
    (finishedWork.flags & PassiveMask) !== NoFlags
  ) {
    if (!rootDoesHavePassiveEffects) {
      rootDoesHavePassiveEffects = true;
      pendingPassiveEffectsRemainingLanes = remainingLanes;
      //调度执行useeffect
      scheduleCallback(NormalSchedulerPriority, () => {
        flushPassiveEffects();
        // This render triggered passive effects: release the root cache pool
        // *after* passive effects fire to avoid freeing a cache pool that may
        // be referenced by a node in the tree (HostRoot, Cache boundary etc)
        return null;
      });
    }
  }

  // Check if there are any effects in the whole tree.
  ...
  const subtreeHasEffects =
    (finishedWork.subtreeFlags &
      (BeforeMutationMask | MutationMask | LayoutMask | PassiveMask)) !==
    NoFlags;
  const rootHasEffect =
    (finishedWork.flags &
      (BeforeMutationMask | MutationMask | LayoutMask | PassiveMask)) !==
    NoFlags;

  if (subtreeHasEffects || rootHasEffect) {
    const prevTransition = ReactCurrentBatchConfig.transition;
    ReactCurrentBatchConfig.transition = null;
    const previousPriority = getCurrentUpdatePriority();
    setCurrentUpdatePriority(DiscreteEventPriority);

    const prevExecutionContext = executionContext;
    executionContext |= CommitContext;

    // Reset this to null before calling lifecycles
    ReactCurrentOwner.current = null;

    // The commit phase is broken into several sub-phases. We do a separate pass
    // of the effect list for each phase: all mutation effects come before all
    // layout effects, and so on.

    // The first phase a "before mutation" phase. We use this phase to read the
    // state of the host tree right before we mutate it. This is where
    // getSnapshotBeforeUpdate is called.
    // dom改变之前
    const shouldFireAfterActiveInstanceBlur = commitBeforeMutationEffects(
      root,
      finishedWork,
    );
...
```

**整体来说，commit 阶段分为几个子阶段：**

1. **处理 passive effects（useEffect）调度：**  
   - 在正式 commit 之前先清空上一次渲染遗留的 passive effects；  
   - 如果这次渲染生成了新的 Passive flag，则通过 `scheduleCallback` 在后续任务中执行 `flushPassiveEffects`。

2. **三大同步提交子阶段：**
   - **`commitBeforeMutationEffects`**：在真正修改 DOM 之前执行的阶段，典型场景是 `getSnapshotBeforeUpdate`；  
   - **Mutation 阶段（`commitMutationEffects` 等）**：根据每个 Fiber 上的 Placement / Update / Deletion 等 flag，对真实 DOM 树做插入 / 删除 / 属性更新等操作；  
   - **Layout 阶段（`commitLayoutEffects` 等）**：在 DOM 更新完成后，执行 class 组件的 `componentDidMount/Update`、函数组件的 `useLayoutEffect` 回调等。

3. **最后：切换根节点的 current 指针：**  
   - 在 Mutation 阶段结束时，`root.current` 会被指向这次渲染得到的 `finishedWork` 树，从此这棵 Fiber 树就变成了新的「当前界面」。

> 上述具体子函数如 `commitBeforeMutationEffects` / `commitMutationEffects` / `commitLayoutEffects` 源码体量较大，但它们的共同目标都是：  
> **根据 render 阶段打在 Fiber 上的 flags，按正确的时序调用宿主环境 API（如 DOM 操作）和用户定义的副作用（生命周期 / effect 等）。**

---

### 8. 从一次 `setState` 到界面更新：完整调用链串联

基于上面的源码分析，可以把「一次 `setState` 更新」的全链路串起来：

1. **组件内部调用：**
   - 业务组件（例如 `AppTest`）在事件回调里调用 `setState`；  
   - 这个 `setState` 实际是 `dispatchSetState.bind(fiber, queue)`。

2. **Hook 层处理：**
   - `dispatchSetState` 创建 `update`，写入 Hook 的 `UpdateQueue`；  
   - 计算更新优先级 `lane`，并调用 `scheduleUpdateOnFiber(fiber, lane, eventTime)`。

3. **调度到根：**
   - `scheduleUpdateOnFiber` 通过 `markUpdateLaneFromFiberToRoot` 把 `lane` 向上标记到根 FiberRoot；  
   - 调用 `markRootUpdated` 记录这次更新；  
   - 通过 `ensureRootIsScheduled` 为根安排一次同步或并发渲染任务（同步时最终进入 `performSyncWorkOnRoot`）。

4. **render 阶段（构建 workInProgress 树）：**
   - `performSyncWorkOnRoot` 调用 `renderRootSync`；  
   - `renderRootSync` 里循环执行 `workLoopSync`；  
   - `workLoopSync` 不断 `performUnitOfWork(workInProgress)`，对每个 Fiber 节点先 `beginWork` 再 `completeUnitOfWork`；  
   - 函数组件在 `beginWork` 的 `FunctionComponent` 分支里执行函数体，期间跑 Hook 链表，消费 `UpdateQueue` 更新 state，产出新的子树结构。

5. **commit 阶段（真正修改 DOM / 触发副作用）：**
   - render 完成后，`performSyncWorkOnRoot` 设置 `root.finishedWork` 等字段，并调用 `commitRoot`；  
   - `commitRoot` -> `commitRootImpl`，先清理 / 调度 passive effects，然后按顺序执行：  
     - `commitBeforeMutationEffects`：DOM 变更前的快照等；  
     - Mutation 阶段（宿主环境实际更新，如 DOM 操作）；  
     - Layout 阶段（`useLayoutEffect`、class component 的 didMount/didUpdate 等）；  
   - 异步的 `useEffect` 通过 `Passive` flag 和 `scheduleCallback` 被延后执行。

**最终效果：**
- **`setState` 不会立刻修改 DOM，而是通过一套完整的 Fiber 调度 + 渲染 + 提交流程，把这次更新整合到一个批次里处理；**  
- **render 阶段负责计算「应该长成什么样」和「有哪些变化」；**  
- **commit 阶段负责「真正应用变化」和「触发副作用」。**

---

### 9. 结合 Diff 视角再看：列表 / 条件渲染中的 setState

你在项目中还专门写了一个关于 Diff 的分析文档 `react-diff-analysis.md`，其中对 `reconcileChildrenArray`、`updateSlot`、`placeChild`、`mapRemainingChildren` 等做了细致中文注释，这些都是 **`beginWork` 在处理子节点时调用的 diff 算法**。例如：

### 1. `reconcileChildrenArray` - 数组子节点diff

这是React diff算法的核心函数，位于`ReactChildFiber.new.js`中：

<details>
<summary><strong>🔍 点击查看完整源码实现</strong></summary>

```javascript
function reconcileChildrenArray(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChildren: Array<*>,
  lanes: Lanes,
): Fiber | null {
  // 第一轮遍历：处理相同位置的节点
  let oldFiber = currentFirstChild;
  let newIdx = 0;
  let lastPlacedIndex = 0;
  ...
}
```
...
```

结合本文主线可以理解为：
- 当 setState 导致某个函数组件重新渲染时，在 `beginWork` 的 `FunctionComponent` 分支中，会重新生成该组件的 `children`；  
- 如果 children 是数组（列表渲染），则会调用 `reconcileChildrenArray`，利用 key + 三轮遍历策略（同位置比较 / 处理剩余节点 / map + 移动/删除）计算出这次更新应该对哪些子 Fiber 执行插入、移动、删除；  
- 这些 diff 结果最终体现在子 Fiber 的 flags（如 Placement / Deletion / Update）上，并在 commit 阶段被真正落实到 DOM。

**因此：**
- 本文从「一次 `setState`」的角度梳理的是「从 Hook 到 Fiber 调度再到 commit」的大流程；  
- `react-diff-analysis.md` 里详细分析的 diff 算法，正是嵌套在 `beginWork` -> 子节点协调 这一环节，在「这次更新涉及哪些具体 DOM 变更」的问题上扮演关键角色。

通过这两部分结合，你已经掌握了 React 内部从「一次 setState」到「页面完成更新」的完整执行机制：  
**从事件 -> Hook -> UpdateQueue -> 调度 -> Fiber 渲染 -> Diff -> Commit -> DOM & 副作用。**


