<center><font face="黑体" size=24 >xhr和fetch的使用于区别</font></center>

##### 通常我们进行网络请求时候最常用的就是xhr、fetch和axios，xhr和fetch是原生方法，axios是对他们的封装同时适用于node环境。

* **XMLHttpRequest常用属性与方法**

  ```javascript
  const xhr = new XMLHttpRexquest()
  xhr.onreadystatechange = () => {} // 当 readyState 属性发生变化时调用
  xhr.readyState // 代表xhr的状态0-4，4表示完成
  xhr.status // 代表http状态， 200表示成功
  xhr.response // **整个**响应体
  xhr.responseText，responseXML，responseURL // 表示该请求的响应
  xhr.responseType // 响应类型 blob，text，json，arraybuffer，document
  xhr.timeout = 5000 // 设置超时时间，超时后请求自动终止调用ontimeout事件
  xhr.upload // 上传相关对象，可进行进度监听xhr.upload.onprogress = (e) => console.log(e.loaded/ e.total)
  xhr.withCredentials = false // 跨域请求是否携带cookie
  xhr.abort() // 终止当前请求
  xhr.getAllResponseHeaders()，getResponseHeader(key) // 获取响应头
  xhr.setRequestHeader(key, value) // 设置请求头，必须在open() 之后、send() 之前调用
  xhr.onpreogress = () => {}	// 请求周期性触发
  xhr.onload = () => {}	// 请求结束触发
  xhr.onerror = () => {}	// 请求出错触发
  xhr.ontimeout = () => {} // 超时后的回调（回调里面不能访问status属性，这时readyState已变成4）
  xhr.open(type, url, isAsync?: boolean,user?: string, password?: string) // 初始化请求,请求类型必须是大写,user, password用于认证之类操作默认为null
  xhr.send() // 发送数据，不需要发送数据必须传入null**一切注册的回调或者设置属性尽量都要放在open和send之前**
  // 调用open（）只是代表启动一个请求以备发送，调用send（）时候才会发起请求
  // XMLHttpRexquest继承了EventTarget可以通过addEventListener添加事件监
  ```

* **fetch的使用**（fetch是基于Promise的封装）

  ```javascript
  const response = await fetch(url, option?: Option | Request) // 不传option默认是get请求，Request对象属性和Option类似
  // 只列举常用的属性配置 
  type Option = {
      method?: string, // 请求方法，GET，POST等
      headers?: Headers | Record<string, string>, // 一个请求头设置，可以是一个对象也可以是一个new Headers实例
      body?: Blob | FormData | JSON, // 请求数据，GET和HEAD请求没有body
      mode?: string, // 请求模式cors、no-cors 或者 same-origin，主要用于跨域这些
      credentials?: string, // 携带cookie 
      cache?: string, // 缓存设置no-store、 reload 、 no-cache 等   
      redirect?: string, // 重定向设置     
      signal?: AbortSignal, // 终端设置，类似xhr的abort方法，const abortControl = new AbortController()；option.signal = abortControl.signal, 调用abortControl.abort()就可以终止请求      
  }
  
  // response是一个Response实例，常用的属性和方法有：
  response.ok // 表示请求是否成功
  response.headers // 返回一个响应头对象
  response.status // 状态码
  response.body // 响应体，是一个ReadableStream，而不是像xhr是一个整体数据
  response.blob() // 返回一个Promise，把响应转化为一个blob对象
  response.json() // 返回一个Promise，把响应转化为一个对象
  response.formData() // 主要用在 Service Worker 里面，拦截用户提交的表单，修改某些数据以后，再提交给服务器
  response.arrayBuffer() // 主要用于获取流媒体文件
  // fetch只有网络错误的时候才会走到Promise的catch里面去，即使状态码是500也不会reject
  // 所以当你的请求受网络影响的时候而你的程序还需要继续运行的时候，最好加上catch
  const response = await fetch(url, option).then(res => res).catch(() => return '网络错误')
  ```

* **两者的区别**

  1. fetch原生没有中止方法,想要实现终止需要借助其他方式

     ```javascript
     // xhr实现终止
     const xhr = new XMLHttpRexquest()
     xhr.abort()
     // fetch实现终止
     const abortControl = new AbortController()
     const option = { ...others, signal: abortControl.signal }
     const response = fetch(url, optin)
     abortControl.abort() // 借助AbortController实现终止
     ```

  2. fetch没有超时设置，需要自己添加

     ```javascript
     // xhr中超时
     const xhr = new XMLHttpRexquest()
     xhr.timeout = 5000 // 5s
     xhr.ontimeout = () => consloe.log('超时')
     // fetch实现超时，需要自行封装fetch
     function withTimeoutFetch (url, fetchOption, timeout: number) {
         const abortControl = new AbortController()
     	const option = { ...fetchOption, signal: abortControl.signal }
     	const fet = fetch(url, optin) 
         const timeout = new Promise((resolve) => {
             setTimeout(() => {
                 abortControl.abort()
                 resolve('超时')
             }, timeout)
         })
         return Promise.race([fet, timeout])
     }
     // or
     function withTimeoutFetch (url, fetchOption, timeout: number) {
         const abortControl = new AbortController()
     	const option = { ...fetchOption, signal: abortControl.signal }
     	const fet = fetch(url, optin) 
         return new Promise((resolve) => {
             fet.then((response) => {
                 resolve(response)
             })
             setTimeout(() => {
                 abortControl.abort()
                 resolve('超时')
             },timeout)
         })
     }
     ```

  3. fetch返回的响应是一个流在body里面，而xhr返回的是一个整体数据，当在下载大文件时候fetch比xhr更好

  4. fetch只有对网络错误才会报错，对404或者500这类错误是不会报错，需要开发中自行定义

  5. fetch没有上传进度，下载进度需要自行编码
  
     ```javascript
     // xhr上传下载进度
     const xhr = new XMLHttpRexquest()
     xhr.upload.onprogress = (e) => console.log(e.loaded/ e.total) // 上传进度
     xhr.onprogress = (e) => { /* do something */ } // 下载进度
     // fetch暂时没有想到怎么实现上传进度，下载进度可以模拟
     const { body, headers } = await fetch(url, option)
     const contentLength = headers.get('Content-Length')
     const reader = body.getReader()
     let loadLength = 0, rotate
     while(true) {
         const { value, done } = await reader.read()
         if(!done) {
             loadLength += value.length
             rotate = loadLength / contentLength
         } else{
             rotate = 1
             break
         }
     }
     ```
  
     **总结：xhr功能更全更具体，兼容性也更好，但是使用起来很麻烦各种回调方法。fetch是基于Promise的封装使用起来更简便对于大文件数据的下载fetch更适合，但是功能没xhr全面兼容性也没xhr强。**
