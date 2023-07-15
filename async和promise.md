<center><font face="黑体" size=24 >Promise和Async的简单介绍</font></center>

##### Promise通常用于一些异步处理属于微任务，解决一些多层回调和模拟睡眠等待等。Promise的状态一旦确定就是不可逆的。

* Promise的一些方法以及实例（**Promise.resolve()**,**Promise.reject()**,**Promise.prototype.finally()**）

  ```javascript
  const a = Promise.resolve(1) // 返回一个成功（fulfilled）的Promise，值为1
  const b = Promise.reject('failed') // 返回一个失败（rejected）的Promise，值为failed
  new Promise().finally(() => {
      // return something
  }) // 无论失败还是成功都会走finally这个回调，回调函数不接受任何参数
  ```

* **Promise.all([])** *等待所有，一有失败就立马变为失败状态*

  ```javascript
  // Promise.all接受一个iterable类型（并非一定是数组），返回一个Promise。
  // Promise.all当接受的可迭代对象都是resolve时那个该Promise就是fulfilled状态，如果其中一个是rejected，则为rejected
  const a = Promise.resolve(1)
  const b = new Promise((resolve, reject) => setTimeout(resolve(2), 100))
  const c = new Promise((resolve, reject) => setTimeout(reject(3), 100))
  const d = Promise.reject(4)
  const f = 5 // 常数也相当于一个fulfilled的状态
  Promise.all([a,b,f]).then((res) => console.log(res)) // 100ms后输出[1,2, 5]
  Promise.all([a,c]).then((res) => console.log(res)).catch((rej) => console.log(rej)) // 100ms后输出3
  ```

* **Promise.race([])** *谁跑的快，状态就取决于它无论成功还是失败*

  ```javascript
  // Promise.race和Promise.all类似，接受一个可迭代对象，返回一个Promise。
  // Promise.race不同点在于，数组里最先完成的Promise的状态就为Promise.race的状态
  const a = Promise.resolve(1)
  const b = new Promise((resolve, reject) => setTimeout(resolve(2), 100))
  const c = new Promise((resolve, reject) => setTimeout(reject(3), 100))
  const d = Promise.reject(4)
  const f = 5
  Promise.race([a,b]).then((res) => console.log(res)) // 直接输出1
  Promise.race([a,d]).then((res) => console.log(res)) // 直接输出1,即便d是立即rejected，但是a比d先运行
  Promise.race([c,f]).then((res) => console.log(res)).catch((res) => console.log(res)) // catch里输出3
  ```

* **Promise.allSettled([])** *等待所有执行完成，resolve所有结果*

  ```javascript
  // Promise.allSettled和Promise.all类似，接受一个可迭代对象，返回一个Promise（只有fulfilled状态）。
  // Promise.allSettled不同点在于，他会等数组里所有promise都完成，无论数组里的promise是什么状态
  const a = Promise.resolve(1)
  const b = new Promise((resolve, reject) => setTimeout(resolve(2), 100))
  const c = new Promise((resolve, reject) => setTimeout(reject(3), 100))
  const d = Promise.reject(4)
  const f = 5
  Promise.allSettled([a,b]).then((res) => console.log(res)) 
  //输出[{status: "fulfilled", value: 1},{status: "fulfilled", value: 2}]
  Promise.allSettled([a,c]).then((res) => console.log(res)) 
  //输出[{status: "fulfilled", value: 1},{status: "rejected", reason: 3}]
  ```

* **Promise.any([])** *等待一个完成的Promise，如果都失败则为失败*

  ```javascript
  // Promise.any也是接受一个可迭代对象，返回一个Promise。re
  /** Promise.any等待数组里为fulfilled的promise，如果数组里都是rejected状态则，reject('AggregateError: All      promises were rejected') **/
  // 数组里只要有一个 promise 状态为完成，那么就会提前结束，而不会继续等待其他的 promise 执行
  const a = Promise.resolve(1)
  const b = new Promise((resolve, reject) => setTimeout(resolve(2), 100))
  const c = new Promise((resolve, reject) => setTimeout(resolve(3), 200))
  const d = Promise.reject(4)
  const f = 5
  Promise.any([a,b]).then((res) => console.log(res)) // 输出1
  Promise.any([c,b]).then((res) => console.log(res)) // 输出2
  Promise.any([d, f]).then((res) => fullconsole.log(res)) // 输出4,因为有一个是fulfilled状态
  Promise.any([d]).then((res) => console.log(res)).catch((res) => console.log(res))
  // catch输出AggregateError: All promises were rejected
  ```

* 在Promise的**链式调用**中只要有一个promise为**rejected**那么就会立马走到**catch的回调**中

  ```javascript
  new Promise((res, rej) => res(1))
      .then((res, rej) => rej(2))
      .then((res,rej) => res(3))
      .catch((rej) => console.log(rej)) // 打印2
  ```

* Promise.prototype.then()可以**接受两个回调**，一个是状态为**fulfilled**的回调，一个是状态为**rejected**的回调

  ```javascript
  new Promise(() => {}).then((res) => {}, (rej) => {}) //第二个回调类似于.catch
  ```
  

##### Async函数是基于Promise的一个语法糖，可以解决链式调用代码繁琐的麻烦，异步函数同步调用，通常需要搭配await使用，async函数返回的也是一个Promise，如果await的promise都是成功的则promise的值为函数return返回的值，如果有一个await的promise是失败，则async为失败，值和那个失败的promise一样。

* **await**相当于Promise的**回调**语法糖

  ```javascript
  const a = new Promise((resolve, reject) => setTimeout(resolve(1), 2000))
  async function example () {
      console.log('start')
      const res = await a
      console.log('after await')
      console.log(res)
  }
  function log () {
      console.log(2)
  }
  example()
  log()
  // 打印顺序 start, 2，两秒后打印after await, 1
  // 在async中await会等待后面的promise执行完成后才会去执行下面的同步代码（console.log(res)）
  // await不会阻塞主线程里面的同步代码，只会影响async函数里面的同步代码的运行
  // await默认是类似promise的.then回调 const result = await a ===》 a.then((res) => result = res)
  // 如果await后面跟的是一个rejected状态的promise或者不知道状态是否会为rejected，最好后面加上catch。
  // 如果a是一个rejected的promise。则async返回的是一个rejected的promise，并且rejected的值为a失败的值，await下面的代码不会执行。
  /* const res = await a.then((res) => res).catch((rej) => { // do something and return a value } )
   * 这样无论a的状态为什么res都会有值，如果a为rejected则为catch回调返回的值，如果是fulfilled则为then回调返回的值
   **/
  ```

* **async**函数返回也是一个Promise，promise值取决于函数**return的值**或者有await后跟的**promise是失败状态**的值

  ```javascript
  const a = new Promise((resolve, reject) => setTimeout(resolve(1), 2000))
  async function example () {
      console.log('start')
      const res = await a
      console.log('after await')
      console.log(res)
      return 'example'
  }
  const result = example()
  result.then((res) => {
      console.log(res) // 打印‘example’
  })
  // 如果a是一个rejected状态const a = new Promise((resolve, reject) => setTimeout(reject(1), 2000))
  const result = example() // result就是一个rejected的promise ===》 
  result.then((res) => {
      console.log(res) // 不打印
  }).catch((rej) => {
      console.log(rej) // 打印1（a的rejected状态的值）
  })
  ```

* **async**直观理解

  ```javascript
  const a = new Promise((rej) => setTimeout(rej(1), 100))
  async function example () {
      return 2
  }
  
  function example () {
      return Promise.resolve(2)
  }
  // 加上await
  async function example () {
      await a
      return 2
  }
  
  function example () {
      return new Promise((res, rej) => a.then((result) => res(2)))
  }
  ```

  