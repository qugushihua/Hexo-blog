---
title: Pinia
keywords: vue3,pinia
cover: [https://www.logosc.cn/uploads/resources/2018/11/29/1543459457_thumb.jpg]
sticky: 10
banner: 
  type: img
  bgurl: https://pic1.zhimg.com/v2-b3c2c6745b9421a13a3c4706b19223b3_r.jpg
  bannerText: Hi my new friend!
---
## Pinia

### 简介：

Pinia.js 有如下特点：

完整的 ts 的支持；
足够轻量，压缩后的体积只有1kb左右;
去除 mutations，只有 state，getters，actions；
actions 支持同步和异步；
代码扁平化没有模块嵌套，只有 store 的概念，store 之间可以自由使用，每一个store都是独立的
无需手动添加 store，store 一旦创建便会自动添加；
支持Vue3 和 Vue2
官方文档[Pinia](https://pinia.vuejs.org/)

git 地址 https://github.com/vuejs/pinia

### 简单使用：

安装

```
npm install pinia -S
```

在main.ts中引入

```ts
import {createPinia} from 'pinia'
//因为是一个hook，所以需要调用一下
const store=createPinia()//导出后是个插件，需要use
createApp(App).use(store).mount('#app')
```

新建store文件夹，index.ts->其他地方导入store会自动寻找index.ts

```ts
//store-name.ts用于命名
export const enum Names{
  TEST='TEST'
}

//index.ts中
import{defineStore} from 'pinia'
import {Names} from './store-name'

export const useTestStore=defineStore(Names.TEST,{//第一个参数为id，可以理解为命名空间；第二个参数为对象
  state:()=>{//是一个箭头函数，返回一个对象
    return {
      current:1,
      name:"Ting"
    }
  },
  //类似于computed，修饰一些值
  getters:{

  },
  //类似于methods，同步异步都可以 提交state
  actions:{

  }
})
```

vue页面中使用

```vue
<script setup lang="ts">
import {useTestStore} from './store'//一般hook以use开头
const Test=useTestStore()
</script>
<template>
  <div>
    pinia:{{Test.current}}-{{Test.name}}
  </div>
</template>
```

### State：

#### 修改state：

方式一：直接修改

```ts
const change=()=>{
  Test.current++;
}
```

方式二：$patch，支持批量修改

```ts
Test.$patch({
    current:1000,
    name:"CY"
  })
```

方式三：$patch,函数式，可以进行逻辑处理

```ts
Test.$patch((state)=>{
    //if(...)...
    state.current=999
    state.name="Li"
  })
```

方式四：$state,缺点是必须修改整个对象

```ts
Test.$state={//必须全传
    current:1919,
    name:"人间烟火"
  }
```

方式五：借助actions

```ts
//index.ts中
actions:{
    setCurrent(){//不要写成箭头函数，否则this指向错误
      this.current=9999
    }
  }

//app.vue中
Test.setCurrent()//直接调就可以
```

也可以传入参数

```ts
actions:{
    setCurrent(num:number){
      this.current=num
    }
  }
  
Test.setCurrent(456)
```

#### 解构store

注意：pinia中解构不具有响应式

```ts
const {current,name}=Test
```

如何解决：storeToRefs

```ts
import { storeToRefs } from 'pinia';//核心原理和toRefs一样
const {current,name}=storeToRefs(Test)
```

### Actions：

#### 同步写法：

直接调用即可

```ts
type User={
  name:string,
  age:number
}
let result:User={
  name:"Ting",
  age:19
}

//defineStore中
state:()=>{
    return {
      user:<User>{},
      name:""
    }
  },
actions:{
    //同步写法
    setUser(){
      this.user=result
    }
  }
```

#### 异步写法：

可以结合async await 修饰

```ts
//先定义一个异步函数
const Login=():Promise<User>=>{//返回一个Promise，接收一个泛型User
  return new Promise((resolve) =>{//resolve, reject两个参数，但此处我们只用到一个resolve
    setTimeout(()=>{
      resolve({
        name:"Ting",
        age:19
      })
    },2000)
  })
}

//defineStore中
actions:{
    async setUser(){//一般用async，await修饰
      const result =await Login()//用result接收返回值
      this.user=result
    }
  }
```

也可以和其他方法相互调用

```ts
actions:{
    async setUser(){
      const result =await Login()
      this.user=result
      this.setName('^_^')
    },
    setName(name:string){
      this.name=name
    }
  }
```

### Getters：

有缓存

```ts
getters:{
    newName():string{//ts无法进行正确的类型推导，所以定义返回值string
      return `$-${this.name}-${this.getUserAge}`
    },
    //也可以相互调用
    getUserAge():number{
      return this.user.age
    }
  }
```

### 实例API

#### $reset

重置store至其初始状态

```ts
//直接调用即可
const reset=()=>{
  Test.$reset()
}
```

#### $subscribe

只要state值有变化，就会触发这个函数（类似于Vuex 的==abscribe== ）

```ts
Test.$subscribe((args,state)=>{//返回一个工厂函数,接收两个参数
  console.log("args",args);
  console.log("state",state);
},{//第二个参数接收一个对象
  detached:true,
  deep:true,
  flush:"post"
})
```

#### $onAction

只要调用actions，就会监听

```ts
Test.$onAction((args)=>{
  args.after(()=>{
    console.log("after");
  })
  console.log(args);
},true)//第二个参数：为true时，当组件被销毁后可以仍然监听
```

![image-20221209115047400](.assets/image-20221209115047400.png)

顺序是这样的，和watchEffect反过来了

args就是捕获到的参数

### pinia插件

pinia 和 vuex 都有一个通病 页面刷新状态会丢失

我们可以利用localStorage自己封装一个持久化插件

```ts
import {createPinia,PiniaPluginContext} from 'pinia'


//定义入参类型
type Options={
  key?:string
}

//定义兜底变量
const __PiniaKey__:string="default"

//取值
const getStorage=(key:string)=>{
  return localStorage.getItem(key)?JSON.parse(localStorage.getItem(key) as string):{}
}

//存值
const setStorage=(key:string,value:any)=>{
  localStorage.setItem(key,JSON.stringify(value))
}

//定义持久化存储插件，利用函数柯里化接受用户入参
const piniaPlugin=(options:Options)=>{
	//将函数返回给pinia  让pinia  调用 注入 context
  return (context:PiniaPluginContext)=>{//return一个函数出去，这个函数来接收pinia的返回值
    const {store}=context
    const data=getStorage(`${options?.key??__PiniaKey__}-${store.$id}`)
    store.$subscribe(()=>{
      setStorage(`${options?.key??__PiniaKey__}-${store.$id}`,toRaw(store.$state))//不能直接用Proxy，要转成原始对象
    })
    //返回值覆盖pinia 原始值
    return{
      ...data
    }
  }
}

//初始化pinia
const store=createPinia()

//注册pinia 插件
store.use(piniaPlugin({
  key:"pinia"
}))
```
