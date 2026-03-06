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
4. **触发回调，当派发状态更新后会立马执行订阅的函数。**

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

### 4. 什么时候在subscription注册订阅（useSelector）

#### 当我们在组件里面执行useSelector时候其实就是往subscription上注册回调函数，这个回调函数就是判断当前组件是否需要调度更新，其核心主要是调用react的useSyncExternalStore：

1. **通过useMemo创建一个优化后的Selector，首先会判断store状态快照发生变化没有，如果快照变化了再判断Selector选择的状态有没有发生改变，如果选择的发生改变了就返回新的值，没有改变就返回之前引用**
2. **调用react的useSyncExternalStore钩子，把注册订阅的方法以及优化后的Selector给这个钩子**
3. **useSyncExternalStore会调用传入的注册订阅的方法，把handleStoreChange注册进去，而handleStoreChange的作用在于判断是否需要更新，需要的话就调度执行react的更新，而这个判断的依据就是当前选择的状态和之前的是否一致**

```javascript
// useSelector
const useSelector = () => {
        // 主要看下入参
        const selectedState = useSyncExternalStoreWithSelector(
            subscription.addNestedSub, // 往subscription注册回调的方法
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
    // 向subscribe注册判断是否需要执行调度更新的函数
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
// 判断当前选择的状态和之前的是否一样
function checkIfSnapshotChanged(inst) {
    const latestGetSnapshot = inst.getSnapshot;
    const prevValue = inst.value;
    try {
        const nextValue = latestGetSnapshot();
        return !is(prevValue, nextValue);
    } catch (error) {
        return true;
    }
}
// 执行调度更新
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
    // 判断选择的状态是否改变，改变后就标记需要更新
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
### 5.中间件的实现
#### 中间件其实就是对Store的一个增强，本质就是一个函数，函数接受store的dispatch和getState，函数内部返回另一个函数，返回的这个函数接受一个参数，这个参数就是下一个中间件函数

```javascript
function applyMiddleware(
  ...middlewares: Middleware[]
): StoreEnhancer<any> {
  return createStore => (reducer, preloadedState) => {
    const store = createStore(reducer, preloadedState)
    let dispatch: Dispatch = () => {
      throw new Error(
        'Dispatching while constructing your middleware is not allowed. ' +
          'Other middleware would not be applied to this dispatch.'
      )
    }
    const middlewareAPI: MiddlewareAPI = {
      getState: store.getState,
      dispatch: (action, ...args) => dispatch(action, ...args)
    }
    // 为每个中间件提供store的访问能力
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    // 可以看到中间件其实就是对dispatch的增强
    dispatch = compose<typeof dispatch>(...chain)(store.dispatch)
    return {
      ...store,
      dispatch
    }
  }
}
// 中间件的执行顺序是右边到左，链式串行
function compose(...funcs: Function[]) {
  if (funcs.length === 0) {
    return <T>(arg: T) => arg
  }
  if (funcs.length === 1) {
    return funcs[0]
  }
  return funcs.reduce(
    (a, b) =>
      (...args: any) =>
        a(b(...args))
  )
}
// 自定义中间示例
const thunkMiddleware = (store) => {
  return (next) => {
    return (action) => {
      if (typeof action === 'function') {
        return action(store.dispatch, store.getState)
      }
      return next(action)
    }
  }
}
// 中间件标准结构
const middleware = (store) => (next) => (action) => {
  // 处理逻辑
  return next(action)
}
// 调用中间件时候就相当于是
compose<typeof dispatch>(...chain)(store.dispatch)(action);
// 金典的洋葱模型
compose(f, g, h)(x) = f(g(h(x)));
```

### 5.5 React-Redux 高级使用：自定义 Context Hooks

#### 为什么需要自定义 Context？

默认情况下，`useSelector`、`useDispatch`、`useStore` 使用 React-Redux 内置的 Context。但在以下场景需要隔离：

- **多 Store 架构**：微前端、独立模块需要各自的状态树
- **状态隔离**：避免不同业务模块的状态互相干扰
- **测试场景**：单元测试时注入 mock store

React-Redux 提供了三个工厂函数来创建绑定到特定 Context 的 Hooks：

```javascript
import {
  createStoreHook,
  createDispatchHook,
  createSelectorHook,
} from 'react-redux';
```

#### 基础示例：创建独立模块的 Store

```javascript
import { createContext } from 'react';
import { Provider } from 'react-redux';
import {
  createStoreHook,
  createDispatchHook,
  createSelectorHook,
} from 'react-redux';
import { configureStore, createSlice } from '@reduxjs/toolkit';

// ====== 1. 创建 Chat 模块的 Slice ======
const chatSlice = createSlice({
  name: 'chat',
  initialState: {
    currentBotId: null,
    isAnswering: false,
    inputValue: '',
    messages: [],
  },
  reducers: {
    setCurrentBot: (state, action) => {
      state.currentBotId = action.payload;
    },
    setAnswering: (state, action) => {
      state.isAnswering = action.payload;
    },
    setInputValue: (state, action) => {
      state.inputValue = action.payload;
    },
    addMessage: (state, action) => {
      state.messages.push(action.payload);
    },
    clearInput: (state) => {
      state.inputValue = '';
    },
  },
});

export const chatActions = chatSlice.actions;

// ====== 2. 创建独立的 Store ======
const chatStore = configureStore({
  reducer: chatSlice.reducer,
});

// ====== 3. 创建自定义 Context ======
const ChatContext = createContext(null);

// ====== 4. 创建绑定到该 Context 的 Hooks ======
export const useChatStore = createStoreHook(ChatContext);
export const useChatDispatch = createDispatchHook(ChatContext);
export const useChatSelector = createSelectorHook(ChatContext);

// ====== 5. 创建 Provider 组件 ======
export function ChatProvider({ children }) {
  return (
    <Provider context={ChatContext} store={chatStore}>
      {children}
    </Provider>
  );
}

// ====== 6. 在组件中使用 ======
function ChatInput() {
  // 使用自定义的 Hooks，只访问 ChatContext 中的 store
  const inputValue = useChatSelector((state) => state.inputValue);
  const isAnswering = useChatSelector((state) => state.isAnswering);
  const dispatch = useChatDispatch();

  const handleChange = (e) => {
    dispatch(chatActions.setInputValue(e.target.value));
  };

  const handleSend = () => {
    dispatch(chatActions.setAnswering(true));
    // 发送逻辑...
  };

  return (
    <div>
      <input value={inputValue} onChange={handleChange} />
      <button disabled={isAnswering} onClick={handleSend}>
        {isAnswering ? 'Sending...' : 'Send'}
      </button>
    </div>
  );
}

function MessageList() {
  const messages = useChatSelector((state) => state.messages);

  return (
    <ul>
      {messages.map((msg, idx) => (
        <li key={idx}>{msg.content}</li>
      ))}
    </ul>
  );
}

// ====== 7. 应用入口 ======
function App() {
  return (
    <ChatProvider>
      <ChatInput />
      <MessageList />
    </ChatProvider>
  );
}
```

#### 进阶示例：多 Store 共存

```javascript
import React, { createContext } from 'react';
import { Provider, useSelector, useDispatch } from 'react-redux';
import {
  createStoreHook,
  createDispatchHook,
  createSelectorHook,
} from 'react-redux';
import { configureStore, createSlice } from '@reduxjs/toolkit';

// ==================== 全局 Store ====================
// 使用默认 Context，不需要创建

const globalSlice = createSlice({
  name: 'global',
  initialState: {
    theme: 'light',
    language: 'zh-CN',
    user: null,
  },
  reducers: {
    setTheme: (state, action) => {
      state.theme = action.payload;
    },
    setUser: (state, action) => {
      state.user = action.payload;
    },
  },
});

const globalStore = configureStore({
  reducer: globalSlice.reducer,
});

export const globalActions = globalSlice.actions;

// ==================== Chat 模块 Store ====================

const ChatContext = createContext(null);

const chatSlice = createSlice({
  name: 'chat',
  initialState: {
    currentBotId: null,
    isAnswering: false,
    inputValue: '',
  },
  reducers: {
    setCurrentBot: (state, action) => {
      state.currentBotId = action.payload;
    },
    setAnswering: (state, action) => {
      state.isAnswering = action.payload;
    },
    setInputValue: (state, action) => {
      state.inputValue = action.payload;
    },
    clearInput: (state) => {
      state.inputValue = '';
    },
  },
});

const chatStore = configureStore({
  reducer: chatSlice.reducer,
});

export const chatActions = chatSlice.actions;
export const useChatDispatch = createDispatchHook(ChatContext);
export const useChatSelector = createSelectorHook(ChatContext);

// ==================== Editor 模块 Store ====================

const EditorContext = createContext(null);

const editorSlice = createSlice({
  name: 'editor',
  initialState: {
    content: '',
    isModified: false,
    history: [],
  },
  reducers: {
    setContent: (state, action) => {
      state.content = action.payload;
      state.isModified = true;
    },
    save: (state) => {
      state.history.push(state.content);
      state.isModified = false;
    },
    reset: (state) => {
      state.content = '';
      state.isModified = false;
    },
  },
});

const editorStore = configureStore({
  reducer: editorSlice.reducer,
});

export const editorActions = editorSlice.actions;
export const useEditorDispatch = createDispatchHook(EditorContext);
export const useEditorSelector = createSelectorHook(EditorContext);

// ==================== 组件中使用多个 Store ====================

function Header() {
  // 全局状态（默认 Context）
  const theme = useSelector((state) => state.theme);
  const user = useSelector((state) => state.user);
  const globalDispatch = useDispatch();

  return (
    <header className={theme}>
      <span>{user?.name || 'Guest'}</span>
      <button onClick={() => globalDispatch(globalActions.setTheme(
        theme === 'light' ? 'dark' : 'light'
      ))}>
        Toggle Theme
      </button>
    </header>
  );
}

function ChatPanel() {
  // Chat 模块状态
  const inputValue = useChatSelector((state) => state.inputValue);
  const isAnswering = useChatSelector((state) => state.isAnswering);
  const chatDispatch = useChatDispatch();

  // 也可以访问全局状态
  const theme = useSelector((state) => state.theme);

  return (
    <div className={`chat-panel ${theme}`}>
      <input
        value={inputValue}
        onChange={(e) => chatDispatch(chatActions.setInputValue(e.target.value))}
        placeholder="输入消息..."
      />
      <button
        disabled={isAnswering || !inputValue}
        onClick={() => {
          chatDispatch(chatActions.setAnswering(true));
          // 发送后清空
          setTimeout(() => {
            chatDispatch(chatActions.clearInput());
            chatDispatch(chatActions.setAnswering(false));
          }, 1000);
        }}
      >
        发送
      </button>
    </div>
  );
}

function EditorPanel() {
  // Editor 模块状态
  const content = useEditorSelector((state) => state.content);
  const isModified = useEditorSelector((state) => state.isModified);
  const editorDispatch = useEditorDispatch();

  return (
    <div className="editor-panel">
      <textarea
        value={content}
        onChange={(e) => editorDispatch(editorActions.setContent(e.target.value))}
      />
      <div>
        <button
          disabled={!isModified}
          onClick={() => editorDispatch(editorActions.save())}
        >
          保存 {isModified && '*'}
        </button>
        <button onClick={() => editorDispatch(editorActions.reset())}>
          重置
        </button>
      </div>
    </div>
  );
}

// ==================== 多层 Provider 嵌套 ====================

function App() {
  return (
    // 全局 Store（默认 Context）
    <Provider store={globalStore}>
      {/* Chat 模块 Store */}
      <Provider context={ChatContext} store={chatStore}>
        {/* Editor 模块 Store */}
        <Provider context={EditorContext} store={editorStore}>
          <div className="app">
            <Header />
            <main>
              <ChatPanel />
              <EditorPanel />
            </main>
          </div>
        </Provider>
      </Provider>
    </Provider>
  );
}

export default App;
```

#### 高级示例：封装可复用的模块化 Store

```javascript
import React, { createContext, useMemo } from 'react';
import { Provider } from 'react-redux';
import {
  createStoreHook,
  createDispatchHook,
  createSelectorHook,
} from 'react-redux';
import { configureStore } from '@reduxjs/toolkit';

/**
 * 创建模块化 Store 的工厂函数
 * 返回：Context、Hooks、Provider、getStore
 */
function createModuleStore(options) {
  const { name, reducer, middleware = [] } = options;

  // 创建 Context
  const Context = createContext(null);
  Context.displayName = `${name}Context`;

  // 创建 Hooks
  const useModuleStore = createStoreHook(Context);
  const useModuleDispatch = createDispatchHook(Context);
  const useModuleSelector = createSelectorHook(Context);

  // Store 引用（用于跨模块通信）
  let storeRef = null;

  // Provider 组件
  function ModuleProvider({ children, initialState }) {
    const store = useMemo(() => {
      const s = configureStore({
        reducer,
        preloadedState: initialState,
        middleware: (getDefault) => getDefault().concat(middleware),
        devTools: { name }, // Redux DevTools 中显示的名称
      });
      storeRef = s;
      return s;
    }, []);

    return (
      <Provider context={Context} store={store}>
        {children}
      </Provider>
    );
  }

  return {
    Context,
    useStore: useModuleStore,
    useDispatch: useModuleDispatch,
    useSelector: useModuleSelector,
    Provider: ModuleProvider,
    getStore: () => storeRef,
  };
}

// ==================== 使用工厂函数创建模块 ====================

import { createSlice } from '@reduxjs/toolkit';

// Chat 模块
const chatSlice = createSlice({
  name: 'chat',
  initialState: {
    botId: null,
    messages: [],
    input: '',
  },
  reducers: {
    setBotId: (state, action) => { state.botId = action.payload; },
    setInput: (state, action) => { state.input = action.payload; },
    addMessage: (state, action) => { state.messages.push(action.payload); },
    clearMessages: (state) => { state.messages = []; },
  },
});

const ChatModule = createModuleStore({
  name: 'Chat',
  reducer: chatSlice.reducer,
});

export const {
  useSelector: useChatSelector,
  useDispatch: useChatDispatch,
  Provider: ChatProvider,
  getStore: getChatStore,
} = ChatModule;

export const chatActions = chatSlice.actions;

// Editor 模块
const editorSlice = createSlice({
  name: 'editor',
  initialState: {
    content: [],
    selection: null,
    history: [],
  },
  reducers: {
    setContent: (state, action) => { state.content = action.payload; },
    setSelection: (state, action) => { state.selection = action.payload; },
    pushHistory: (state) => { state.history.push([...state.content]); },
    undo: (state) => {
      if (state.history.length > 0) {
        state.content = state.history.pop();
      }
    },
  },
});

const EditorModule = createModuleStore({
  name: 'Editor',
  reducer: editorSlice.reducer,
});

export const {
  useSelector: useEditorSelector,
  useDispatch: useEditorDispatch,
  Provider: EditorProvider,
  getStore: getEditorStore,
} = EditorModule;

export const editorActions = editorSlice.actions;

// ==================== 跨模块通信 ====================

/**
 * 跨模块操作 Hook
 * 当需要同时操作多个模块时使用
 */
function useCrossModuleActions() {
  const chatDispatch = useChatDispatch();
  const editorDispatch = useEditorDispatch();

  return useMemo(() => ({
    // 发送消息并保存到编辑器
    sendAndSaveToEditor: async (content) => {
      // 1. 发送消息
      chatDispatch(chatActions.addMessage({
        role: 'user',
        content
      }));
      chatDispatch(chatActions.setInput(''));

      // 2. 模拟 AI 回复
      await new Promise(resolve => setTimeout(resolve, 1000));
      const aiResponse = `AI Response to: ${content}`;

      chatDispatch(chatActions.addMessage({
        role: 'assistant',
        content: aiResponse
      }));

      // 3. 保存到编辑器
      editorDispatch(editorActions.pushHistory());
      editorDispatch(editorActions.setContent([
        { type: 'paragraph', text: aiResponse }
      ]));
    },

    // 清空所有模块状态
    resetAll: () => {
      chatDispatch(chatActions.clearMessages());
      chatDispatch(chatActions.setInput(''));
      // Editor 的 reset 逻辑...
    },
  }), [chatDispatch, editorDispatch]);
}

// ==================== 完整应用示例 ====================

function ChatInput() {
  const input = useChatSelector(state => state.input);
  const dispatch = useChatDispatch();
  const { sendAndSaveToEditor } = useCrossModuleActions();

  return (
    <div>
      <input
        value={input}
        onChange={e => dispatch(chatActions.setInput(e.target.value))}
        onKeyPress={e => {
          if (e.key === 'Enter' && input) {
            sendAndSaveToEditor(input);
          }
        }}
      />
    </div>
  );
}

function ChatMessages() {
  const messages = useChatSelector(state => state.messages);

  return (
    <div>
      {messages.map((msg, i) => (
        <div key={i} className={msg.role}>
          {msg.content}
        </div>
      ))}
    </div>
  );
}

function Editor() {
  const content = useEditorSelector(state => state.content);
  const history = useEditorSelector(state => state.history);
  const dispatch = useEditorDispatch();

  return (
    <div>
      <div className="toolbar">
        <button
          disabled={history.length === 0}
          onClick={() => dispatch(editorActions.undo())}
        >
          撤销 ({history.length})
        </button>
      </div>
      <div className="content">
        {content.map((block, i) => (
          <p key={i}>{block.text}</p>
        ))}
      </div>
    </div>
  );
}

function App() {
  return (
    <ChatProvider initialState={{ botId: 'bot-1', messages: [], input: '' }}>
      <EditorProvider initialState={{ content: [], selection: null, history: [] }}>
        <div className="app">
          <div className="chat-panel">
            <ChatMessages />
            <ChatInput />
          </div>
          <div className="editor-panel">
            <Editor />
          </div>
        </div>
      </EditorProvider>
    </ChatProvider>
  );
}

export default App;
```

#### 总结

| API | 作用 |
|-----|------|
| `createSelectorHook(Context)` | 创建绑定到指定 Context 的 useSelector |
| `createDispatchHook(Context)` | 创建绑定到指定 Context 的 useDispatch |
| `createStoreHook(Context)` | 创建绑定到指定 Context 的 useStore |

| 使用场景 | 方案 |
|----------|------|
| 单一全局状态 | 使用默认的 useSelector/useDispatch |
| 多模块独立状态 | 每个模块创建独立 Context + Store |
| 跨模块通信 | 通过 getStore() 或封装 Cross-Module Hooks |
| 微前端架构 | 每个子应用使用独立的 Context |

### 6.整体流程

1. **首先创建一个redux的store状态管理对象，在Provider组件中会创建一个subscription订阅管理对象，把subscription的<u><span style='color: red'>批量执行注册的回调函数的函数<span></u>注册到store状态管理对象里面，当store状态发生改变后会就触发subscription订阅的回调的批量执行。**

2. **useSelector会向subscription注册一个回调函数，而这个回调函数的作用是判断当前选择的状态和之前的有没有发生改变，如果发生改变了就会执行react的调度更新**

3. **所以当store更新后就会执行react的调度更新**
