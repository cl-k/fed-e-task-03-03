# Vuex 数据流管理及 Vue.js 服务端渲染（SSR）

## Vuex 状态管理

- Vue 组件间通信方式回顾
- Vuex 核心概念和基本使用回顾
- 购物车案例
- 模拟实现 Vuex

### 组件内状态管理流程

状态管理

- state, 驱动应用的数据源
- view，以声明方式将 state 映射到视图
- actions，响应在 view 上的用户输入导致的状态变化

### 组件间通信方式——父组件给子组件传值

- 子组件中通过 props 接收数据
- 父组件中给子组件通过相应属性传值

```vue
// child.vue
<template>
  <div>
    <h1>Props Down Child</h1>
    <h2>{{ title }}</h2>
  </div>
</template>

<script>
export default {
  // props: ['title'],
  props: {
    title: String
  }
}
</script>
```

```vue
// parent.vue
<template>
  <div>
    <h1>Props Down Parent</h1>
    <child title="test title"></child>
  </div>
</template>

<script>
import child from './01-Child'
export default {
  components: {
    child
  }
}
</script>
```



### 组件间通信方式——子组件给父组件传值

- 在子组件中使用 $emit 发布自定义事件
- 在父组件中使用 v-on 监听子组件的自定义事件

```vue
// child.vue
<template>
  <div>
    <h1 :style="{ fontSize: fontSize + 'em' }">Props Down Child</h1>
    <button @click="handler">文字增大</button>
  </div>
</template>

<script>
export default {
  props: {
    fontSize: Number
  },
  methods: {
    handler () {
      this.$emit('enlargeText', 0.1)
    }
  }
}
</script>

```

```vue
// parent.vue
<template>
  <div>
    <h1 :style="{ fontSize: hFontSize + 'em'}">Event Up Parent</h1>

    这里的文字不需要变化

    <child :fontSize="hFontSize" v-on:enlargeText="enlargeText"></child>
    <child :fontSize="hFontSize" v-on:enlargeText="enlargeText"></child>
    <child :fontSize="hFontSize" v-on:enlargeText="hFontSize += $event"></child>
  </div>
</template>

<script>
import child from './02-Child'
export default {
  components: {
    child
  },
  data () {
    return {
      hFontSize: 1
    }
  },
  methods: {
    enlargeText (size) {
      this.hFontSize += size
    }
  }
}
</script>
```



### 组件间通信方式——不相关组件之间传值

也是使用自定义事件的方式，但使用总线（bus）的方式触发

使用 Event Bus 来解决

```eventbus.js```:

```js
export default new Vue()
```

然后在需要通信的两端分别进行订阅和发布的操作

使用 ```$on``` 订阅：

```js
// 没有参数
bus.$on('funcName', () => {
	// do something
})

// 有参数
bus.$on('funcName', data => {
	// do something
})
```

使用 ```$emit``` 发布：

```js
// 没有自定义传参
bus.$emit('funcName')

// 有自定义传参
bus.$emit('funcName', data)
```

### 组件间通信方式——其他方式

- $root
- $parent
- $children
- $refs

但不推荐使用，滥用这些方法会导致数据管理的混乱

通过 ref 获取子组件：

ref 有两个作用

- 在普通 HTML 标签上使用 ref，获取到的是 DOM
- 在组件标签上使用 ref，获取到的是组件实例

```vue
// child.vue
<template>
  <div>
    <h1>ref Child</h1>
    <input ref="input" type="text" v-model="value">
  </div>
</template>

<script>
export default {
  data () {
    return {
      value: ''
    }
  },
  methods: {
    focus () {
      this.$refs.input.focus()
    }
  }
}
</script>
```

```vue
// parent.vue
<template>
  <div>
    <h1>ref Parent</h1>

    <child ref="c"></child>
  </div>
</template>

<script>
import child from './04-Child'
export default {
  components: {
    child
  },
  mounted () {
    this.$refs.c.focus()
    this.$refs.c.value = 'hello input'
  }
}
</script>
```

> ```$ref``` 只会在组件渲染完成之后生效，并且它们不是响应式的，这仅作为一个用于直接操作子组件的“逃生舱”——应该避免在模板或计算属性中访问```$ref```

### 简易的状态管理方案

如果多个组件之间要共享数据，使用前面的方式虽可实现，但比较麻烦，而且多个组件之间互相传值很难跟踪数据的变化，且出现问题很难定位。

当遇到多个组件需要共享状态的时候，比如购物车。如果使用上面的方案就不合适，会遇到以下问题：

- 多个视图依赖于同一状态
- 来自不同视图的行为需要变更同一状态

对于问题一，传参的方法对于多层嵌套的组件将会非常繁琐，并且对于兄弟组件间的状态传递无能为力

对于问题二，经常会采用父子组件直接引用或者通过事件来变更和同步状态的多份拷贝

但以上的这些模式非常脆弱，通常会导致代码无法维护。因此，我们可以把组件的共享状态抽取出来，以一个全局单例模式管理。在这种模式下，组件树构成了一个巨大的“视图”，不管在树的哪个位置，任何组件都能获取或者触发行为。

简单的实现方式

- 首先创建一个共享的仓库 store 对象

  ```js
  export default {
    debug: true,
    state: {
      user: {
        name: 'xiaomao',
        age: 18,
        sex: '男'
      }
    },
    setUserNameAction (name) {
      if (this.debug) {
        console.log('setUserNameAction triggered：', name)
      }
      this.state.user.name = name
    }
  }
  ```

- 把共享的仓库 store 对象，存储到需要共享状态的组件的 data 中

  ```js
  import store from './store'
  export default {
    methods: {
      change () {
        // 点击按钮时通过 action 修改状态
        store.setUserNameAction('componentB')
      }
    },
    data () {
      return {
        privateState: {},
        sharedState: store.state
      }
    }
  }
  ```

接着要继续延伸约定，组件不允许直接变更属于 store 对象的 state，而应执行 action 来分发（dispatch）事件通知 store 去改变，这样最终的样子跟 Vuex 的结构就类似。这样约定的好处是，我们能够记录所有 store 中发生的 state 变更，同时实现能做到记录变更、保存状态快照、历史回滚/时光旅行的先进的调试工具

### Vuex 概念回顾

什么是 Vuex

- Vuex 是专门为 Vue.js 设计的状态管理库
- Vuex 采用集中式的方式存储需要共享的状态
- Vuex 的作用是进行状态管理，解决复杂组件通信，数据共享
- Vuex 集成到了 devtools 中，提供了 time-travel 时光旅行历史回滚功能

什么情况下使用 Vuex

- 非必要情况不要使用 Vuex
- 中大型的单页应用程序
  - 多个视图依赖于同一状态
  - 来自不同视图的行为需要变更同一状态

### Vuex 核心概念

- Store（仓库），每一个应用仅有一个 store, store 是一个容器，包含着应用中的大部分状态，我们不能直接改动 store 中的状态， 要通过提交 mutation 的方式改变状态
- State (状态)，保存在 store 中，因为 store 是唯一的，所以状态也是唯一的，称为单一状态树。但是所有的状态都保存在 state 中的话会导致程序难以维护，可以通过 Module (模块)解决该问题。注意：这里的状态是响应式的
- Getter 就像是 Vuex 中的计算属性，方便从一个属性派生出其他的值，它内部可以对计算的结果进行缓存，只有当依赖的状态发生改变的时候才会重新计算
- Mutation，状态的变化必须要通过提交 Mutation 来完成
- Action，和 Mutation 类似，不同的是 action 可以进行异步操作，内部改变状态的时候，都需要提交 mutation
- Module(模块)，由于使用单一状态树，应用的所有状态会集中到一个比较大的对象上来，当应用变得非常复杂时，store 对象很有可能变得相当臃肿，为了解决以上问题，Vuex 允许我们将 Store 分割成模块。每个模块拥有自己的 state, mutation,  action, getter,甚至是嵌套的子模块

### Vuex 基本代码结构

- 导入 Vuex
- 注册 Vuex
- 注入 $store 到 Vue 实例

### State

Vuex 使用单一状态树，用一个对象就包含了全部的应用层级状态

使用 mapState 简化 State 在视图中的使用，mapState 返回计算属性，mapState 有两种用法：

- 接受数组参数

  ```js
  // 该方法是 vuex 提供的，所以使用前要先导入
  import { mapState } from 'vuex'
  // mapState 返回名称为 count 和 msg 的计算属性，在模板中直接使用 count 和 msg
  computed: {
  	...mapState(['count', 'msg'])
  }
  ```

- 接收对象参数

  如果当前视图中已经存在了 count 和 msg，那么使用上述方式就会存在命名冲突，解决的方式就是使用对象参数

  ```js
  // 该方法是 vuex 提供的，所以使用前要先导入
  import { mapState } from 'vuex'
  // 通过传入对象，可以重命名返回的计算属性，在模板中直接使用 num 和 message
  computed: {
  	// ...mapState({
      //   num: state => state.count,
      //   message: state => state.message
      // })
      ...mapState({ num: 'count', message: 'msg' })
  }
  ```

### Getters

Getters 就是 store 中的计算属性，使用 mapGetters 简化视图中的使用

```js
export default new Vuex.Store({
  state: {
    count: 0,
    msg: 'Hello Vuex'
  },
  getters: {
    reverseMsg (state) {
      return state.msg.split('').reverse().join('')
    }
  }
})
```

```js
import { mapGetters } from 'vuex'

computed: {
	...mapGetters(['reverseMsg']),
	// 改名，在模板中使用 reverse
	...mapGetter({
		revers: 'reverseMsg'
	})
}
```

### Mutation

更改 Vuex 的 store 中的状态的唯一方法是提交 mutation。Vuex 中的 mutation 非常类似于事件：每个 mutation 都有一个字符串的**事件类型（Type）**和一个**回调函数（handler）**。这个回调函数就是实际进行状态更改的地方，并且它会接受 state 作为第一个参数

使用 Mutation 改变状态的好处是，集中的一个位置对状态修改，不管在什么地方修改，都可以追踪到状态的修改。可以实现高级的 time-travel 调试功能

```js
export default new Vuex.Store({
  state: {
    count: 0,
    msg: 'Hello Vuex'
  },
  mutations: {
    increate (state, payload) {
      state.count += payload
    }
  }
})

```

```js
import { mapMutations } from 'vuex'

methods: {
	...mapMutations(['increate']),
	// 传对象解决重名问题
	...mapMutations({
		increateMut: 'increate'
	})
}
```

Mutation 中只能执行同步操作，使用 commit 来调用 Mutation

### Action

Action 类似于 mutaion,不同在于：

- Action 提交的是 mutation，而不是直接变更状态
- Action 可以包含任意的以部操作

```js
import { mapActions } from 'vuex'

methods: {
	...mapActions(['increateAsync'])
	// 传对象解决重名问题
	...mapActions({ incAsync: 'increateAsync' })
}
```

如果不使用 mapActions 可使用 dispatch 来调用 Action

### Module

由于使用单一状态树，应用的所有状态会集中到一个比较大的对象上来，当应用变得非常复杂时，store 对象很有可能变得相当臃肿，为了解决以上问题，Vuex 允许我们将 Store 分割成模块。每个模块拥有自己的 state, mutation,  action, getter,甚至是嵌套的子模块

### Vuex 的严格模式

不要在生产环境下使用严格模式，严格模式会深度检查不合规的状态改变，会影响性能。

### Vuex 模拟实现

#### 实现思路

- 实现 install 方法
  - Vuex 是 Vue 的一个插件，所以和模拟 VueRouter 类似，先实现 Vue 插件约定的 install 方法
- 实现 store 类
  - 实现构造函数，接收 options
  - state 的响应化处理
  - getter 处理
  - commit，dispatch 方法

#### install 方法

```js
let _vue = null
function install (Vue) {
    _Vue = Vue
    _VUe.mixin({
        beforeCreate() {
            if (this.$options.store) {
                Vue.prototype.$store = this.$options.store
            }
        }
    })
}
```

#### Store 类

```js
class Store {
  constructor(options) {
    const {
      state = {},
      getters = {},
      mutations = {},
      actions = {}
    } = options

    this.state = _Vue.observable(state)
    // 此处不直接 this.getters = getters，是因为下面的代码中要方法 getters 中的 key
    // 如果这么写的话，会导致 this.getters 和 getters 指向同一个对象
    // 当访问 getters 的 key 的时候，实际上就是访问 this.getters 的 key 会触发 key 属性的 getter
    // 会产生死递归
    this.getters = Object.create(null)
    Object.keys(getters).forEach(key => {
      Object.defineProperty(this.getters, key, {
        get: () => getters[key](state)
      })
    })
    this._mutations = mutations
    this._actions = actions
  }

  commit (type, payload) {
    this._mutations[type](this.state, payload)
  }

  dispatch (type, payload) {
    this._actions[type](this, payload)
  }
}

// 导出模块
export default {
  Store,
  install
}

```

#### 使用自己实现的 Vuex

```js
import Vuex from '../myvuex'
// 注册插件
Vue.use(Vuex)
```

## 服务端渲染基础

### 概述

SPA 单页面应用

优点：

- 用户体验好
- 开发效率高
- 渲染性能好
- 可维护性好等

缺点：

- 首屏渲染时间长

  于传统服务端渲染直接获取服务端渲染好的 HTML 不同，单页面应用使用 JavaScript 在客户端生成 HTML 来呈现内容，用户需要等待客户端 JS 解析执行完成才能看到页面，这就使得首屏加载时间变长，从而影响用户体验

- 不利于 SEO

  当搜索引擎爬取网站 HTML 文件时，单页应用的 HTML 没有内容，因为它需要通过客户端 JavaScript 解析执行才能生成网页内容，而目前主流的搜索引擎对于这一部分内容的抓取还不是很好

为了解决以上两个缺陷，业界借鉴了传统的服务端直出 HTML 方案，提出在服务端执行前端框架（React/Vue/Angular）代码生成网页内容，然后将渲染好的网页内容返回给客户端，客户端只需要负责展示就可以了。当然不仅如此，为了获得更好的用户体验，同时会在客户端将来自服务端渲染的内容激活为一个 SPA 应用，也就是说之后的页面内容交互都是通过客户端渲染处理

同构应用

	- 通过服务端渲染首屏直出，解决 SPA  应用首屏渲染慢以及不利于 SEO 问题
	- 通过客户端渲染接管页面内容交互得到更好的用户体验

这种方式通常称之为现代化的服务端渲染，也叫同构渲染，这种方式构建的应用称之为服务端渲染应用或者是同构应用

相关概念

- 什么是渲染
- 传统的服务端渲染
- 客户端渲染
- 现代化的服务端渲染（同构渲染）

### 什么是渲染

渲染：把 【数据】 + 【模板】拼接到一起

渲染的本质其实就是字符串的解析替换，实现方式有很多种，但目前我们要关注的不是如何渲染，而是在哪里渲染

### 传统的服务端渲染

早期的 Web 页面渲染都是在服务端进行的。服务端运行过程中将所需的数据结合页面模板渲染为 HTML，响应给客户端浏览器，所以浏览器呈现出来的是直接包含内容的页面

在网页越来越复杂的情况下，存在很多不足

- 前后端代码完全耦合在一起，不利于开发和维护
- 前端没有足够发挥空间，无法充分利用现在前端生态下的一些更优秀的方案
- 由于内容都是在服务端动态生成的，服务端压力大
- 相比目前流行的 SPA 应用来说。用户体验一般

但在网页应用并不复杂的情况下，这种方式也是可取的。

### 客户端渲染（Client Side Renderer）

传统的服务端渲染存在的很多问题随着客户端 Ajax 技术的普及得到了有效的解决。Ajax 技术可以使得客户端动态获取数据成为可能，也就是说原本服务端渲染这件事也可以拿到客户端做了。

我们可以把【数据处理】和【页面渲染】分开，后端来负责数据的处理，前端负责页面渲染，这种分离模式极大提高了开发效率和可维护性。这样一来，前端更为独立，也不再受限于后端。可以选择任意的技术方案或框架来处理页面渲染

但这种模式下，存在一些明显的不足：

- 首屏渲染慢：因为 HTML 中没有内容，必须等到 JavaScript 加载并执行完成才能呈现页面内容
- SEO 问题：同样因为 HTML 中没有内容，所以对于目前的搜索引擎爬虫来说，页面中没有任何有用的信息，自然无法提取关键词进行索引

对于客户端渲染的 SPA 应用的问题有没有解决方案呢？

- 服务端渲染，严格来说是现代化的服务端渲染，也叫同构渲染

### 现代化的服务端渲染——同构渲染

同构渲染 = 后端渲染 + 前端渲染

- 基于 React、Vue 等框架，客户端渲染和服务端渲染的结合
  - 在服务器端执行一次，用于实现服务器端渲染（首屏指出）
  - 在客户端再执行一次，用于接管页面交互
- 核心解决 SEO 和首屏渲染慢的问题
- 拥有传统服务端渲染的优点，也有客户端渲染的优点

如何实现同构渲染？

- 使用 Vue、React 等框架的官网解决方案
  - 优点：有助于理解原理
  - 缺点：需要搭建环境，比较麻烦
- 使用第三方解决方案
  - React 生态的 Next.js
  - Vue 生态的 Nuxt.js
  - Angular 生态的 Angular Universal

同构渲染应用的问题：

- 开发条件有限
  - 浏览器特定的代码只能在某些生命周期钩子函数中使用
  - 一些外部扩展库可能需要特殊处理才能在服务端渲染应用中运行
  - 不能在服务端渲染期间操作 DOM
  - 某些代码操作需要区分运行环境等
- 涉及构建设置和部署的更多要求
  - 客户端渲染只需要构建客户端应用即可，可以部署在任意 Web 服务器中
  - 同构渲染需要构建两个端，只能部署在 Node.js Server
- 更多的服务器端负载
  - 在 Node 中渲染完整的应用程序，相比仅仅提供静态文件的服务器 需要大量占用 CPU 资源
  - 如果应用在高流量环境下使用，需要准备相应的服务器负载
  - 需要更多的服务端渲染优化工作处理

服务端渲染使用建议

- 首屏渲染速度是否真的重要？
- 是否真的需要 SEO？

## NuxtJS 基础

### NuxtJS 介绍

Nuxt.js 是什么

- 一个基于 Vue.js 生态的第三方开源服务端渲染应用框架
- 它可以帮我们轻松的使用 Vue.js 技术栈构建同构应用
- 官网：https://zh.nuxtjs.org/
- GitHub 仓库地址：https://github.com/nuxt/nuxt.js

### 初始化 NuxtJS 项目

Nuxt.js 的使用方式

- 初始项目
- 已有的 Node.js 服务端项目
  - 直接把 Nuxt 当作一个中间件集成到 Node Web Server 中
- 现有的 Vue.js 项目
  - 非常熟悉 Nuxt.js
  - 至少百分之 10 的代码改动

初始化 Nuxt.js 应用的方式

- 官方文档 https://zh.nuxtjs.org/docs/2.x/get-started/installation
  - 方式一：使用 create-nuxt-app
  - 方式二：手动创建

手动创建项目过程：

1. 准备

   ```bash
   # 创建项目
   mkdir nuxt-app-demo
   # 进入项目木木中
   cd nuxt-app-demo
   # 初始化 package.json 文件
   npm init -y
   # 安装 nuxt
   npm install nuxt
   ```

   在 ```package.json``` 文件中的 ```scripts``` 中新增

   ```json
   "scripts": {
   	"dev": "nuxt"
   }
   ```

   这样使得我们可以通过运行 ```npm run dev``` 来运行 ```nuxt```

2. 创建页面并启动项目

   创建```pages``` 目录

   ```bash
   mkdir pages
   ```

   创建第一个页面 ```pages/index.vue```

   ```html
   <template>
     <div>
       <h1>Hello Nuxt.js</h1>
     </div>
   </template>
   ```

   然后启动项目

   ```bash
   npm run dev
   ```

   应用运行在 http://localhost:3000/ 上

   > 注意：Nuxt.js 会监听 pages 目录中的文件更改，因此在添加新页面时无需启动应用程序


### Nuxt 路由

Nuxt.js 依据 ```pages```目录结构自动生成 vue-router 模块的路由配置

基础路由：https://zh.nuxtjs.org/docs/2.x/get-started/routing

1. Nuxt 中的基础路由

   Nuxt.js 会根据 pages 目录中的所有 ```.vue``` 文件生成应用的路由配置

   假设 pages 的目录结构如下：

   ```
   pages/
   --| user/
   -----| index.vue
   -----| one.vue
   --| index.vue
   ```

   那么，Nuxt.js 自动生成的路由配置如下：

   ```js
   router: {
     routes: [
       {
         name: 'index',
         path: '/,
         component: 'pages/index.vue'
       },
       {
         name: 'user',
         path: '/user',
         component: 'pages/user/index.vue'
       },
       {
         name: 'user-one',
         path: '/user/one',
         component: 'pages/user/one.vue
       }
     ]
   }
   ```

2. 路由导航

   - a 标签——会刷新整个页面，走服务端渲染，不要使用
   - ```<nuxt-link>``` 组件——https://zh.nuxtjs.org/docs/2.x/features/nuxt-components/#the-nuxtlink-component
   - 编程式导航——https://zh.nuxtjs.org/docs/2.x/get-started/routing#navigation
   
3. 动态路由

   - Vue Router 动态路由匹配
     - https://router.vuejs.org/zh/guide/essentials/dynamic-matching.html
   - Nuxt.js 动态路由
     - https://zh.nuxtjs.org/docs/2.x/features/file-system-routing#dynamic-routes

4. 嵌套路由

   - Vue Router 嵌套路由
     - https://router.vuejs.org/zh/guide/essentials/nested-routes.html
   - Nuxt.js 嵌套路由
     - https://zh.nuxtjs.org/docs/2.x/features/file-system-routing#nested-routes

5. 路由配置

   - 参考文档：https://zh.nuxtjs.org/docs/2.x/configuration-glossary/configuration-router/

### NuxtJS 视图

视图模板

- https://zh.nuxtjs.org/docs/2.x/concepts/views

视图布局

- https://zh.nuxtjs.org/docs/2.x/concepts/views#layouts

### NuxtJS 异步数据—— asyncData 方法

如何在服务端渲染动态页面

- https://zh.nuxtjs.org/docs/2.x/features/data-fetching/#async-data
- 基本用法
  - 它会将 asyncData 返回的数据融合组件 data 方法返回数据一并给组件
  - 调用时机：服务端渲染期间和客户端路由更新之前
- 注意事项：
  - 只能在页面组件中使用
  - 没有 this,因为它是在组件初始化之前被调用的

### NuxtJS 异步数据——上下文对象

- https://zh.nuxtjs.org/docs/2.x/concepts/context-helpers/

## NuxtJS 综合案例

### 案例介绍

- 案例名称：RealWorld
- 一个开源的学习项目，目的就是帮助开发者快速学习新技能
- GitHub 仓库：https://github.com/gothinkster/realworld
- 在线示例：https://demo.realworld.io/#/
- 页面模板：https://github.com/gothinkster/realworld-starter-kit/blob/master/FRONTEND_INSTRUCTIONS.md
- 接口文档：https://github.com/gothinkster/realworld/tree/master/api

学习前提：

- Vue.js 使用经验
- Nuxt.js 基础
- Node.js、webpack 相关使用经验

学习收获：

- 掌握使用 Nuxt.js 开发同构渲染应用
- 增强 Vue.js 实践能力
- 掌握同构渲染应用中常见的功能处理
  - 用户状态管理
  - 页面访问权限处理
  - SEO 优化
  - ...
- 掌握同构渲染应用的发布与部署

## Nuxt.js 发布部署

### 打包 Nuxt.js 应用

- https://zh.nuxtjs.org/docs/2.x/get-started/commands/

| 命令          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| nuxt          | 启动一个热加载的 Web 服务器（开发模式）                      |
| nuxt build    | 利用 webpack 编译应用，压缩 JS 和 CSS 资源（发不用）         |
| nuxt start    | 以生产模式启动一个 Web 服务器（需要先执行 nuxt build）       |
| nuxt generate | 编译应用，并依据路由配置生成对应的 HTML 文件（用于静态站点的部署） |

### 最简单的部署方式

- 配置 Host + Port
- 压缩发布包
- 把发布包传到服务器
- 解压
- 安装依赖
- 启动服务

### 自动部署

CI/CD 服务

- Jenkins
- Gitlab CI
- GitHub Actions
- Travis CI
- Circle CI
- ...

环境准备

- Linux 服务器
- 把代码提交到 GitHub 远程仓库

配置 GitHub Access Token

- 生成：https://github.com/settings/tokens
- 配置到项目的 Secrets 中：https://github.com/cl-k/realworld-nuxtjs/settings/secrets/actions

配置 GitHub Actions 执行脚本

- 在项目根目录创建 .github/workflows 目录
- 下载 main.yml 到 workflows 目录中
- 修改配置
- 配置 PM2 配置文件
- 提交更新
- 查看自动部署状态
- 访问网站
- 提交更新...