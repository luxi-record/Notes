vue中**计算属性**是**可以缓存**的，当计算属性依赖的值没发生改变时，用的就是上次的值。

vue中**modelValue属性和update:modelValue事件**是**对应**的**v-modal**这个指令的语法糖类似，当父组件在修改了子组件的v-modal的值对应子组件的props.modalValue也会发生改变，同样子组件自身可以通过emit('update:modelValue', value)修改v-modal的值，对应父组件绑定的值也会改变

vue3可以通过widthDefaults(defineProps(), {})，设置组件默认props的值，大括号就是要填入的默认值

vue中v-if的优先级高于v-for所以这两个指令不能用在同一元素，v-if是访问不到v-for的作用域的。

```html
<template>
    <div v-for="val in list" v-if="val.show"> // 这里v-if是访问不到v-for中val的值的
        123
    </div>
</template>
<script setup>
    import { ref } from 'vue'
	const list = ref([{show：false, age: 22}])
</script>
```

vue3中常用两种定义组件的方法：**defineComponent**，**setup**

```html
<template>
    <Teleport to="body"> // Teleport 组件可以把我们定义的组件渲染到我们想挂载的地方。vue自带组件
        <div v-if="modelValue">
            <slot />
        </div>
    </Teleport>
</template>
<script lang="ts">
import { defineComponent, ref, toRefs } from 'vue'
export default defineComponent({
    name: 'modal', // 可有可无
    props: ['modelValue'], // 属性。
    emits: ['update:modelValue', 'ok'], //事件
    setup(props,{ emit,expose,attrs,slots}) { // 组合式api，在created和beforeCreated之间执行，不能访问data和method的值及组件自身this
        const { modelValue } = toRefs(props) // toRefs可以解构props是的解构出来也是响应式的，父组件传入的值发生改变子组件解构出来的也会改变
        const cancel = () => {
            emit('update:modelValue', false)
        }
        const ok = () => {
            emit('ok', 'log') 
        }
        console.log(attrs) // attrs是那些没有定义在props中的属性
        expose({}) //expose暴露的值是供给父组件通过ref访问的
        return { // 只有return暴露出来的值可以供子组件模板使用
            modelValue,
            cancel,
            ok 
        }
    },
})
</script>
```

**defineComponent**也可以写jsx

```javascript
import { defineComponent, ref, toRefs } from 'vue'
export default defineComponent({
    name: 'modal', // 可有可无
    props: ['modelValue'], // 属性。
    emits: ['update:modelValue', 'ok'], //事件
    setup(props,{ emit,expose,attrs,slots}) { // 组合式api，在created和beforeCreated之间执行，不能访问data和method的值及组件自身this
        const { modelValue } = toRefs(props) // toRefs可以解构props是的解构出来也是响应式的，父组件传入的值发生改变子组件解构出来的也会改变
        const cancel = () => {
            emit('update:modelValue', false)
        }
        const ok = () => {
            emit('ok', 'log') 
        }
        console.log(attrs) // attrs是那些没有定义在props中的属性
        expose({}) //expose暴露的值是供给父组件通过ref访问的
        return () => { // 这里写jsx，类似react
            return (
            	<div></div>
            )
        }
    },
})
```



```html
<template>
    <Teleport to="body">
        <div v-if="modelValue">
            <slot />
        </div>
    </Teleport>
</template>
<script lang="ts" setup> // 标签中的setup相当于是一个编译语法糖，我们可以不用像上面setup函数中那样return主动暴露值给模板
import { useSlots, useAttrs } from 'vue'
const props = defineProps({ // 定义属性
    modelValue: Boolean
})
const emit = defineEmits(['update:modelValue', 'ok']) // 事件
defineExpose({}) // 暴露给父组件点通过ref调用子组件的对象
const slots = useSlots() // 访问插槽
const attrs = useAttrs() // 访问除定义在props中的属性
const ok = () => {
    emit('ok', 'log')
}
const cancel = () => {
    emit('update:modelValue', false)
}
</script>
```

**script中setup语法糖和defineComponent的对应关系:**

1. defineComponent的**props**对应**defineProps**
2. defineComponent中setup的**expose**对应**defineExpose**
3. defineComponent中setup的**attrs**对应**useAttrs**
4. defineComponent中setup的**slots**对应**useSlots**
5. defineComponent中setup需要**返回一个对象**供组件模板使用**语法糖不需要**

expose暴露的值父组件怎么使用**案例**：

```html
// 子组件
<template>
    <div>
        child
    </div>
</template>
<script lang="ts" setup>
const log = () => {}
defineExpose({log}) // 暴露给父组件点通过ref调用子组件的对象
</script>

// 父组件
<template>
    <child ref="refs" />
</template>
<script lang="ts" setup>
import child from './child.vue'
const refs = ref()
refs.value.log() // 可以通过子组件的ref属性访问，refs.value对应的就是expose暴露的对象
</script>
```

在组件模板中我们可以通过$props或者$attrs直接访问props属性和其他属性

```html
<template>
    <div>
        {{ $props.key }}
        {{ $attrs.key }}
    </div>
</template>
```

watch和watchEffect的使用：

1. watch注册的回调不会立马执行，而watchEffect注册的回调是立马执行。watch想要立马执行，则需要传入第三个参数配置项{immediate: true}
2. watch需要显示的写出监听的对象，而watchEffect会在第一次运行时自动需找出监听的依赖。
3. 他们两个都是可以是设置回调触发时机**默认是在更新渲染beforeUpdate之前**，watch回调可接受依赖改变后的值
4. watch第一个参数可以是一个响应式对象或者响应式对象数组或者一个getter函数，例如：() => state.key
5. watchEffect和watch可以返回一个停止监听的函数，watchEffect回调函数接受一个清除副作用的函数
6. watchEffect接受第二个参数表示副作用函数执行时机watchEffect((fn) => {}, {flush: 'post'}) ，post代表在beforeUpdate之后updated之前执行，sync代表依赖改变时立即执行（需要谨慎使用，因为有多个依赖时候会存在性能问题），pre默认值在beforeUpdate之前调用

```html
<template>
    <div>
        child
    </div>
</template>
<script lang="ts" setup>
    import { ref, watch, watchEffect } from 'vue'
    const state = ref('1')
   	const state2 = ref('2')
    const stopWatch = watch([state，state2], (news, pre) => { 
        console.log(news, pre)
    }, 
    {immediate: true, deep: true, flush: 'post'}) // immediate代表是否立即执行和watchEffect一样，deep代表是否深度监听,flush代表执行时机
    const stop = watchEffect((fn) => { // fn是一个可以**清除本次副作用**的函数，比如当我们分页的state改变时候去请求异步数据，然而异步接口响应时间不固定，我们先在fn立马注册一个取消本次请求的函数，当我们连续点击第1，2，3页时候会触发三次请求，当点击2的时候如果1的请求还没响应则会被fn注册的回调所取消
        if(state.value) {
            console.log('update')
        }
        fn(() => { // fn只会在依赖改变或卸载组件时候执行，并且当依赖改变时候会先执行fn回调清除上次的副作用再执行本次改变后的副作用
            // clean
        })
    }，{flush: 'post'/'sync'})
</script>
```

插槽slot的使用：

1. 插槽类似react中的child，只不过比child更高级
2. 插槽标签也可以绑定一些属性指令：v-for，v-if等
3. 插槽可以在<slot :value="value"></slot>标签上绑定暴露给父组件的值。切记不要在slot绑定一些特殊的属性，例如：name等

```html
// child
<template>
	<div>
        <slot>默认内容</slot>
        <slot name='content'>我是默认content内容</slot>
        <slot name='s3' :state='state' :otherName='otherState'></slot> // 插槽作用域暴露子组件内数据供父组件调用
    </div>
</template>
<script setup>
    import { ref } from 'vue'
    const state = ref(1)
    const otherState = ref('other')
</script>
// parent
<template>
	<child>
    	child 标签之间的类容就是插槽，如果有内容会覆盖掉插槽的默认内容
        <template #content>我是覆盖名为content插槽的内容</template>
        <template #[name]>我是动态插槽的内容</template>
        <template #s3='props'>{{ props.state }}</template> // 对应就是插槽标签绑定的props中的state
    </child>
</template>
<script setup>
    import { ref } from 'vue'
    const name = ref('content')
</script>
```

vue中的动态组件和递归组件：**通过setup语法糖定义的组件默认组件名和文件名一样**

```html
// 动态组件主要借助<component />标签实现， 他有一个props：is
// is可以是字符串也可以是一个vue组件名
<template>
    <component :is="ismodal ? Modal : 'text content' " />
</template>
<script setup>
    import { ref } from 'vue'
    import Modal from './modal.vue'
    const isModal = ref(false)
</script>

// 递归组件（组件自身调用自身）
// tree.vue
<template>
	<li>
    	<p>
            {{ props.tree.name }}
        </p>
        <ul>
            <tree v-if="props.tree.child" v-for="(item, index) in props.tree.child" :key="index" :tree="item" />
        </ul>
    </li>
</template>
<script setup>
	const props = defineProps(['tree'])
</script>
```

vue跨多层次组件数据传递可以通过**provide**和**inject**实现。类似react的createContext和useContext

```html
// 父组件
<script setup>
import { provide，ref} from 'vue'
const count = ref(0)
provide('count', count)
</script>
// 子组件
<script setup>
import { inject} from 'vue'
const count = inject('count', 1) // 如果没有count就用1为默认值
</script>
```

vue自定义指令：

```html
// 在setup语法糖中
<script setup>
// 在模板中启用 v-focus
const vFocus = {
  mounted: (el) => el.focus() // v的驼峰就代表一个指令
}
// 其实指令就是在生命周期中做一些事情
const myDirective = {
  // 在绑定元素的 attribute 前
  // 或事件监听器应用前调用
  created(el, binding, vnode, prevVnode) {
    // 下面会介绍各个参数的细节
  },
  // 在元素被插入到 DOM 前调用
  beforeMount(el, binding, vnode, prevVnode) {},
  // 在绑定元素的父组件
  // 及他自己的所有子节点都挂载完成后调用
  mounted(el, binding, vnode, prevVnode) {},
  // 绑定元素的父组件更新前调用
  beforeUpdate(el, binding, vnode, prevVnode) {},
  // 在绑定元素的父组件
  // 及他自己的所有子节点都更新后调用
  updated(el, binding, vnode, prevVnode) {},
  // 绑定元素的父组件卸载前调用
  beforeUnmount(el, binding, vnode, prevVnode) {},
  // 绑定元素的父组件卸载后调用
  unmounted(el, binding, vnode, prevVnode) {}
}
// el：指令绑定到的元素。这可以用于直接操作 DOM。
binding：一个对象，包含以下属性。
value：传递给指令的值。例如在 v-my-directive="1 + 1" 中，值是 2。
oldValue：之前的值，仅在 beforeUpdate 和 updated 中可用。无论值是否更改，它都可用。
arg：传递给指令的参数 (如果有的话)。例如在 v-my-directive:foo 中，参数是 "foo"。
modifiers：一个包含修饰符的对象 (如果有的话)。例如在 v-my-directive.foo.bar 中，修饰符对象是 { foo: true, bar: true }。
instance：使用该指令的组件实例。
dir：指令的定义对象。
vnode：代表绑定元素的底层 VNode。
prevNode：之前的渲染中代表指令所绑定元素的 VNode。仅在 beforeUpdate 和 updated 钩子中可用。
</script>

// export default写法
export default {
  setup() {
    /*...*/
  },
  directives: {
    // 在模板中启用 v-focus
    focus: {
      /* ... */
    }
  }
}

```

#### 状态管理工具Pinia
