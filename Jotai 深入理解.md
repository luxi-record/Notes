# Jotai 深入理解

## 核心设计理念

### 1. 原子化状态管理

Jotai 的核心思想是将应用状态分解为最小的、独立的单元（atom），然后通过组合这些单元来构建复杂的状态。不需要手动声明依赖关系，系统会自动追踪。像堆积木一样，每个积木是独立的，可以组合成任何形状。

### 2. 使用示例

```typescript
// 定义 atom
const countAtom = atom(0)
// 派生的 atom
const doubleAtom = atom((get) => get(countAtom) * 2)
const isEvenAtom = atom((get) => get(countAtom) % 2 === 0)

// 在组件中使用
function Counter() {
  const [count, setCount] = useAtom(countAtom)
  const double = useAtomValue(doubleAtom)
  const isEven = useAtomValue(isEvenAtom)
  
  return (
    <div>
      <div>Count: {count}</div>
      <div>Double: {double}</div>
      <div>Is Even: {isEven ? 'Yes' : 'No'}</div>
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  )
}
```

## 核心数据结构

### 要理解 Jotai 首先需要知道其内部的主要几种数据结构：Atom、AtomState、Store、Mounted。

- **Atom（定义状态和行为的描述对象）**
  - **定义**：
    ```typescript
    interface Atom<Value> {
      read: Read<Value>  // 读取函数
      unstable_is?: (a: Atom<unknown>) => boolean
      debugLabel?: string
    }
    
    interface WritableAtom<Value, Args, Result> extends Atom<Value> {
      read: Read<Value, SetAtom<Args, Result>>
      write: Write<Args, Result>  // 写入函数
      onMount?: OnMount<Args, Result>
    }
    ```
  - **作用**：
    - **描述状态**：定义这个 atom 如何读（`read`）/写（`write`）
    - **作为 key**：在 `atomStateMap` / `mountedMap` 里作为 WeakMap 的 key
    - **生命周期钩子**：`onMount`、`unstable_onInit` 等
  - **特点**：
    - **不可变配置**：创建后一般不修改
    - **唯一标识**：同一个 atom 实例在整个应用中是唯一的

- **AtomState（某个 atom 在某个 store 中的运行时状态）**
  - **定义**：
    ```typescript
    type AtomState<Value> = {
      d: Map<AnyAtom, EpochNumber>  // 依赖映射：依赖哪个 atom，以及依赖时它的版本号
      p: Set<AnyAtom>               // 有 pending Promise 的依赖 atom 集合
      n: EpochNumber                // 版本号（epoch number）
      v?: Value                     // 当前值（可以是 Promise）
      e?: AnyError                  // 当前错误
    }
    ```
  - **作用**：
    - 存储 atom 的**当前值/错误**（`v`/`e`）
    - 记录 atom 的**版本号**（`n`），用于判断是否变化
    - 记录依赖关系（`d`），用于缓存和依赖检查
    - 记录 pending Promise 依赖（`p`），用于异步依赖处理
  - **特点**：
    - **可变**：值、版本号、依赖会随着计算/写入而变化
    - **持久化**：即使 atom 未挂载，AtomState 仍然保留（在 WeakMap 里）
    - **生命周期**：
      - 创建：首次 `readAtomState` / `writeAtomState` 时通过 `ensureAtomState` 创建
      - 销毁：当 atom 对象被 GC 后，WeakMap 条目自动清理

- **Mounted（订阅和依赖图层面的挂载信息）**
  - **定义**：
    ```typescript
    type Mounted = {
      l: Set<() => void>  // 监听器集合（订阅者回调）
      d: Set<AnyAtom>     // 当前依赖的已挂载 atom 集合
      t: Set<AnyAtom>     // 依赖当前 atom 的已挂载 atom 集合（反向依赖）
      u?: () => void      // 卸载回调（onMount 返回的函数）
    }
    ```
  - **作用**：
    - 记录有哪些订阅者需要在值变化时被通知（`l`）
    - 记录挂载层面的依赖关系（`d`/`t`），用于更新传播和自动挂载/卸载
    - 保存卸载时需要执行的清理逻辑（`u`）
  - **特点**：
    - **临时性**：只有有订阅者或被其他挂载 atom 依赖时才存在
    - **生命周期**：
      - 创建：第一次 `store.sub(atom, listener)` 或作为已挂载 atom 的依赖时，通过 `mountAtom` 创建
      - 销毁：`store.sub` 返回的取消函数被调用且不再被依赖时，通过 `unmountAtom` 删除

- **Store（状态容器，持有一组 AtomState/Mounted）**
  - **定义**：
    ```typescript
    type Store = {
      get: <Value>(atom: Atom<Value>) => Value
      set: <Value, Args extends unknown[], Result>(
        atom: WritableAtom<Value, Args, Result>,
        ...args: Args
      ) => Result
      sub: (atom: AnyAtom, listener: () => void) => () => void
    }
    ```
  - **作用**：
    - 对外暴露三个核心能力：
      - `get`：读取 atom 的当前值
      - `set`：写入可写 atom
      - `sub`：订阅 atom 的变化
    - 内部管理整套运行时状态：
      - 哪些 atom 有状态（`atomStateMap`）
      - 哪些 atom 被挂载、订阅关系如何（`mountedMap`）
      - 哪些 atom 被标记为失效/已改变（`invalidatedAtoms` / `changedAtoms`）
  - **内部状态**（简化理解）：
    - `atomStateMap: WeakMap<Atom, AtomState>`：每个 atom 的运行时状态
    - `mountedMap: WeakMap<Atom, Mounted>`：每个已挂载 atom 的订阅和依赖信息
    - `invalidatedAtoms: WeakMap<Atom, EpochNumber>`：失效标记
    - `changedAtoms: Set<Atom>`：这一轮更新中实际变化的 atom
    - `mountCallbacks` / `unmountCallbacks`：挂载/卸载时要执行的回调队列


## 了解了这些基本数据结构，再来看 useAtomValue和useSetAtom做了些啥

### 1. `useAtomValue`的源码如下：

  ```typescript
  export function useAtomValue<Value>(atom: Atom<Value>, options?: Options) {
    const { delay, unstable_promiseStatus: promiseStatus = !React.use } =
      options || {}
    const store = useStore(options) // 如果提供options，则使用提供的store，否则使用默认的defaultStore
    const [[valueFromReducer, storeFromReducer, atomFromReducer], rerender] =
      useReducer<readonly [Value, Store, typeof atom], undefined, []>(
        (prev) => {
          const nextValue = store.get(atom)
          // 这是一个优化，如果值没变化，则直接返回之前的 state
          if (
            Object.is(prev[0], nextValue) &&
            prev[1] === store &&
            prev[2] === atom
          ) {
            return prev
          }
          return [nextValue, store, atom]
        },
        undefined,
        () => [store.get(atom), store, atom],
      )

    let value = valueFromReducer
    // 这里当store或者atom的引用发生变化时，强制rerender，重新获取最新的值
    if (storeFromReducer !== store || atomFromReducer !== atom) {
      rerender()
      value = store.get(atom)
    }
    // 可以看到调用useAtomValue会主动订阅atom，当atom变化时，触发rerender
    useEffect(() => {
      const unsub = store.sub(atom, () => {
        if (promiseStatus) {
          try {
            const value = store.get(atom)
            if (isPromiseLike(value)) {
              attachPromiseStatus(
                createContinuablePromise(value, () => store.get(atom)),
              )
            }
          } catch {
            // ignore
          }
        }
        if (typeof delay === 'number') {
          // delay rerendering to wait a promise possibly to resolve
          setTimeout(rerender, delay)
          return
        }
        rerender()
      })
      rerender()
      return unsub
    }, [store, atom, delay, promiseStatus])

    useDebugValue(value)
    // 这里对promise类型的值做了处理
    if (isPromiseLike(value)) {
      const promise = createContinuablePromise(value, () => store.get(atom))
      if (promiseStatus) {
        attachPromiseStatus(promise)
      }
      return use(promise)
    }
    return value as Awaited<Value>
  }
  ```
  
  **核心逻辑总结**：调用useAtomValue时，会订阅atom，当atom值变化时，触发rerender，这个rerender就是React的调度更新。

  
### 2. `useSetAtom`的源码如下：

  ```typescript
  export function useSetAtom<Value, Args extends unknown[], Result>(
    atom: WritableAtom<Value, Args, Result>,
    options?: Options,
  ) {
    // 如果提供options，则使用提供的store，否则使用默认的defaultStore
    const store = useStore(options)
    const setAtom = useCallback(
      (...args: Args) => {
        if (import.meta.env?.MODE !== 'production' && !('write' in atom)) {
          throw new Error('not writable atom')
        }
        // 直接调用store.set(atom, ...args)
        return store.set(atom, ...args)
      },
      [store, atom],
    )
    return setAtom
  }
  ```

  **核心逻辑总结**：调用useSetAtom时，其实就是间接的调用store.get(atom, ...args)，并且不会触发React的重新渲染。

## 可以看出useAtomValue和useSetAtom其实就是对store.get和store.set的封装，并且useAtomValue会订阅atom，当atom变化时，触发rerender，这个rerender就是React的调度更新。

### 终极奥妙 store 的内部处理机制，从数据结构上我们了解到store主要有 get、set、sub 三个核心能力。store的创建是通过creteStore函数，这个函数是暴露出来的，我们可以创建额外单独的store。

```typescript
  // 这是creteStore创建出的store对象
  const store: Store = {
    get: (atom) => returnAtomValue(readAtomState(atom)),
    set: (atom, ...args) => {
      try {
        return writeAtomState(atom, ...args)
      } finally {
        recomputeInvalidatedAtoms()
        flushCallbacks()
      }
    },
    sub: (atom, listener) => {
      const mounted = mountAtom(atom)
      const listeners = mounted.l
      listeners.add(listener)
      flushCallbacks()
      return () => {
        listeners.delete(listener)
        unmountAtom(atom)
        flushCallbacks()
      }
    },
  }
```

### 1. `store.get`：

  ```typescript
  // returnAtomValue没啥好说的，就是返回atom的当前值，atom的值咋来的？？？
  const readAtomState =
    buildingBlockFunctions[3] ||
    ((atom) => {
      // 确保当前 atom 有对应的 AtomState，没有就创建
      const atomState = ensureAtomState(atom)
      // 如果已经有值（初始化过），先尝试走缓存分支
      if (isAtomStateInitialized(atomState)) {
        if (
          // 如果这个 atom 当前是挂载状态
          mountedMap.has(atom) &&
          // 并且 invalidatedAtoms 里记录的版本号和当前版本号不一致
          invalidatedAtoms.get(atom) !== atomState.n
        ) {
          // 说明之前已经在依赖重算里更新过了，直接返回缓存
          return atomState
        }
        if (
          // 否则检查依赖列表 d（Map<依赖Atom, 依赖时的版本号>）
          Array.from(atomState.d).every(
            // 递归读取每个依赖的 AtomState，看版本号是否和记录的一致
            ([a, n]) => readAtomState(a).n === n,
          )
        ) {
          // 所有依赖都没变，当前 atom 也就没必要重算，继续用缓存
          return atomState
        }
      }
      // 走到这里说明缓存不能用，先清空依赖 Map，准备重新收集依赖
      atomState.d.clear()
      // 标记当前是否仍在同步阶段（调用 atom.read 的这一轮）
      let isSync = true
      // 一个辅助函数：在异步 Promise resolve/reject 之后挂载依赖
      const mountDependenciesIfAsync = () => {
        // 只有当前 atom 还处于挂载状态才需要做这些事
        if (mountedMap.has(atom)) {
          // 根据新的依赖关系挂载子依赖
          mountDependencies(atom)
          // 重新计算所有被标记为失效的 atom
          recomputeInvalidatedAtoms()
          // 执行订阅回调
          flushCallbacks()
        }
      }
      // 提供给 atom.read 使用的 get 函数，用于读取其他 atom
      const getter: Getter = <V>(a: Atom<V>) => {
        // 读取的是自己（自依赖的情况）
        if (isSelfAtom(atom, a)) {
          const aState = ensureAtomState(a)
          // 自己还没有初始化
          if (!isAtomStateInitialized(aState)) {
            // 如果是 primitive atom，有 init 值
            if (hasInitialValue(a)) {
              // 用 init 填充 AtomState.v，并且递增版本号
              setAtomStateValueOrPromise(a, a.init, ensureAtomState!)
            } else {
              // 对于没有 init 的只读派生 atom，不允许直接读自己
              throw new Error('no atom init')
            }
          }
          // 自己的值直接从 aState 里取出来返回
          return returnAtomValue(aState)
        }
        // 读取其他依赖 atom，会递归执行同样的逻辑
        const aState = readAtomState(a)
        try {
          // 取出依赖的值（或抛出错误/Promise）
          return returnAtomValue(aState)
        } finally {
          // 在 finally 里记录依赖：依赖哪个 atom，以及依赖时它的版本号
          atomState.d.set(a, aState.n)
          // 如果当前 atom 的值是 pending Promise
          if (isPendingPromise(atomState.v)) {
            addPendingPromiseToDependency(atom, atomState.v, aState)
            // 把这个依赖关系记录到依赖方，用于 Promise 完成后失效传播
          }
          // 在依赖的 Mounted.t 里记录反向依赖：谁依赖了我
          mountedMap.get(a)?.t.add(atom)
          // 如果已经脱离同步阶段（说明 read 返回了 Promise）
          if (!isSync) {
            // 在 Promise 完成后再挂载依赖 + 重算 + 刷新回调
            mountDependenciesIfAsync()
          }
        }
      }
      // 用于中断本次 read 对应的异步请求（如 fetch）
      let controller: AbortController | undefined
      let setSelf: ((...args: unknown[]) => unknown) | undefined
      // 提供给 atom.read 的第二个参数（signal + setSelf）
      const options = {
        get signal() {
          if (!controller) {
            controller = new AbortController()
          }
          // 允许在 read 里把 signal 传给 fetch 等可中断操作
          return controller.signal
        },
        get setSelf() {
          if (
            import.meta.env?.MODE !== 'production' &&
            !isActuallyWritableAtom(atom)
          ) {
            console.warn('setSelf function cannot be used with read-only atom')
          }
          if (!setSelf && isActuallyWritableAtom(atom)) {
            // 只在异步阶段允许调用 setSelf
            setSelf = (...args) => {
              if (import.meta.env?.MODE !== 'production' && isSync) {
                console.warn('setSelf function cannot be called in sync')
              }
              if (!isSync) {
                try {
                  // 内部直接走 writeAtomState，等同于在 read 里调用 setAtom
                  return writeAtomState(atom, ...args)
                } finally {
                  // 写完之后也要重算依赖
                  recomputeInvalidatedAtoms()
                  // 并且刷新订阅回调
                  flushCallbacks()
                }
              }
            }
          }
          return setSelf
        },
      }
      // 记录之前的版本号，用于判断本次 read 是否真的让值发生变化
      const prevEpochNumber = atomState.n
      try {
        // 调用真正的 atom.read（用户定义或默认实现）
        const valueOrPromise = atomRead(atom, getter, options as never)
        // 把返回的值或 Promise 写入 AtomState.v，并递增版本号
        setAtomStateValueOrPromise(atom, valueOrPromise, ensureAtomState!)
        if (isPromiseLike(valueOrPromise)) {
          // 如果是 Promise，注册 abort 处理器，支持中断
          registerAbortHandler(valueOrPromise, () => controller?.abort())
          valueOrPromise.then(
            // Promise resolve/reject 后挂载依赖并重算
            mountDependenciesIfAsync,
            mountDependenciesIfAsync,
          )
        }
        // 返回更新后的 AtomState
        return atomState
      } catch (error) {
        // read 抛错时的处理：清掉旧值，记录错误，并递增版本号
        delete atomState.v
        atomState.e = error
        // 版本号仍然要递增（表示状态发生了变化）
        ++atomState.n
        return atomState
      } finally {
        // 标记同步阶段结束，后续再触发 mountDependenciesIfAsync
        isSync = false
        if (
          // 如果这一轮 read 确实让版本号发生了变化
          prevEpochNumber !== atomState.n &&
          // 并且 invalidatedAtoms 里记录的还是旧版本
          invalidatedAtoms.get(atom) === prevEpochNumber
        ) {
          // 更新 invalidatedAtoms 中的版本号
          invalidatedAtoms.set(atom, atomState.n)
          // 把当前 atom 放入 changedAtoms，后续 flushCallbacks 会通知订阅者
          changedAtoms.add(atom)
          // 调试用 hook：有 atom 被标记为 changed
          storeHooks.c?.(atom)
        }
      }
    })
  ```   
  **核心逻辑总结**：readAtomState 做的事就是——能用缓存就用缓存；不能用就调用 atom.read 重算，并在过程中自动收集依赖关系、处理 Promise、更新 AtomState 和版本号，再把“我变了”这件事登记到 invalidatedAtoms / changedAtoms，交给后面的依赖重算和订阅通知机制去完成整张图的更新。


### 2. `store.set`：

  ```typescript
  const writeAtomState =
    buildingBlockFunctions[5] ||
    ((atom, ...args) => {
      // 标记当前是否仍在同步阶段（调用 atom.write 的这一轮）
      let isSync = true
      // 提供给 atom.write 使用的 get 函数，用于读取其他 atom
      const getter: Getter = <V>(a: Atom<V>) =>
        returnAtomValue(readAtomState(a))
      // 提供给 atom.write 使用的 set 函数，用于写入自身或其他 atom
      const setter: Setter = <V, As extends unknown[], R>(
        a: WritableAtom<V, As, R>,
        ...args: As
      ) => {
        const aState = ensureAtomState(a)
        try {
          // 写的是当前这个 atom 自己
          if (isSelfAtom(atom, a)) {
            // 必须要有 init（即 primitive / 有初始值的可写 atom），否则视为不可写
            if (!hasInitialValue(a)) {
              // NOTE technically possible but restricted as it may cause bugs
              throw new Error('atom not writable')
            }
            // 记录写入前的版本号，用于判断这次写入是否真的让值变化
            const prevEpochNumber = aState.n
            // 这里简单起见，只取第一个参数作为新值（更复杂的情况在 atom.write 里处理）
            const v = args[0] as V
            // 把新值（或 Promise）写入 AtomState.v，并递增版本号
            setAtomStateValueOrPromise(a, v, ensureAtomState!)
            // 根据最新依赖挂载子依赖（对该 atom 的依赖图做更新）
            mountDependencies(a)
            // 如果版本号发生变化，说明值真的变了
            if (prevEpochNumber !== aState.n) {
              // 标记这个 atom 已改变，后续 flushCallbacks 会通知订阅者
              changedAtoms.add(a)
              storeHooks.c?.(a)
              // 把所有依赖当前 atom 的派生 atom 标记为失效，等待重算
              invalidateDependents(a)
            }
            // 写自身时，约定返回值为 undefined
            return undefined as R
          } else {
            // 写的是其他 atom：递归调用 writeAtomState，走同一套流程
            return writeAtomState(a, ...args)
          }
        } finally {
          // 对于异步写入（例如在 setSelf 里、或 onMount 里异步调用 setAtom）
          // 写完之后需要立刻触发依赖重算和订阅回调
          if (!isSync) {
            recomputeInvalidatedAtoms()
            flushCallbacks()
          }
        }
      }

      try {
        // 真正执行用户定义的 atom.write(get, set, ...args)
        // 内部可以任意调用 getter / setter 来读写其他 atom
        return atomWrite(atom, getter, setter, ...args)
      } finally {
        // 标记同步阶段结束，之后 setter 里的写入会走「异步写」分支
        isSync = false
      }
    })
  ```

  **核心逻辑总结**：  
  `store.set(atom, ...args)` 会调用 `writeAtomState(atom, ...args)`，它做的事可以概括为：  
  - 把 `atom.write` 包装成一层：给它注入统一的 `getter` / `setter` 实现。  
  - `getter` 内部直接走 `readAtomState`，保证读出来的都是经过缓存/依赖追踪之后的值。  
  - `setter` 写入自身时，通过 `setAtomStateValueOrPromise` 更新 `AtomState.v/n`，然后 `mountDependencies`、`changedAtoms.add`、`invalidateDependents`，把“我变了”这件事记到依赖图里；写入别的 atom 时递归 `writeAtomState`。  
  - 对于异步写入，`setter` 在异步分支里会自动调用 `recomputeInvalidatedAtoms + flushCallbacks`，完成失效重算和订阅者通知。  
  - 这样一来，组件侧只需要拿到 `useSetAtom` 返回的函数去 `setAtom(...)`，其背后所有版本号维护、依赖失效、拓扑排序重算、订阅通知都由 `writeAtomState` + `recomputeInvalidatedAtoms` + `flushCallbacks` 这条链路统一负责。

