<center><font face="黑体" size=24 >一些常用的工具函数和方法</font></center>

1. **异步的并发控制**（只实现大致逻辑，具体业务逻辑再添加）

   ```javascript
   /*
   	** Promise.all()版本
   	** @params task为任务队列，每个任务返回一个promise
   	** @params limit为并发数量
   	** @return Promise
   */
   interface task { (): Promise<any>}
   function Taskpool(task: task[], limit: number) {
       if (task.length < 1) {
           return
       }
       let index = 0, resulet = []
       let queue = Array(limit).fill(null)
       queue = queue.map(() => {
           return new Promise((resolve, reject) => {
               const fn = () => {
                   if (index >= task.length) {
                       resolve()
                       return
                   }
                   const taskItem = task[index]
                   const resuletIdnex = index
                   index++
                   taskItem().then((res) => { 
                       resulet[resuletIdnex] = res
                       fn()
                   }).catch((err) => {
                       reject(err)
                   })
               }
               fn()
           })
       })
       return Promise.all(queue).then(() => { return resulet })
   }
   /*
   	** 构造函数版本
   	** @params task任务队列，每个任务返回promise
   	** @params limit为并发数量
   	** @return Promise
   */
   interface task { (): Promise<any>}
   class Control {
       private taskList: task[]
       private isStop: boolean = true
       private maxRunner: number = 0
       result: any[] = []
       private running: number = 0
       private index: number = 0
       constructor(task: task[], limit: number) {
           this.taskList = task
           this.maxRunner = limit
       }
       public run() {
           this.isStop = false
           return new Promise((resolve, reject) => {
               const runTask = () => {
                   while (this.running < this.maxRunner && this.taskList.length && !this.isStop) {
                       const task = this.taskList.shift()
                       this.running++
                       task().then((r) => {
                           this.result[this.index] = r
                           this.running--
                           this.index++
                           if (!this.isStop) {
                               runTask()
                           }
                       }).catch((r) => {
                           this.isStop = true
                           reject(r)
                       })
                   }
                   if (this.running === 0 && this.taskList.length === 0) {
                       resolve(this.result)
                   }
               }
               runTask()
           })
       }
       public stop() {
           this.isStop = true
       }
   }
   ```

2. **复制**

   ```javascript
   /*
   	** @params val为需要复制的字符串
   	** @return boolean复制成功与否
   */
   function copy(val: string) {
       const input = document.createElement('input')
       input.value = val
       document.body.appendChild(input)
       input.select() // 选择复制内容
       const IsCopySuccess = document.execCommand('copy') // 执行复制命令
       document.body.removeChild(input)
       return IsCopySuccess
   }
   ```

3. **一般的文件下载方式**

   ```javascript
   /*
   	** @params res为下载地址或者blob数据
   	** @params name为文件名
   */
   function download(res: string | Blob, name: string) {
       let url
       if (Object.prototype.toString.call(res) === '[object String]') {
           url = res
       } else if (Object.prototype.toString.call(res) === '[object Array]') {
           const blob = new Blob(res)
           url = window.URL.createObjectURL(blob)
       }
       const a = document.createElement('a')
       a.href = url
       a.download = name // a 标签的download属性为文件名
       document.body.appendChild(a)
       a.click()	// 点击a标签下载
       document.body.removeChild(a)
       window.URL.revokeObjectURL(url)
   }
   ```

4. **表格导出xls**

   ```javascript
   /*
   	** @params title为表格表头, [{key: 'name', value: '名字'},{key: 'age', value: '年龄'}]
   	** @params data为数据 [{name: '小王', age: 25}]
   	** @params filename为文件名
   */
   type title = {
       key: string,
       value: string
   }
   function exportCsv(title: title[], data: any[], filename?: string) {
       const res: string[] = [], titleKey: string[] = [], titles: string[] = []
       title.forEach((item: any) => {
           titleKey.push(item.key)
           titles.push(item.value)
       })
       res.push(titles.join(',') + '\n')
       data.forEach((item: any) => {
           const tmp: any = {}
           titleKey.forEach((key: string) => {
               tmp[key] = item[key]
           })
           res.push(Object.values(tmp).join(',') + '\n')
       })
       const blob = new Blob(['\uFEFF' + res.join('')], { // ’\uFEFF‘防止乱码
           type: 'text/plain;charset=utf-8',
       })
       const a = document.createElement('a'), url = URL.createObjectURL(blob)
       a.download = filename || 'download.csv'
       a.href = url
       document.body.appendChild(a)
       a.click()
       document.body.removeChild(a)
       URL.revokeObjectURL(url)
   }
   ```

5. **回溯搜索目标路径**

   ```javascript
   /*
   	** @params data一颗树对象{id: 1, child: [ {id: 2, child: []}, {id: 3, child: []}]}
   	** @params id要寻找的id
   	** @return [], 路径
   */
   function searchNode(data, id) {
       let res, path = []
       path.push(data.id)
       if (id === data.id) {
           return path
       }
       const child = data.child
       let traverse = (curKey, path, data) => {
           if (data.length === 0) {
               return;
           }
           for (let item of data) {
               path.push(item.id);
               if (item.id === curKey) {
                   res = [...path];
                   break;
               }
               const children = Array.isArray(item.child) ? item.child : [];
               traverse(curKey, path, children); // 遍历
               path.pop(); // 回溯
           }
       }
       traverse(id, path, child)
       return res
   }
   
   ```

6. **大数相加**（由于精度问题js中超大数字不能直接相加）

   ```javascript
   /*
   	** @params a 大数a整数
   	** @params b 大数b整数
   	** @return res 相加结果
   */
   function bigNumAdd(a, b) {
       const maxLen = String(a).length > String(b).length ? String(a).length : String(b).length
       const str1 = String(a).padStart(maxLen, 0), str2 = String(b).padStart(maxLen, 0)
       let j = 0, res = '', t
       for (let i = maxLen - 1; i >= 0; i--) {
           t = Number(str1[i]) + Number(str2[i]) + j
           j = Math.floor(t / 10)
           res = res + t % 10
       }
       return res
   }
   ```

7. **浮点数相加**（也是精度问题）

   ```javascript
   /*
   	** @params num1浮点数
   	** @params num2浮点数
   	** @return res 相加结果
   */
   // 如果浮点数位数过多可采用大数相加
   // 如果浮点数浮点左右两边数字都多可把浮点两端都当作大数再相加再组合
   function floatAdd(num1, num2) {
       const len1 = String(num1).split('.')[1].length, len2 = String(num2).split('.')[1].length
       const q = len1 > len2 ? Math.pow(10, len1) : Math.pow(10, len2)
       const a = num1 * q, b = num2 * q
       const res = (a + b) / q
       return res
   }
   ```

8. **原生cookie添加和删除**

   ```javascript
   // 添加
   function setCookie(name: string, val: string, d?: number) {
       let cok = `${name}=${val};`
       if (d !== undefined) {
           const date = new Date(), expiresDate = date.getDate() + d
           date.setDate(expiresDate)
           cok = cok + `expires=${date.toUTCString()};`
       }
       document.cookie = cok
   }
   // 获取
   function getCookie(name: string) {
       if (!document.cookie) return null
       const str = document.cookie, arr = str.split(';')
       let res: string | null = null
       for (let i = 0; i < arr.length; i++) {
           if (arr[i].trim().split('=')[0] === name) {
               res = arr[i].trim().split('=')[1]
               break
           }
       }
       return res
   }
   // 删除
   function removeCookie(name: string) {
       setCookie(name, '', -1)
   }
   ```

9. **GET请求参数拼接**（最简便）

   ```javascript
   function getUrl (url: string, data: Record<string, string | number>) {
       return url =+ '?' + Object.keys(data).reduce((pre, n) => pre + '&' + n + '=' + String(data[n]), '').slice(1)
   }
   ```

10. **防抖节流**

   ```javascript
   // 防抖
   function debance(fn: Function, delay?: numbner) {
       let timer = null;
       return function() {
           let that = this;
           let args = arguments;
           clearTimeout(timer);
           timer = setTimeout(function() {
               fn.apply(context, args);
           }, delay || 300);
       }
   }
   // 节流
   function throttle(cb: Function, delay?: number) {
       let time = 0
       const delayTime = delay || 300
       return (arg?: any) => {
           const t = new Date().getTime()
           if (t - time >= delayTime) {
               cb(arg)
               time = t
           }
       }
   }
   ```

11. **深拷贝**（简单实现只适用于对象和数组，解决循环引用的问题）

    ```javascript
    function deepCopy(obj,map = new WeakMap()){
        if (typeof obj != 'object') return 
        var newObj = Array.isArray(obj)?[]:{}
        if(map.get(obj)){ 
          return map.get(obj); 
        } 
        map.set(obj, newObj);
        for(var key in obj){
            if (obj.hasOwnProperty(key)) {
                if (typeof obj[key] == 'object') {
                    newObj[key] = deepCopy(obj[key],map);
                } else {
                    newObj[key] = obj[key];
                }
            }
        }
        return newObj;
    }
    ```