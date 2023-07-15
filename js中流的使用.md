<center><font size=24>JavaScript中流的使用</font></center>

##### 在浏览器中流分为ReadableStream，WriteableStream，TransformStream，流主要的作用在于数据传输时候减少对内存的使用，在一些超大数据传输时候特别有(*本文主要讲解通过流对超大文件下载*)。

* **流的一些小知识**（[详细介绍]('https://developer.mozilla.org/zh-CN/docs/Web/API/Streams_API')）

  1. ReadableStream,WriteableStream

     ```javascript
     // ReadableStream，这里只列举了常用的option和strategy和一些方法属性
     type Optin = {
         start?: (controller) => void, // 初始化会立即调用，可用作数据通道绑定
         pull?: (controller) => void, // 当流的内部队列不满时，会重复调用这个方法，直到队列补满
         cancel?: (reason) => void, // 流将被取消时调用
     }
     type Strategy = {
         highWaterMark?: number, // 数据块最大数
         size?: (chunk) => void
     }
     const readStream = new ReadableStream(option?: Optin, strategy: Strategy)
     const isLocked = readStream.locked /* 表示当前流是否锁定到reader,当一个流被锁就无法调用它的一些方法（会导致流被锁定的方法），只有将流解锁后才能调用， 例如：const reader = readStream.getReader()；再去调用readStream.tee()会报错，这时需要调用reader.releaseLock()解锁后才能调用tee方法 */
     const reader = readStream.getReader() // 返回一个读取当前流数据的句柄，可用于读取数据，此时这个流的locked为true
     const promise = readStream.pipeTo(writeStream，option?:any) // 返回一个Promise，把可读流数据传输给指定可写流
     const arr = readStream.tee() //对当前流进行拷贝，返回包含两个流的数组
     
     // WriteableStream
     type Optin = {
         start?: (controller) => void, // 初始化会立即调用，可用作数据通道绑定
         write?: (chunk,controller) => void, // 当数据被写入时调用 writeStream.getWriter().write()
         close?: (controller) => void, // 当完成写入时调用 writeStream.getWriter().colse()
         abort?: (reason) => void, // 立即关闭流并且将其移至错误状态调用，类似close
     }
     type Strategy = {
         highWaterMark?: number, // 数据块最大数
         size?: (chunk) => void
     }
     const writeStream = new WriteableStream(option?: Optin, strategy: Strategy)
     const isLocked = writeStream.locked // 原理和可读流一样
     writeStream.abort(reason)  // 中止流，不能再向流写入数据会报错
     const promise = writeStream.close() // 表示流将关闭 
     const writer = writeStream.getWriter() // 获取写入数据句柄 
     ```

  2. TransformStream

     ```javascript
     type Option = {
         start?: (controller) => void, // 初始化会立即调用
     	transform?: (chunk, controller) => void, // 当可写端写入转化数据块时调用
     	flush?: (controller) => void // 当所有写入可写端的分块成功转换后被调用也就是close时，并且可写端将会关闭
     }
     // Strategy和可读可写流一样
     const transformStream = new TransformStream(option?: Option, writableStrategy, readableStrategy)
     const readStream = transformStream.readable // 返回一个可读流
     const writeStream = transformStream.writable   // 返回一个可写流         
     // TransformStream会把可写端写入的数据通过调用transform（程序员自定义转化方法）转化后会把数据流入可读端           
     ```

* **常用的文件下载方法**

  通常我们在下载文件时候最常用的方法就是后端返回一个二进制文件数据，前端通过把数据转化成链接进行下载，或者后端直接返回一个链接前端进行下载。

  ```javascript
  // 通过创建a标签进行下载，ts简单的实现（name是文件名， data是一个下载地址（base64或后端返回地址）或者是Blob数据）
  function downloadFile(name: string, data: string | Blob) {
      const isBlob = Object.prototype.toString.call(data) === '[object Blob]'
      const url = isBlob ? URL.createObjectURL(data) : data
      const a = document.createElement('a')
      a.download = name
      a.href = url
      a.click()
      URL.revokeObjectURL(url)
  }
  // 原理就是当后端返回二进制数据时候，我们通过URL.createObjectURL()把blob数据转化成对象url
  // 我们再通过a标签或者window.open打开这个url，浏览器会默认去下载这个文件
  // 因为URL.createObjectURL()创建的url的响应头里面的描述信息默认是让浏览器下载
  ```

​		无论是通过**a标签**还是**window.open**进行下载原理都一样，对于小文件很适用，但是对于超大文件（文件大小超过GB）就会**存在		内存问题**，你会发现这时内存大小和文件大小成正比，这时候这种方式就不再适合，就需要用到流进行下载。

* **结合Service Worker实现流的下载**(直接上代码，原理在后面)

  ```javascript
  // ts版本简单实现， 引入web-streams-polyfill/ponyfill是由于一些浏览器对流的实现不完全会造成一些函数无法使用
  import { WritableStream as PofWritableStream, TransformStream as PofTransformStream } from 'web-streams-polyfill/ponyfill'
  
  class StreamSaver {
      private WritableStream: any
      private TransformStream: any
      private supportsTransferable: boolean = false
      useBlob: boolean = false
      constructor() { // 检测一些流的api是否可用
          const global: any = window
          this.WritableStream = global.WritableStream || PofWritableStream
          this.useBlob = /constructor/i.test(global.HTMLElement) || !!global.safari || !!global.WebKitPoint
          try {
              new TransformStream()
              this.TransformStream = TransformStream
              this.supportsTransferable = true
          } catch (err) {
              console.error(err)
              try {
                  new PofTransformStream()
                  this.TransformStream = PofTransformStream
                  this.supportsTransferable = true
              } catch(err) {
                  console.error(err)
                  this.supportsTransferable = false
              }
          }
          try {
              new Response(new ReadableStream())
              if (!('serviceWorker' in navigator)) {
                  this.useBlob = true
              }
          } catch (err) {
              console.error(err)
              this.useBlob = true
          }
      }
  	// 创建iframe标签触发Service Worker拦截
      private makeIframe(src: string) {
          const iframe: any = document.createElement('iframe')
          iframe.hidden = true
          iframe.src = src
          iframe.loaded = false
          iframe.name = 'iframe'
          iframe.addEventListener('load', () => {
              iframe.loaded = true
          }, { once: true })
          document.body.appendChild(iframe)
      }
  	// 创建一个可写流
      async createWriteStream(fileName: string, size: number, opts: callback = {} ): Promise<WritableStream | null> {
          let bytesWritten = 0
          let transformStream: any = {}
          let sw: ServiceWorker = null
          const { port1, port2 } = new MessageChannel()
          const { progress, closed } = opts
          const supportsTransferable = this.supportsTransferable
          if (!this.useBlob) {
              sw = await registerWorker('sw.js', '/')
              if(!sw) {
                  this.useBlob = true
                  return null
              }
              const header = fileName.endsWith('.zip') ? {'Content-Type': 'application/zip'} : null
              // Make filename RFC5987 compatible
              // 这里是给HTTP响应头Content-Description使用，防止文件名非法字符或者中文时会报错
              const filename = encodeURIComponent(fileName.replace(/\//g, ':')).replace(/['()]/g, escape).replace(/\*/g, '%2A')
              const downloadUrl = location.origin + '/' + Math.random() + '/' + filename
              const transformer = supportsTransferable && {
                  transform(chunk, controller) {
                      if (!(chunk instanceof Uint8Array)) {
                          throw new TypeError('Can only write Uint8Arrays')
                      }
                      bytesWritten += chunk.length
                      progress&&progress(bytesWritten)
                      controller.enqueue(chunk)
                  },
                  flush() {
                      closed && closed()
                  }
              } 
              transformStream = supportsTransferable && new this.TransformStream(transformer) 
              const readableStream = transformStream.readable 
              port1.onmessage = evt => {
                  if (evt.data.abort) {
                      port1.postMessage('abort')
                      port1.onmessage = null
                      port1.close()
                      port2.close()
                  }
              }
              const params: any = { downloadUrl, size, filename, header }
              if(readableStream) {
                  params['readableStream'] = readableStream
                  sw.postMessage(params, [port2, readableStream])
              } else {
                  sw.postMessage(params, [port2])
              }
              this.makeIframe(downloadUrl)
              return (!this.useBlob && transformStream && transformStream.writable) ||  new this.WritableStream({
                  write(chunk) {
                      if (!(chunk instanceof Uint8Array)) {
                          throw new TypeError('Can only write Uint8Arrays')
                      }
                      port1.postMessage(chunk)
                      bytesWritten += chunk.length
                      progress&&progress(bytesWritten)
                  },
                  close() {
                      closed && closed()
                      port1.postMessage('end')
                  },
                  abort() {
                      port1.postMessage('abort')
                      port1.onmessage = null
                      port1.close()
                      port2.close()
                  }
              })
          }
          return null
      }
  }
  // 注册Service Worker 
  async function registerWorker(url: string, scope: string): Promise<ServiceWorker | null> {
      if(!navigator.serviceWorker) return null
      // url代表Service Worker文件路径, 'sw.js'代表根目录下/sw.js
      // scope表示该Service Worker的作用范围
      // '/'代表作用域根目录下所有路径， '/f'表示只作用域/f下的路径，作用于'/f/w','/f/#'，对'/c'无效
      const swReg = await navigator.serviceWorker.getRegistration(scope).then((swReg) => swReg || navigator.serviceWorker.register(url, { scope })).catch(() => null)
      if (!swReg) return null
      const swRegTmp = swReg.installing || swReg.waiting
      if (swReg.active) {
          return swReg.active
      } else {
          return new Promise(resolve => {
              const fn = () => {
                  if (swRegTmp.state === 'activated') {
                      swRegTmp.removeEventListener('statechange', fn)
                      resolve(swReg.active)
                  }
              }
              swRegTmp.addEventListener('statechange', fn)
          })
      }
  }
  // 让Service Worker保持活性
  function keepAlive(sw: ServiceWorker | null, scope: string) {
      const url = location.origin + scope + 'alive'
      const timer = setInterval(() => {
          if (sw) {
              sw.postMessage({ alive: true })
          } else {
              fetch(url).then((res: Response) => {
                  if (!res.ok) clearInterval(timer)
              })
          }
      }, 10000)
      return timer
  }
  
  // sw.js是一个单独文件，需要放在服务器静态资源下面，这样浏览器在注册Service Worker时候才能正确下载到sw.js,不然会404
  // sw.js
  self.addEventListener('install', () => {
      // Skip old wroker
      self.skipWaiting() // 跳过之前老的Service Worker
  })
  
  self.addEventListener('activate', (event) => {
      // Can be used without refreshing after successful registration
      // We can avoid this step by using iframe to download, but we still need to add for prevent accidents
      event.waitUntil(self.clients.claim())
  })
  
  const urlMap = new Map()
  // 接受主程序传递过来的一些信息
  self.onmessage = (event) => {
      const { alive } = event.data
      if (alive) return
      const { downloadUrl, size, filename, header, readableStream = null } = event.data
      const port = event.ports[0]
      urlMap.set(downloadUrl, { port, size, filename, header, readableStream })
  }
  // 创建响应流
  function creatReadableStream (port) {
      return new ReadableStream({
          start(controller) {
              port.onmessage = ({ data }) => {
                  try {
                      if (data === 'end') {
                          return controller.close()
                      }
                      if (data === 'abort') {
                          controller.error('Cancel Download')
                          return
                      }
                      controller.enqueue(data)
                  } catch (e) {
                      port.postMessage({ abort: true })
                  }
              }
          },
          cancel(reason) {
              port.postMessage({ abort: true })
          }
      })
  }
  // 拦截请求
  self.onfetch = (event) => {
      const url = event.request.url
      if (urlMap.get(url)) {
          const { port, size, filename, header, readableStream } = urlMap.get(url)
          urlMap.delete(url)
          const responseHeaders = new Headers({
              'Content-Type': 'application/octet-stream; charset=utf-8',
              'Content-Length': size,
              'Content-Disposition': "attachment;filename*=UTF-8''" + filename,
              'Content-Security-Policy': "default-src 'none'",
              'X-Content-Security-Policy': "default-src 'none'",
              'X-WebKit-CSP': "default-src 'none'",
              'X-XSS-Protection': '1; mode=block',
          })
          if (header) {
              for (let k in header) {
                  responseHeaders.set(k, header[k])
              }
          }
          const stream = readableStream || creatReadableStream(port)
          event.respondWith(new Response(stream, { headers: responseHeaders }))
      }
      if (url.endsWith('alive')) {
          event.respondWith(new Response('alive'))
      }
  }
  ```

* **原理**

  首先我们通过[navigator.serviceWorker]('https://developer.mozilla.org/zh-CN/docs/Web/API/Service_Worker_API')**注册一个serviceworker**，再**创建一个可写流**，当我们向可写流写入数据时候，数据是流向serviceworker里面去的，所以注册以后通过postmessage把**WriteableStream和ServiceWorker通信方式给绑定**（*这里如果我们是创建的转化流的话，我们就可以把**转化流的可读端传递给ServiceWorker**，因为转化流分为可读和可写两端，当可写端写入数据时候默认是流向可读端的，如果不是转化流，那么我们就需要再构建可写流的时候通过**MessageChannel**创建两个端口，一端绑定在可写流上面，一端**传输给ServiceWorker**在sw.js里面把这**一端绑定**在**可写流**里面，这样就实现数据通信）。我们再把下载地址传递给ServiceWorker，当我们通过iframe访问下载地址时候，**serviceworker**会**拦截**该**请求**并设置响应头描述Content-Disposition让浏览器下载该响应返回的东西，拦截请求后**返回我们关联的可读流**响应回来，浏览器就会默认创建下载，此时再在程序里面往可写流写入数据，默认就是下载到磁盘。[参考]('https://github.com/jimmywarting/StreamSaver.js')

* **使用示例**

  fetch(fetch返回的body是一个可读流)

  ```javascript
  const saver = new StreamSaver()
  const writeStream = saver.createWriteStream('test.txt', 1024)
  const { body } = await fetch(url) // fetch
  const writer = writeStream.getWriter()
  const reader = body.getReader()
  while(true) {
      const { value, done } = await reader.read()
      if(!done) {
          writer.write(value)
      } else {
          writer.close()
          break
      }
  }
  ```

* **存在的BUG**

  测试发现当createWriteStream返回的可写流是新new的WiriteableStream的时候使用的内存是TransformStream的可写端时的双倍，这个原因还未明确。测试时候能够发现垃圾清除机制未把一些内存释放掉，手动点击浏览器的垃圾回收内存就会降下去。