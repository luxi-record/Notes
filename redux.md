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
//     replaceReducer,
//     [$$observable]: observable
// }
```
