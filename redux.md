**Redux的源码理解**

redux核心库的基础用法：
```javascript
import { createStore, combineReducers } from 'redux'

const initState = 2
const reducer1 = (state: any = initState, action: {type: string, payload: any}) => {
  switch(action.type) {
    case 'action1':
      // 对state做相应修改并返回
      // return state - 1;
    case 'action2':
      // 对state做相应修改并返回
      // return state - 2;
    default:
      return state;
  }
}

const reducer2 = (state: any, action: {type: string, payload: any}) => {
  switch(action.type) {
    case 'action1':
      // 对state做相应修改并返回
      // return state - 1;
    case 'action2':
      // 对state做相应修改并返回
      // return state - 2;
    default:
      return state;
  }
}
// 如果想要创建单一store
const singleStore = createStore(reducer1, 2) // 第二个参数是state的初始值，可不传在定义reducer时候定义
// 创建更细粒的store
const reducer = combineReducers({
  store1: reducer1,
  store2: reducer2,
})
const store = createStore(reducer)
// 创建的store是一个对象
// {
//     dispatch,  触发更新dispatch({type: 'actionType', payload: 其余参数})
//     subscribe, 注册订阅 subscribe(() => {})
//     getState,  获取状态
//     replaceReducer, 替换reducer
//     [$$observable]: observable
// }
```
redux的createStore的源码：
```javascript
  export function createStore<
  S,
  A extends Action,
  Ext extends {} = {},
  StateExt extends {} = {},
  PreloadedState = S
>(
  reducer: Reducer<S, A, PreloadedState>,
  preloadedState?: PreloadedState | undefined,
  enhancer?: StoreEnhancer<Ext, StateExt>
): Store<S, A, UnknownIfNonSpecific<StateExt>> & Ext
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
    throw new Error()
  }

  if (
    (typeof preloadedState === 'function' && typeof enhancer === 'function') ||
    (typeof enhancer === 'function' && typeof arguments[3] === 'function')
  ) {
    throw new Error()
  }

  if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
    enhancer = preloadedState as StoreEnhancer<Ext, StateExt>
    preloadedState = undefined
  }

  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error()
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

  function ensureCanMutateNextListeners() {
    if (nextListeners === currentListeners) {
      nextListeners = new Map()
      currentListeners.forEach((listener, key) => {
        nextListeners.set(key, listener)
      })
    }
  }
// 获取状态
  function getState(): S {
    if (isDispatching) {
      throw new Error()
    }
    return currentState as S
  }
// 注册订阅事件
  function subscribe(listener: () => void) {
    if (typeof listener !== 'function') {
      throw new Error()
    }

    if (isDispatching) {
    //正在触发更新时候注册会报错
      throw new Error()
    }
    let isSubscribed = true
    ensureCanMutateNextListeners()
    const listenerId = listenerIdCounter++
    nextListeners.set(listenerId, listener)
    // 返回一个解除注册的函数
    return function unsubscribe() {
      if (!isSubscribed) {
        return
      }
      if (isDispatching) {
      //正在触发更新时候解除注册会报错
        throw new Error()
      }
      isSubscribed = false
      ensureCanMutateNextListeners()
      nextListeners.delete(listenerId)
      currentListeners = null
    }
  }
// 出发订阅的事件，并且更改state的值
  function dispatch(action: A) {
    if (!isPlainObject(action)) {
    // 判断action
      throw new Error()
    }
    if (typeof action.type === 'undefined') {
      throw new Error()
    }
    if (typeof action.type !== 'string') {
      throw new Error()
    }
    if (isDispatching) {
      throw new Error('Reducers may not dispatch actions.')
    }
    try {
      isDispatching = true
      currentState = currentReducer(currentState, action)
    } finally {
      isDispatching = false
    }
    const listeners = (currentListeners = nextListeners)
    listeners.forEach(listener => {
      listener()
    })
    return action
  }
// 替换掉reducer
  function replaceReducer(nextReducer: Reducer<S, A>): void {
    if (typeof nextReducer !== 'function') {
      throw new Error()
    }
    currentReducer = nextReducer as unknown as Reducer<S, A, PreloadedState>
    dispatch({ type: ActionTypes.REPLACE } as A)
  }

  /**
   * Interoperability point for observable/reactive libraries.
   * @returns A minimal observable of state changes.
   * For more information, see the observable proposal:
   * https://github.com/tc39/proposal-observable
   */
  function observable() {
    const outerSubscribe = subscribe
    return {
      /**
       * The minimal observable subscription method.
       * @param observer Any object that can be used as an observer.
       * The observer object should have a `next` method.
       * @returns An object with an `unsubscribe` method that can
       * be used to unsubscribe the observable from the store, and prevent further
       * emission of values from the observable.
       */
      subscribe(observer: unknown) {
        if (typeof observer !== 'object' || observer === null) {
          throw new TypeError(
            `Expected the observer to be an object. Instead, received: '${kindOf(
              observer
            )}'`
          )
        }

        function observeState() {
          const observerAsObserver = observer as Observer<S>
          if (observerAsObserver.next) {
            observerAsObserver.next(getState())
          }
        }

        observeState()
        const unsubscribe = outerSubscribe(observeState)
        return { unsubscribe }
      },

      [$$observable]() {
        return this
      }
    }
  }

  dispatch({ type: ActionTypes.INIT } as A)

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
combineReducers的源码
```javascript
export default function combineReducers(reducers: {
  [key: string]: Reducer<any, any, any>
}) {
  const reducerKeys = Object.keys(reducers)
  const finalReducers: { [key: string]: Reducer<any, any, any> } = {}
  for (let i = 0; i < reducerKeys.length; i++) {
    const key = reducerKeys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (typeof reducers[key] === 'undefined') {
        warning(`No reducer provided for key "${key}"`)
      }
    }

    if (typeof reducers[key] === 'function') {
      finalReducers[key] = reducers[key]
    }
  }
  const finalReducerKeys = Object.keys(finalReducers)
  let unexpectedKeyCache: { [key: string]: true }
  if (process.env.NODE_ENV !== 'production') {
    unexpectedKeyCache = {}
  }

  let shapeAssertionError: unknown
  try {
    assertReducerShape(finalReducers)
  } catch (e) {
    shapeAssertionError = e
  }

  return function combination(
    state: StateFromReducersMapObject<typeof reducers> = {},
    action: Action
  ) {
    if (shapeAssertionError) {
      throw shapeAssertionError
    }

    if (process.env.NODE_ENV !== 'production') {
      const warningMessage = getUnexpectedStateShapeWarningMessage(
        state,
        finalReducers,
        action,
        unexpectedKeyCache
      )
      if (warningMessage) {
        warning(warningMessage)
      }
    }

    let hasChanged = false
    const nextState: StateFromReducersMapObject<typeof reducers> = {}
    for (let i = 0; i < finalReducerKeys.length; i++) {
      const key = finalReducerKeys[i]
      const reducer = finalReducers[key]
      const previousStateForKey = state[key]
      const nextStateForKey = reducer(previousStateForKey, action)
      if (typeof nextStateForKey === 'undefined') {
        const actionType = action && action.type
        throw new Error()
      }
      nextState[key] = nextStateForKey
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey
    }
    hasChanged =
      hasChanged || finalReducerKeys.length !== Object.keys(state).length
    return hasChanged ? nextState : state
  }
}
```
