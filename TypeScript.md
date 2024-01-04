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

