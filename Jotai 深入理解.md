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


## 了解了这些基本数据结构，再来看 useAtomValue和useSetAtom做了些啥。useAtom不用讲，就是返回[useAtomValue(atom, options),useSetAtom(atom, options)]。

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

```javascript
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
  - writeAtomState的实现
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
                    // 例如：
                    // const deepAtom = atom(1)
                    // const valAtom = atom((get) => get(deepAtom)),这种派生的atom就是不可写的
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

  - recomputeInvalidatedAtoms的实现
```typescript
const recomputeInvalidatedAtoms =
    buildingBlockFunctions[2] ||
    (() => {
        // Step 1: 通过深度优先搜索构建拓扑排序列表（逆序）
        // 不检查循环依赖，简化算法
        // 存储拓扑排序的结果，从后往前遍历就是按依赖顺序（先依赖，后依赖者）
        const topSortedReversed: [atom: AnyAtom, atomState: AtomState][] = []
        // 正在访问中的 atom（在栈上但还没处理完）
        const visiting = new WeakSet<AnyAtom>()
        // 已访问并处理完的 atom
        const visited = new WeakSet<AnyAtom>()
        // 从 changedAtoms 开始遍历，这些是依赖图的"根节点"（没有入边的节点）
        const stack: AnyAtom[] = Array.from(changedAtoms)
        while (stack.length) {
            // 取栈顶的 atom（不弹出，用于深度优先搜索）
            const a = stack[stack.length - 1]!
            // 确保这个 atom 有对应的 AtomState
            const aState = ensureAtomState(a)
            // 如果已经处理完，直接出栈
            if (visited.has(a)) {
                stack.pop()
                continue
            }
            // 如果正在访问中（说明形成了环，但算法不检查），准备加入结果列表
            if (visiting.has(a)) {
                // 只有当 invalidatedAtoms 里记录的版本号和当前版本号一致时，才需要重算
                if (invalidatedAtoms.get(a) === aState.n) {
                    // 加入拓扑排序结果列表（逆序，后续从后往前遍历）
                    topSortedReversed.push([a, aState])
                } else if (
                    import.meta.env?.MODE !== 'production' &&
                    invalidatedAtoms.has(a)
                ) {
                    // 开发环境下检查：如果 invalidatedAtoms 里有记录但版本号不匹配，说明有 bug
                    throw new Error('[Bug] invalidated atom exists')
                }
                // 标记为已处理完
                visited.add(a)
                stack.pop()
                continue
            }
            // 标记为正在访问中
            visiting.add(a)
            // 把所有未访问的依赖者（反向依赖）压入栈
            for (const d of getMountedOrPendingDependents(a, aState, mountedMap)) {
                if (!visiting.has(d)) {
                    stack.push(d)
                }
            }
        }
        // Step 2: 按拓扑排序的逆序（即依赖顺序）重新计算所有受影响的 atom
        // 从后往前遍历，确保先计算依赖，再计算依赖者
        for (let i = topSortedReversed.length - 1; i >= 0; --i) {
            const [a, aState] = topSortedReversed[i]!
            // 检查是否有依赖发生了变化
            let hasChangedDeps = false
            // 遍历当前 atom 的所有依赖
            for (const dep of aState.d.keys()) {
                // 跳过自己（自依赖的情况），检查依赖是否在 changedAtoms 中
                if (dep !== a && changedAtoms.has(dep)) {
                    hasChangedDeps = true
                    break
                }
            }
            // 只有当依赖发生变化时，才重新计算
            if (hasChangedDeps) {
                // 重新计算 atom 的值（会更新 AtomState.v 和版本号）
                readAtomState(a)
                // 根据新的依赖关系更新挂载状态
                mountDependencies(a)
            }
            // 从 invalidatedAtoms 中删除，表示已处理完
            invalidatedAtoms.delete(a)
        }
    })
//flushCallbacks没什么好说的，就是把执行队列里的回调
```

  **`store.get` 核心逻辑流程总结**：  
  `store.get(atom)` 的完整流程是：`returnAtomValue(readAtomState(atom))`，核心在 `readAtomState`。  
  - **Step 1：缓存检查**  
    先通过 `ensureAtomState(atom)` 确保有 `AtomState`，然后尝试使用缓存：  
    - 如果 atom 已挂载且 `invalidatedAtoms` 记录的版本号 ≠ 当前版本号 → 说明已在依赖重算中更新过，直接返回缓存。  
    - 否则检查所有依赖的版本号是否都未变 → 如果都没变，返回缓存。  
  - **Step 2：重新计算**  
    缓存不可用时，清空依赖 Map，构造 `getter` 和 `options`，调用 `atom.read(getter, options)`：  
    - **`getter`**：读取其他 atom 时递归 `readAtomState`，在 `finally` 中记录依赖关系（`atomState.d.set(a, aState.n)`）、反向依赖（`mountedMap.get(a)?.t.add(atom)`）、Promise 依赖（`addPendingPromiseToDependency`）。  
    - **`options`**：提供 `signal`（用于中断异步请求）和 `setSelf`（异步阶段可写自身）。  
  - **Step 3：更新 AtomState**  
    把 `atom.read` 返回的值或 Promise 通过 `setAtomStateValueOrPromise` 写入 `AtomState.v`，递增版本号 `n`。如果是 Promise，注册 abort 处理器，并在 resolve/reject 后调用 `mountDependenciesIfAsync`。  
  - **Step 4：标记变化**  
    在 `finally` 中，如果版本号变化且 `invalidatedAtoms` 记录的是旧版本，则更新 `invalidatedAtoms`、加入 `changedAtoms`，供后续 `recomputeInvalidatedAtoms` 和 `flushCallbacks` 使用。  
  - **最终返回**：`returnAtomValue(atomState)` 从 `AtomState` 中取出值（或抛出错误/Promise）。  
  **一句话概括**：`store.get` 通过 `readAtomState` 实现智能缓存、自动依赖追踪、Promise 处理，并标记变化，为后续的依赖重算和订阅通知做准备。

### 3. `store.sub`：
  **核心逻辑**：sub的逻辑很简单，订阅 atom 的变化，当 atom 的值或依赖变化时，会触发回调。  
```typescript
const sub = (atom, listener) => {
    const mounted = mountAtom(atom)
    const listeners = mounted.l
    listeners.add(listener)
    flushCallbacks()
    return () => {
        listeners.delete(listener)
        unmountAtom(atom)
        flushCallbacks()
    }
}
``` 

## Jotai 的其他 React Hooks

除了 `useAtomValue` 和 `useSetAtom`，Jotai 还提供了以下 React Hooks：

1. **`useAtom`** - 同时返回值和 setter
   - 位置：`src/react/useAtom.ts`
   - 作用：`useAtomValue` + `useSetAtom` 的组合，返回 `[value, setAtom]`
   - 用法：`const [value, setValue] = useAtom(atom)`
   - 实现：内部调用 `useAtomValue` 和 `useSetAtom`

2. **`useAtomCallback`** - 创建临时 atom 执行回调
   - 位置：`src/react/utils/useAtomCallback.ts`
   - 作用：创建一个临时 atom，用于在回调中访问 `get` 和 `set`
   - 用法：`const callback = useAtomCallback((get, set, ...args) => { ... })`
   - 实现：使用 `useMemo` 创建临时 atom，然后返回 `useSetAtom` 的结果

3. **`useHydrateAtoms`** - SSR/SSG 初始化
   - 位置：`src/react/utils/useHydrateAtoms.ts`
   - 作用：在服务端渲染或静态生成时初始化 atom 的值
   - 用法：`useHydrateAtoms([[atom1, value1], [atom2, value2]])`
   - 实现：使用 `WeakMap` 记录已初始化的 atom，避免重复初始化

4. **`useResetAtom`** - 重置 atom 到初始值
   - 位置：`src/react/utils/useResetAtom.ts`
   - 作用：返回一个函数，用于将 atom 重置为初始值（使用 `RESET` 符号）
   - 用法：`const reset = useResetAtom(atom)`
   - 实现：内部使用 `useSetAtom`，返回一个调用 `setAtom(RESET)` 的函数

5. **`useReducerAtom`** - Reducer 模式（已废弃）
   - 位置：`src/react/utils/useReducerAtom.ts`
   - 作用：类似 React 的 `useReducer`，但基于 atom
   - 状态：已废弃，建议使用自定义 recipe 替代
   - 实现：内部使用 `useAtom`，然后用 `useCallback` 包装 reducer


### `useStore` Hook

`useStore` 是 Jotai 中用于获取当前 store 的 Hook，所有其他 Hooks（如 `useAtomValue`、`useSetAtom`）都依赖它来获取 store。

**源码**（`src/react/Provider.ts`）：
```typescript
export function useStore(options?: Options): Store {
    const store = useContext(StoreContext)
    return options?.store || store || getDefaultStore()
}
```

**作用**：
- 获取当前组件树中使用的 store
- 优先级：`options.store` > Context 中的 store > 默认 store（`getDefaultStore()`）
- 允许在组件中显式指定使用哪个 store

**使用场景**：
- 在组件中需要直接访问 store 时（如 `store.get(atom)`、`store.set(atom, value)`）
- 需要跨多个 store 操作时
- 需要访问 store 的底层 API 时

# 最后再说一些Jotai的高级使用

## createStore Provider
**作用**：createStore可以创建不同的store，而Provider则可以提供不同的store。通过这种方式，可以实现不同的组件树使用不同的store，从而实现状态的隔离。
**示例**：
```typescript
import { createStore, Provider, atom } from 'jotai';
const customStore = createStore();
const countAtom = atom(0);
const App = () => {
    return (
        <>
            // customStore是一个全新的store，与getDefaultStore()提供的store是不同的
            <Provider store={customStore}>
                <Counter />
            </Provider>
            <AnotherComponent />
        </>
    );
}
const Counter = () => {
    const [count, setCount] = useAtom(countAtom);
    return (
        <div>
            <p>{count}</p>
            <button onClick={() => setCount(count + 1)}>Increment</button>
        </div>
    );  
}
// 如果其他组件使用了countAtom，它们将使用getDefaultStore()提供的store，而不会影响customStore中的countAtom。
const AnotherComponent = () => {
    const [count, setCount] = useAtom(countAtom);
    useEffect(() => {
        // AnotherComponent使用的是默认的store，所以它与Counter使用的不是同一个store
        // 当Counter修改了countAtom的值时，AnotherComponent并没有响应，因为它使用的是默认的store。
        console.log('AnotherComponent count changed:', count);
    }, [count])
    return (
        <div>
            <p>{count}</p>
            <button onClick={() => setCount(count + 1)}>Increment</button>
        </div>
    );
}
// 如果想使AnotherComponent也响应Counter的修改，可以封装hooks。
const useLocalAtomValue = (localAtom, localStore) => {
    const [value, setValue] = useState(() => localStore.get(localAtom))
    useEffect(() => {
        // 订阅局部 store 的变化
        const unsubscribe = localStore.sub(localAtom, () => {
            setValue(localStore.get(localAtom))
        })
        return unsubscribe
    }, [localAtom, localStore])
    return value
}
const useSetLocalAtom = (localAtom, localStore) => {
    const setLocalAtom = useCallback(
        (value) => localStore.set(localAtom, value),
        [localAtom, localStore]
    )
    return setLocalAtom
}
const useLocalAtom = (localAtom, localStore) => {
    const value = useLocalAtomValue(localAtom, localStore)
    const setAtom = useSetLocalAtom(localAtom, localStore)
    return [value, setAtom]
}
// 然后在AnotherComponent中使用这个hook：
const AnotherComponent = () => {
    const [count, setCount] = useLocalAtom(countAtom, customStore);
    useEffect(() => {
        // 这样就能监听到customStore中的countAtom的变化了。
        console.log('AnotherComponent count changed:', count);
    }, [count])
    return (
        <div>
            <p>{count}</p>
            <button onClick={() => setCount(count + 1)}>Increment</button>
        </div>
    );
}
```

## atomWithStorage 创建持久化的状态

**作用**：atomWithStorage允许我们创建一种持久化的原子状态，比如你想持久保存登录用户信息等，如果不指定它的storage，默认就是存储在localStorage中。

**示例**：
```typescript
import { atomWithStorage, useAtom } from 'jotai';
// 创建一个持久化的用户信息atom，必须传入一个key，这个key会作为localStorage的key。
const userAtom = atomWithStorage('user', {});
// 使用这个atom，和一般的atom没区别
const [user, setUser] = useAtom(userAtom);
```
- 如果想自定义storage，可以传入一个自定义Storage对象，包含getItem,setItem, removeItem方法，比如你想有的情况使用localStorage，有的情况使用sessionStorage。
```typescript
import { atomWithStorage, useAtom } from 'jotai';
const adpStorage = (useSession?: boolean) => {
    const storage = useSession ? sessionStorage : localStorage;
    return {
        getItem: (key: string, initialValue: any) => {
            // 自定义逻辑
            storage.getItem(key)
        },
        setItem: (key, value) => {
            // 自定义逻辑
            storage.setItem(key, value)
        },
        removeItem: (key) => {
            // 自定义逻辑
            storage.removeItem(key)
        }
    }
}
const atomWithStorageAdaptive = (
    key,
    initialValue,
    options
) => {
    return atomWithStorage(
        key,
        initialValue,
        adpStorage(options?.useSession),
        options
    )
};
const userAtomWithCustomStorage = atomWithStorageAdaptive('user', {}, {useSession: true});
```
- 对于 atomWithStorage有一点需要特别说明，在创建时如果不指定options中的getOnInit为true的话，如果本次修改了原子的值，下次再访问的时候会导致依赖该原子的hooks执行两次。原因在于onMount阶段会去走storage中去获取值再setAtom一次。源码分析如下：
```typescript
export function atomWithStorage<Value>(
    key: string,
    initialValue: Value,
    storage:
        | SyncStorage<Value>
        | AsyncStorage<Value> = defaultStorage as SyncStorage<Value>,
    options?: { getOnInit?: boolean },
) {
    const getOnInit = options?.getOnInit
    // 创建一个基础的原子对象，用于存储状态。这里看到如果没有getOnInit用的是初始值，有才用最新的storage的值
    const baseAtom = atom(
        getOnInit
            ? (storage.getItem(key, initialValue) as Value | Promise<Value>)
            : initialValue,
    )
    if (import.meta.env?.MODE !== 'production') {
        baseAtom.debugPrivate = true
    }
    /*
        onMount: 执行时机是在组件挂载时，会执行一次setAtom，如果这个值和上次的值不一样，就会触发刷新回调。
        为什么会存在不一样呢？比如我第一次访问页面时候执行某些操作修改了atom的值，这个时候storage中的值也会被修改，当我下次再访问页面时候，如果getOnInit不存在，那么创建baseAtom的时候用的就是代码里面的初始值，在执行onMount的时候又会走storage去get一次，所以这次的值和初始值是不一样的，就会造成二次触发。
    */
    baseAtom.onMount = (setAtom) => {
        setAtom(storage.getItem(key, initialValue) as Value | Promise<Value>)
        let unsub: Unsubscribe | undefined
        if (storage.subscribe) {
            unsub = storage.subscribe(key, setAtom, initialValue)
        }
        return unsub
    }
    const anAtom = atom(
        (get) => get(baseAtom),
        (get, set, update: SetStateActionWithReset<Value | Promise<Value>>) => {
            const nextValue =
                typeof update === 'function'
                    ? (
                        update as (
                            prev: Value | Promise<Value>,
                        ) => Value | Promise<Value> | typeof RESET
                    )(get(baseAtom))
                    : update
            if (nextValue === RESET) {
                set(baseAtom, initialValue)
                return storage.removeItem(key)
            }
            if (isPromiseLike(nextValue)) {
                return nextValue.then((resolvedValue) => {
                    set(baseAtom, resolvedValue)
                    return storage.setItem(key, resolvedValue)
                })
            }
            set(baseAtom, nextValue)
            return storage.setItem(key, nextValue)
        },
    )
    return anAtom as never
}
```
