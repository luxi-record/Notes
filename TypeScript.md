## TypeScript的一些知识点 ##

1. any和unknown是TS中顶层类型，任何类型都可赋值给顶层类型，但是顶层类型不能赋值给指定类型（any除外，any不会进行类型判断，尽量不要用any，使用unknown代替），unknown类型在进行操作时候需要先进行类型断言

   ```javascript
   let a: any
   let b: unknown = 4
   let c: number = 2
   c = a，b = c， a = c //不报错
   c = b //报错
   b + c //报错，这里需要先进行对进行断言 if(typeof b === 'number') b + c or b as number
   ```

2. nerver类型是底层类型，它是任何类型的子集，他可以赋值给任何类型，但是其他类型不能赋值给它，集合理论中，nerver相当于空集，空集合是任何集合的子集，而any和unkonwn是顶层集合包含所有类型。

3. 常用的一些类型工具

   ```typescript
   // Partial<type>
   type people = {
     name: string
     age: number
   }
   Partial<people> = {
     name?: string
     age?: number
   }
   const a: Partial<people> = {name: 'pick'} // Partial<type>可以把类型中必填的key改为选填 
   // Pick<Type, Keys> 
   // Omit<Type, Keys> 和pick相反，表示删除相应的key
   Pick<people, 'name'> = {
     name: string
   }
   const a: Pick<people, 'name'> = {name: 'pick'} // Pick<Type, Keys>相当于把people中的一些key提取出来
   // Required<Type>
   type people2 = {
     name: string,
     age?: number
   }
   Required<people2> = {
     name: string
     age: number
   }
   const a: Required<people2> = { name: 'pick', age: 22 } // Required<Type>相当于把type中的可选变为必填
   // ReturnType<Type> 它的类型与函数返回的类型一样
   function number () { return 2 }
   let a: ReturnType<typeof setTimeout> // a: Timeout
   let b: ReturnType<typeof number> // b: number
   let c: ReturnType<() => string> // c: string
   ```

4. //@ts-ignore 表示不对下一行代码进行类型检查
5. 泛型是指没有固定类型例如
   ```javascript
   function log<T>(val: T): T {
      return val
   }
   // 这样log可以打印任何类型，不用为每一种类型单独定义
   // log<string>('name') log<number>(2)
   // 还可以指定多个泛型
   function log<T,R>(val: T, val2: R): T | R {
      return val || val2
   }
   ```
