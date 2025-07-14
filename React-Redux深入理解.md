# React-Redux深入理解

### 概述

React-Redux是我们常用的react状态管理工具，所以理解其核心执行流程尤为重要，下面将走以下几个方面详细讲解： store是干什么？Provider组件的实现？store的改变是怎么触发状态更新的？为什么使用useSelector时候建议选择具体的状态？

### 基本的使用

```javascript
import React, { useEffect } from "react";
import { Provider, useSelector, useDispatch } from "react-redux";
import { configureStore, createSlice } from '@reduxjs/toolkit'

const App: React.FC<any> = () => {
    const app = useSelector((state: any) => state.app);
    const dispatch = useDispatch();
    return(
        <div>
            <div>app: {app}</div>
            <button onClick={() => dispatch(appStore.actions.change({data: 'appChange'}))}>change APP</button>
            <Name />
            <Age />
        </div>
    )
}

const Name = () => {
    const name = useSelector((state: any) => state.name);
    const dispatch = useDispatch();
    useEffect(() => {
        console.log(name, 'name');
    })
    return(
        <>
            <div>name: {name}</div>
            <button onClick={() => dispatch(nameStore.actions.change({data: 'nameChange'}))}>change NAME</button>
        </>
    )
}

const Age = () => {
    const age = useSelector((state: any) => state.age);
    const dispatch = useDispatch();
    return(
        <>
            <div>age: {age}</div>
            <button onClick={() => dispatch(ageStore.actions.change({data: 27}))}>change AGE</button>
        </>
    )
}

const nameStore = createSlice({
    name: 'name',
    initialState: 'gzq',
    reducers: {
        change(state, payload) {
           return payload.payload.data
        },
    }
});

const ageStore = createSlice({
    name: 'age',
    initialState: 22,
    reducers: {
        change(state, payload) {
           return payload.payload.data
        },
    }
});

const appStore = createSlice({
    name: 'app',
    initialState: 'app',
    reducers: {
        change(state, payload) {
           return payload.payload.data
        },
    }
});

const store = configureStore({
    reducer: {
        name: nameStore.reducer,
        age: ageStore.reducer,
        app: appStore.reducer
    }
})

const WrapApp = () => {
    return(
        <Provider store={store}>
            <App />
        </Provider>
    )
}

export default WrapApp
```

### 1. 创建的Store是个啥？

#### Store是redux里面的状态管理对象，主要有以下几个核心功能：

1. **获取当前状态的快照。** *store.getState()*
2. **订阅状态改变。** *store.subscribe(callback)*
3. **派发状态更新。 ** *store.dispatch(params: Action)*   **Action: {type: string, payload: any}**
4. **触发回调，当派发状态更新后会里面执行订阅的函数。**

```javascript
// Store的核心代码
export function createStore<
    S,
    A extends Action,
    Ext extends {} = {},
    StateExt extends {} = {},
    PreloadedState = S
>(
    reducer: Reducer<S, A, PreloadedState>,
    preloadedState?: PreloadedState | StoreEnhancer<Ext, StateExt> | undefined,
    enhancer?: StoreEnhancer<Ext, StateExt>
): Store<S, A, UnknownIfNonSpecific<StateExt>> & Ext {
    if (typeof reducer !== 'function') {
      // 对传入的reducer进行合法校验
    }
    if (
        (typeof preloadedState === 'function' && typeof enhancer === 'function') ||
        (typeof enhancer === 'function' && typeof arguments[3] === 'function')
    ) {
      // 初始状态相关校验
    }
    if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
        enhancer = preloadedState as StoreEnhancer<Ext, StateExt>
        preloadedState = undefined
    }
    if (typeof enhancer !== 'undefined') {
        if (typeof enhancer !== 'function') {
            // enhancer用于增强Store
        }
        return enhancer(createStore)(
            reducer,
            preloadedState as PreloadedState | undefined
        )
    }

    let currentReducer = reducer
    let currentState: S | PreloadedState | undefined = preloadedState as
        | PreloadedState
        | undefined
    let currentListeners: Map<number, ListenerCallback> | null = new Map()
    let nextListeners = currentListeners
    let listenerIdCounter = 0
    let isDispatching = false
		// 双缓冲作用，确保本次调用的监听不受影响
    // store.subscribe(() => {
  	// 	store.subscribe(anotherListener) // 可能导致问题
		// })
    function ensureCanMutateNextListeners() {
        if (nextListeners === currentListeners) {
            nextListeners = new Map()
            currentListeners.forEach((listener, key) => {
                nextListeners.set(key, listener)
            })
        }
    }
    // 获取当前状态的快照
    function getState(): S {
        if (isDispatching) {
            // 正在dispatch报错
        }
        return currentState as S
    }
		// 订阅事件，返回解绑方法
    function subscribe(listener: () => void) {
        if (isDispatching) {
					// 正在dispatch报错
        }
        let isSubscribed = true
        ensureCanMutateNextListeners()
        const listenerId = listenerIdCounter++
        nextListeners.set(listenerId, listener)
        return function unsubscribe() {
            if (!isSubscribed) {
                return
            }
            if (isDispatching) {
                // 正在dispatch报错
            }
            isSubscribed = false
            ensureCanMutateNextListeners()
            nextListeners.delete(listenerId)
            currentListeners = null
        }
    }
    // 触发状态更新并执行监听
    function dispatch(action: A) {
        try {
            isDispatching = true
          	// 更新状态
            currentState = currentReducer(currentState, action)
        } finally {
            isDispatching = false
        }
        const listeners = (currentListeners = nextListeners)
        // 执行监听
        listeners.forEach(listener => {
            listener()
        })
        return action
    }
		// 替换reducer
    function replaceReducer(nextReducer: Reducer<S, A>): void {
        currentReducer = nextReducer as unknown as Reducer<S, A, PreloadedState>
        dispatch({ type: ActionTypes.REPLACE } as A)
    }
		// 对store的增强，不怎么用到就不copy出来了
    function observable() {}
    dispatch({ type: ActionTypes.INIT } as A)
		// 这就是我们的store
    const store = {
        dispatch: dispatch as Dispatch<A>,
        subscribe,
        getState,
        replaceReducer,
        [$$observable]: observable
    } as unknown as Store<S, A, StateExt> & Ext
    return store
}
```

### 2. Provider的实现

#### Provider组件相对比较简单，以下是它的核心源码，主要流程如下：

1. **创建react context上下文，这个上下文主要包含redux的store以及react-redux的订阅对象subscription。**
2. **缓存store状态的快照。**
3. **Provider挂载，上下文和store状态快照改变时候，subscription订阅状态更新回调。当状态改变时候批量执行回调。**

```javascript
function Provider<A extends Action<string> = UnknownAction, S = unknown>(
  providerProps: ProviderProps<A, S>,
) {
  const { children, context, serverState, store } = providerProps
  const contextValue = React.useMemo(() => {
    const subscription = createSubscription(store)
    const baseContextValue = {
      store,
      subscription,
      getServerState: serverState ? () => serverState : undefined,
    }
    return baseContextValue
  }, [store, serverState])
  const previousState = React.useMemo(() => store.getState(), [store])
  useIsomorphicLayoutEffect(() => {
    const { subscription } = contextValue
    // 这里是核心之一订阅回调
    subscription.onStateChange = subscription.notifyNestedSubs
    subscription.trySubscribe()
    if (previousState !== store.getState()) {
      subscription.notifyNestedSubs()
    }
    return () => {
      subscription.tryUnsubscribe()
      subscription.onStateChange = undefined
    }
  }, [contextValue, previousState])
  const Context = context || ReactReduxContext
  return <Context.Provider value={contextValue}>{children}</Context.Provider>
}
```

### 3. subscription订阅对象的作用

#### subscription的本质其实也是一个发布订阅，不同在于它的订阅采用的是双向链表的数据结构，确保父组件在子组件之前更新。

在Provider组件里面执行**subscription.trySubscribe()**时候会往redux的store上注册一个回调**store.subscribe(handleChangeWrapper)**而这个回调函数正是**notifyNestedSubs**。它会去执行**listeners的notify**批量执行注册在**subscription**的回调函数。

结论：<u>**所以当store的状态更新后会触发subscription上所有的回调**</u>

```javascript
function createListenerCollection() {
    let first: Listener | null = null
    let last: Listener | null = null
    return {
        clear() {
            first = null
            last = null
        },
        notify() {
            batch(() => {
                let listener = first
                while (listener) {
                    listener.callback()
                    listener = listener.next
                }
            })
        },
        get() {
            const listeners: Listener[] = []
            let listener = first
            while (listener) {
                listeners.push(listener)
                listener = listener.next
            }
            return listeners
        },
        subscribe(callback: () => void) {
            let isSubscribed = true

            const listener: Listener = (last = {
                callback,
                next: null,
                prev: last,
            })
            if (listener.prev) {
                listener.prev.next = listener
            } else {
                first = listener
            }
            return function unsubscribe() {
                if (!isSubscribed || first === null) return
                isSubscribed = false

                if (listener.next) {
                    listener.next.prev = listener.prev
                } else {
                    last = listener.prev
                }
                if (listener.prev) {
                    listener.prev.next = listener.next
                } else {
                    first = listener.next
                }
            }
        },
    }
}

function createSubscription(store: any, parentSub?: Subscription) {
    let unsubscribe: VoidFunc | undefined
    let listeners: ListenerCollection = nullListeners
    let subscriptionsAmount = 0
    let selfSubscribed = false
    function addNestedSub(listener: () => void) {
        trySubscribe()
        const cleanupListener = listeners.subscribe(listener)
        let removed = false
        return () => {
            if (!removed) {
                removed = true
                cleanupListener()
                tryUnsubscribe()
            }
        }
    }
    function notifyNestedSubs() {
        listeners.notify()
    }
    function handleChangeWrapper() {
        if (subscription.onStateChange) {
            subscription.onStateChange()
        }
    }
    function isSubscribed() {
        return selfSubscribed
    }
    function trySubscribe() {
        subscriptionsAmount++
        if (!unsubscribe) {
            unsubscribe = parentSub
                ? parentSub.addNestedSub(handleChangeWrapper)
                : store.subscribe(handleChangeWrapper)

            listeners = createListenerCollection()
        }
    }
    function tryUnsubscribe() {
        subscriptionsAmount--
        if (unsubscribe && subscriptionsAmount === 0) {
            unsubscribe()
            unsubscribe = undefined
            listeners.clear()
            listeners = nullListeners
        }
    }
    function trySubscribeSelf() {
        if (!selfSubscribed) {
            selfSubscribed = true
            trySubscribe()
        }
    }
    function tryUnsubscribeSelf() {
        if (selfSubscribed) {
            selfSubscribed = false
            tryUnsubscribe()
        }
    }
    const subscription: Subscription = {
        addNestedSub,
        notifyNestedSubs,
        handleChangeWrapper,
        isSubscribed,
        trySubscribe: trySubscribeSelf,
        tryUnsubscribe: tryUnsubscribeSelf,
        getListeners: () => listeners,
    }
    return subscription
}
```

#### 4. 什么时候在subscription注册订阅（useSelector）

#### 当我们在组件里面执行useSelector时候其实就是往subscription上注册回调函数，其核心主要是调用react的useSyncExternalStore：

1. **通过useMemo创建一个优化后的Selector，首先会判断store状态快照发生变化没有，如果快照变化了再判断Selector选择的状态有没有发生改变，如果选择的发生改变了就返回新的值，没有改变就返回之前引用**
2. **调用react的useSyncExternalStore钩子，把注册订阅的方法以及优化后的Selector给这个钩子**
3. **useSyncExternalStore会调用传入的注册订阅的方法，把handleStoreChange注册进去，而handleStoreChange的作用在于判断是否需要更新，需要的话就调度执行react的更新，而这个判断的依据就是当前选择的状态和之前的是否一致**

```javascript
// useSelector
const useSelector = () => {
        // 主要记下入参
        const selectedState = useSyncExternalStoreWithSelector(
            subscription.addNestedSub, // 往subscription注册回调
            store.getState, // 获取store快照
            getServerState || store.getState,
            wrappedSelector, // useSelector的入参，(store) => store.name.value
            equalityFn, // 比较函数
        )
        return selectedState
 }
// useSyncExternalStoreWithSelector
function useSyncExternalStoreWithSelector<Snapshot, Selection>(
  subscribe: (() => void) => () => void,
  getSnapshot: () => Snapshot,
  getServerSnapshot: void | null | (() => Snapshot),
  selector: (snapshot: Snapshot) => Selection,
  isEqual?: (a: Selection, b: Selection) => boolean,
): Selection {
  const instRef = useRef(null);
  let inst;
  if (instRef.current === null) {
    inst = {
      hasValue: false,
      value: (null: Selection | null),
    };
    instRef.current = inst;
  } else {
    inst = instRef.current;
  }
  const [getSelection, getServerSelection] = useMemo(() => {
    let hasMemo = false;
    let memoizedSnapshot;
    let memoizedSelection;
    const memoizedSelector = nextSnapshot => {
      if (!hasMemo) {
        hasMemo = true;
        memoizedSnapshot = nextSnapshot;
        const nextSelection = selector(nextSnapshot);
        if (isEqual !== undefined) {
          if (inst.hasValue) {
            const currentSelection = inst.value;
            if (isEqual(currentSelection, nextSelection)) {
              memoizedSelection = currentSelection;
              return currentSelection;
            }
          }
        }
        memoizedSelection = nextSelection;
        return nextSelection;
      }
			const prevSnapshot: Snapshot = (memoizedSnapshot: any);
      const prevSelection: Selection = (memoizedSelection: any);

      if (is(prevSnapshot, nextSnapshot)) {
        return prevSelection;
      }
      const nextSelection = selector(nextSnapshot);
			if (isEqual !== undefined && isEqual(prevSelection, nextSelection)) {
        return prevSelection;
      }
      memoizedSnapshot = nextSnapshot;
      memoizedSelection = nextSelection;
      return nextSelection;
    };
    const maybeGetServerSnapshot =
      getServerSnapshot === undefined ? null : getServerSnapshot;
    const getSnapshotWithSelector = () => memoizedSelector(getSnapshot());
    const getServerSnapshotWithSelector =
      maybeGetServerSnapshot === null
        ? undefined
        : () => memoizedSelector(maybeGetServerSnapshot());
    return [getSnapshotWithSelector, getServerSnapshotWithSelector];
  }, [getSnapshot, getServerSnapshot, selector, isEqual]);
  
  const value = useSyncExternalStore(
    subscribe,
    getSelection,
    getServerSelection,
  );
  
  useEffect(() => {
    inst.hasValue = true;
    inst.value = value;
  }, [value]);
  useDebugValue(value);
  return value;
}
// useSyncExternalStore分为初次渲染和更新，两种情况调用的方法不一样
// mount时
function mountSyncExternalStore<T>(
    subscribe: (() => void) => () => void,
        getSnapshot: () => T,
            getServerSnapshot ?: () => T,
): T {
    const fiber = currentlyRenderingFiber;
    const hook = mountWorkInProgressHook();
    let nextSnapshot;
    const isHydrating = getIsHydrating();
    if (isHydrating) {
        if (getServerSnapshot === undefined) {
          
        }
        nextSnapshot = getServerSnapshot();
    } else {
        nextSnapshot = getSnapshot();
        const root: FiberRoot | null = getWorkInProgressRoot();

        if (root === null) {
        }
        if (!includesBlockingLane(root, renderLanes)) {
            pushStoreConsistencyCheck(fiber, getSnapshot, nextSnapshot);
        }
    }
    hook.memoizedState = nextSnapshot;
    const inst: StoreInstance<T> = {
        value: nextSnapshot,
        getSnapshot,
    };
    hook.queue = inst;
    mountEffect(subscribeToStore.bind(null, fiber, inst, subscribe), [subscribe]);
    fiber.flags |= PassiveEffect;
    pushEffect(
        HookHasEffect | HookPassive,
        updateStoreInstance.bind(null, fiber, inst, nextSnapshot, getSnapshot),
        undefined,
        null,
    );

    return nextSnapshot;
}
// 下面这函数是更新的关键
function subscribeToStore(fiber, inst, subscribe) {
    const handleStoreChange = () => {
        if (checkIfSnapshotChanged(inst)) {
            forceStoreRerender(fiber);
        }
    };
    return subscribe(handleStoreChange);
}
function forceStoreRerender(fiber) {
    scheduleUpdateOnFiber(fiber, SyncLane, NoTimestamp);
}
// update时候
function updateSyncExternalStore<T>(
    subscribe: (() => void) => () => void,
        getSnapshot: () => T,
            getServerSnapshot ?: () => T,
): T {
    const fiber = currentlyRenderingFiber;
    const hook = updateWorkInProgressHook();
    const nextSnapshot = getSnapshot();
    const prevSnapshot = hook.memoizedState;
    const snapshotChanged = !is(prevSnapshot, nextSnapshot);
    if (snapshotChanged) {
        hook.memoizedState = nextSnapshot;
        markWorkInProgressReceivedUpdate();
    }
    const inst = hook.queue;
    updateEffect(subscribeToStore.bind(null, fiber, inst, subscribe), [
        subscribe,
    ]);
    if (
        inst.getSnapshot !== getSnapshot ||
        snapshotChanged ||
        (workInProgressHook !== null &&
            workInProgressHook.memoizedState.tag & HookHasEffect)
    ) {
        fiber.flags |= PassiveEffect;
        pushEffect(
            HookHasEffect | HookPassive,
            updateStoreInstance.bind(null, fiber, inst, nextSnapshot, getSnapshot),
            undefined,
            null,
        );
        const root: FiberRoot | null = getWorkInProgressRoot();

        if (root === null) {
          
        }
        if (!includesBlockingLane(root, renderLanes)) {
            pushStoreConsistencyCheck(fiber, getSnapshot, nextSnapshot);
        }
    }
    return nextSnapshot;
}

```

### 5.总结

1. **首先创建一个redux的store状态管理对象，在Provider组件中会创建一个subscription订阅管理对象，把subscription的批量执行注册的回调函数的函数注册到store状态管理对象里面，当store状态发生改变后会就触发subscription订阅的回调的批量执行。**

2. **useSelector会向subscription注册一个回调函数，而这个回调函数的作用是判断当前选择的状态和之前的有没有发生改变，如果发生改变了就会执行react的调度更新**

3. **所以当store更新后就会执行react的调度更新**

