<center><font face="黑体" size=24 >js中的函数、函数调用栈、单线程</font></center>

##### 浏览器是一个多线程，主要包含GUI渲染线程、js引擎线程、定时器线程、网络异步请求线程和事件触发线程，其中js线程和渲染线程是互斥的关系。

* **函数的定义和类型**

  1. **函数声明**，该方式**存在变量提升**类似var

     ```javas
     function fn (params) {
         console.log(params)
     }
     例子：
     a(2)
     var a = 1
     function a (params){                       
         consloe.log(params)
     }
     console.log(a) 
     // 这里会先打印2再打印1，因为function函数声明存在变量提升比var还要提前，会把声明放在最前面

  2. **匿名函数**，该方式不存在变量提升

     ```javascript
     const fn = function () => {}
     const fn2 = () => {}
     ```

  3. **箭头函数** 

     * 箭头函数没有自己的this, 在箭头函数里面调用this指向的是它的上一层作用域

       ```javascript
       const fn = () => {
           console.log(this)
       }
       fn() // 打印window，因为这里fn定义在最外层，它的上一层就是全局对象window
       const object = {
           a: function() {
               const that = this
               const arr = []
               const fn = () => {
                   console.log(this)
                   this === that // true
               }
               fn()
           },
           b: [],
       }
       object.a() // 这里打印的this就会指向object这个对象
       ```

     * 箭头函数里面this的指向在定义的时候就已经绑定了，后面通过bind， apply， call也无法改变其指向

       ```javascript
       const fn = () => { console.log(this) }
       const object = { a: 123 }
       fn.call(object) // 这里打印的是window，不会打印object
       ```

     * 箭头函数不能当作构造函数因为没有constructor，箭头函数没有arguments

       ```javascript
       const Class = () => { console.log(arguments) }
       new Class() // 报错 Class is not a constructor
       Class() // 报错 arguments is not defined
       ```
  
  4. **函数参数传递** *个人理解复制值传递，对于**基本数据类型复制**的就是存在栈内存中的**值本身**，对于**复杂数据类型复制**的就是存在栈内存中的**指针***
  
     js中数据类型分为两大类基本数据类型（**number, string, null, undefined, boolean**）和复杂数据类型。一个值存在于栈内存一个存在于堆内存，对于这两大数据类型当作参数传递给函数的时候：
  
     * 基本数据类型
  
       ```javascript
       const a = 1
       function fn (params) {
           params = 2
           console.log(params)
           console.log(a)
       }
       fn(a) // 这里分别打印 2和1
       // 如果执行fn()不传递参数，不会报错，因为函数会默认创建一个名为params的局部变量
       // 类似
       function fn (params) {
           var params = params
           params = 2
           console.log(params)
       }
       ```
  
     * 复杂数据类型
  
       ```javascript
       const a = {
           k1: 1,
           k2: 2
       }
       function fn (params) {
           params.k3 = 3
           console.log(params)
           console.log(a)
       }
       fn(a) 
       /* 
       这里会打印两个{ k1: 1, k2: 2, k3: 3 }，因为复制的是a指向对象{k1:1,k2:2}的指针，所以params也是一个指向对象{k1:1,k2:2}的指针，由于他们指向同一个值，所以params修改以后，a也会跟着变化
       */
       ```
  
* **函数调用栈**

  js的函数调用是一个压栈和出栈的过程，调用栈的大小是有限的，当达到限额时候会报错栈溢出。[js词法分析和语法分析]('https://juejin.cn/post/6844904089449414670')

  ```javascript
  function fn (){
      console.log(1)
      sayHi()
  }
  function sayHi(){
      console.log('hi')
  }
  fn()
  // 当我们在执行这一段代码的时候，js引擎会把代码解析成AST树结构，创建一个全局执行上下文
  // 当执行fn函数的时候会创建一个fn的执行上下文并压入执行栈
  // 在fn函数中执行到sayHi函数时候会为sayHi创建一个执行上下文并压入执行栈
  // 当sayHi执行完成以后会把sayHi的执行上下文弹出，并垃圾清除其创建的变量以及内存（闭包除外）
  // sayHi执行完成并弹出后继续执行fn中sayHi下面的代码，直到执行完成和sayHi一样弹出执行栈
  ```

* **单线程**

  js的执行是单线程模式（一个原因是防止多个线程操作同一DOM导致浏览器不知道怎么渲染），当把当前同步执行栈任务执行完成以后，回去检测异步执行栈是否有任务等待执行，如果有取出执行。

  当JS引擎执行异步函数的时候，会将异步任务放在事件触发线程中，当对应的异步任务符合触发条件被触发时，事件触发线程会把异步任务添加到JS引擎中的主线程的队尾，等待执行。

  同步任务 ==> 微任务（promise.then, MutationObserver，process.nextTick等） ==> 宏任务（定时器，DOM事件，网络请求，requestAnimationFrame等）

  执行流程图：

  ![图片](/JSExecutionFlowchart.png)

