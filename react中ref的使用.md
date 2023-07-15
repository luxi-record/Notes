<center><font face="黑体" size=24 >React中ref的使用</font></center>

##### 	在react的hooks组件中，**ref**用于保存一个不变的值（ 因为函数组件每次更新后都会重新执行该函数组件，里面的hooks也会执行，此时ref的值和上一次的值是一样的）

 * **判断hooks组件是否初次渲染**（模拟componentDidUpdate）

   ``` javascript
   import { useRef, useEffect } from 'react'
   function Component () {
       const isFirstRender = useRef<boolean>(true) // 初始为true表示是第一次渲染
       useEffect(() => {
           isFirstRender.current = false
       }, []) 
       useEffect(() => {
           if(!isFirstRender.current) {
               // Do Update 这里的逻辑就和componentDidUpdate类似
           }
       })
   }
   // ref在ts中申明的接口： React.RefObject<T>
   ```

 * **保存DOM的引用**

   ```javascript
   // 函数组件
   import { useRef } from 'react'
   function Component () {
       const box = useRef<HTMLElement>()
       return (
           <div ref={box}>BOX</div>
       )
   }
   
   // 类组件
   import React from 'react'
   class Component extends React.Component {
       ref = React.createRef()
       render() {
           return (
           	<div ref={this.ref}>BOX</div>
           )
       }
   }
   ```

 * **ref全部转发或部分转发**

   ```javascript
   // 函数组件中ref转发
   import { forwardRef } from 'react'
   const Component = forwardRef((props, ref) => { // ref为Component组件接受的ref属性
       // 子组件逻辑
       return (
       	<div>
           	<span>first</span>
           	<input ref={ref} />
           </div>
       )
   })
   // 在父组件引用
   import { useRef, useEffect } from 'react' 
   function Parent () {
       const input = useRef()
       useEffect(() => {
   		console.log(input) // 此时打印的就是对Component组件里面input**全部**的引用
       }, [])
       return (
       	<div>
           	<Component ref={input} />
           </div>
       )
   }
   /****如果Component组件不想把input全部暴露出去可以使用useImperativeHandle****/
   import { forwardRef, useImperativeHandle, useRef } from 'react'
   const Component = forwardRef((props, ref) => {
       // 子组件逻辑
       const input = useRef()
       // useImperativeHandle接受三个参数，第一个是传入的ref， 第三个是一个可选参数依赖项
       // 第二个是一个函数返回一个任意对象，该对象就是透传的ref.current的值
       useImperativeHandle(ref, () => {
           // 其他逻辑
           return {
               click: () => { input.current.click() }
           }
       }, [])
       return (
       	<div>
           	<span>first</span>
           	<input ref={input} />
           </div>
       )
   })
   function Parent () {
       const ref = useRef()
       useEffect(() => {
   		console.log(ref) // 此时打印的就是对Component组件里面input**部分**的引用
           console.log(ref.current) // 此时打印就是useImperativeHandle第二个参数函数返回值
       }, [])
       return (
       	<div>
           	<Component ref={ref} />
           </div>
       )
   }
   ```

   