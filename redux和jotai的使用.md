## react中常用状态管理工具redux和jotai的一些使用和区别
1. redux通常结合react-redux，@reduxjs/toolkit一起在react中使用
   ```javascript
   // 通常api
   import { Provider, useSelect, useDispatch } from 'react-redux' // 将子组件通过Provider包裹，传入store，useSelect((store) => store.counter),
   import { createStore } from 'redux'
   import { createSlice，configureStore } from '@reduxjs/toolkit' // 创建分片store和合成分片store
   // Provider用于包裹组件
   // useSelect 在子组件内订阅某个状态。 const state = useSelect((store) => store.counter)
   // useDispatch 在子组件内创建dispatch用于更新state。 const dispatch = useDispatch(); dispatch(action)
   // createStore 原始创建store的方法。 const store = createStore(reducer)
   // createSlice 创建分片store。const storeSlice = createSlice({name: 'name', initialState: initialState, reducers: { add: () => initialState.val ++} })
   // 一般导出处storeSlice的reducer和actions。storeSlice.reducer，storeSlice.actions。
   // configureStore 用于合并多个store。const store = configureStore({reducer: { name: storeSlice.reducer } })
   ```
2. jotai常用api
   ```javascript
   import { atom, useAtom, useAtomValue, useSetAtom } from 'jotai'
   import { atomWithStorage, atomWithReset, atomWithDefault } from 'jotai/utils'
   const state = atom(1) // 可以有不同的方式创建atom，1:默认值，2:提供get函数，3:提供默认值和set函数。
   const memoState = atom((get) => get(state) + 1) // 依赖state，类似Vue计算属性, 提供get函数的atom不用用于setAtom
   const state2 = atom(1, (get, set, news) => { const gets = get(state); set(state2, gets + news) }) // 这种可以通过setAtom设置状态的值
   // atomWithStorage 创建的atom可以关联到浏览器的storage中
   // atomWithReset 创建一个可以reset的atom
   // atomWithDefault 用于解决第二种方式创建atom时候不能set的问题， const atom = atomWithDefault((get) => get(state))
   // const [atom, setAtom] = useAtom(state) 类似react的useState的hooks
   // const val = useAtomValue(state) 只读
   // const setVal = useSetAtom(state) 只写
   ```
两者区别：
1. 两者的底层实现原理都是通过发布订阅模式实现的。
2. jotai它是一种原子化的思想，随处可创建。但是useAtom时候需要在react组件中
3. react-redux是基于Provide实现的，当Provide的store更新后会导致所有子组件的卸载和渲染，即便是这个组件没订阅这个状态，这回造成性能消耗
4. react-redux偏向于集中管理，创建store有一套规范的要求
使用考虑：
看个人喜好，jotai更灵活方便，redux更集中。
