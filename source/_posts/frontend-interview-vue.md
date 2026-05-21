---
title: 前端面试题-Vue
date: 2026-05-21 15:00:00
tags: [前端, 面试, Vue]
categories: 前端面试
---

## 生命周期

- Vue3/Vue2 生命周期钩子对比:
  - setup（beforeCreate之后/created之前）：在组件实例创建（beforeCreate是之前，created是创建完成）被调用，用于设置组件的响应式数据和方法。
  - onBeforeMount（beforeMount）：在挂载开始之前被调用，此时模板已经编译，但还未挂载到 DOM 上。
  - onMounted（mounted）：实例被挂载后被调用，此时 DOM 已经可用。
  - onBeforeUpdate（beforeUpdate）：数据更新时被调用，此时数据已经更新，但 DOM 还未更新。
  - onUpdated（updated）：数据更新后被调用，此时 DOM 已经更新。
  - onBeforeUnmount（beforeUnmount）：实例卸载之前被调用，此时实例仍然可用。
  - onUnmounted（unmounted）：实例卸载后被调用，此时实例已经被销毁。
  - onActivated（activated）：keep-alive 组件激活时被调用。
  - onDeactivated（deactivated）：keep-alive 组件停用时被调用。
  - onErrorCaptured：在捕获了后代组件传递的错误时调用。
  - onRenderTracked（Vue3）：当组件渲染过程中追踪到响应式依赖时调用。
  - onRenderTriggered（Vue3）：当响应式依赖的变更触发了组件渲染时调用。
  - onServerPrefetch（Vue3）：服务端渲染时被调用，用于预取数据。
- 高频面试题
  - 父子组件生命周期执行顺序
    - 挂载阶段：父onBeforeMount->子onBeforeMount->子onMounted->父onMounted
    - 更新阶段：父onBeforeUpdate->子onBeforeUpdate->子onUpdated->父onUpdated
    - 卸载阶段：父onBeforeUnmount->子onBeforeUnmount->子onUnmounted->父onUnmounted
  - 在setup()中如何访问this？
    - setup()中没有this，响应式数据通过ref/reactive定义，方法直接申明。
  - 异步请求放在哪个钩子？
    - 客户端渲染（CSR）:onMounted（确保DOM可用）
    - 服务端渲染（SSR）:setup（onServerPrefetch）或onMounted（根据需求）

## 组件通信

- 父向子：props、属性透传（$attr）、ref/expose（实例、DOM）。
- 子向父：$emit 触发事件。
- v-model：可以父向子、子向父。
- 兄弟：
  - 父组件作为中介
  - 使用全局事件总线（Event Bus）或状态管理库（如 Vuex）来实现跨组件通信。
- 跨层级通信：provide/inject、Vuex/Pinia、Event Bus（vueuse的useEventBus）。

## 事件修饰符

- 事件修饰符
  - .stop：阻止事件冒泡。
  - .prevent：阻止默认事件行为。
  - .capture：使用事件捕获模式。
  - .self：只当事件目标是当前元素时触发。
  - .once：事件只触发一次，之后自动移除监听器。
  - .passive：告诉浏览器事件监听器不会调用 preventDefault()，提高滚动性能。
- v-model 修饰符
  - .lazy：在 change 事件而不是 input 事件时更新数据。
  - .number：将输入值转换为数字。
  - .trim：去除输入值的首尾空格。
- 键盘事件修饰符
  - .enter：监听 Enter 键。
- 鼠标事件修饰符
  - .left：监听鼠标左键。
  - .right：监听鼠标右键。
  - .middle：监听鼠标中键。

## 指令

- v-show：通过切换元素的 display 样式来控制元素的显示和隐藏，元素始终存在于 DOM 中。
- v-if：根据条件渲染元素，元素在条件为 false 时被完全移除 DOM。
- v-for：用于渲染列表，根据提供的数据源生成多个元素。
  - key：必须提供唯一标识，优化虚拟DOM复用，避免渲染错误，不建议使用index除非列表不变。
  - 优先级：vue2：v-for高于v-if ，vue3：v-if高于v-for
- v-bind：动态绑定元素的属性或组件的 props。
- v-on：绑定事件监听器。
- v-model：实现双向数据绑定，常用于表单输入元素。
- v-slot：用于定义组件的插槽内容。
- v-pre：跳过元素和子元素的编译过程，直接输出原始内容。
- v-cloak：在 Vue 编译完成之前保持元素的可见性，常用于防止闪烁。
- v-once：只渲染元素和组件一次，之后不再更新。

## ref和reactive区别

| 特性         | `ref`                                         | `reactive`                          |
| ------------ | --------------------------------------------- | ----------------------------------- |
| 适用数据类型 | 基本类型（`string`/`number`/`boolean`）或对象 | 对象或数组                          |
| 访问方式     | 通过 `.value` 访问                            | 直接访问属性                        |
| 响应式原理   | 内部对对象类型调用 `reactive`                 | 基于 `Proxy` 的深层代理             |
| 模板自动解包 | 在模板中无需 `.value`                         | 直接使用属性                        |
| 解构响应性   | 需用 `toRefs` 保持响应性                      | 直接解构会丢失响应性，需用 `toRefs` |

**为什么ref需要.value?**

- 设计目的：统一处理基本类型和对象类型。
  - 基本类型无法通过Proxy代理，ref通过封装对象({ value:... })以及getter/setter 拦截 value实现响应式
  ```js
  function ref(value) {
    return {
      __v_isRef: true,
      get value() {
        track(this, "value");
        return value;
      },
      set value(newVal) {
        value = newVal;
        trigger(this, "value");
      },
    };
  }
  ```

**ref的响应式实现原理是什么？**

- 基本类型：通过 RefImpl 类实现，而 RefImpl 类内部使用了 JavaScript 原生的 getter/setter 语法（不是直接调用Object.defineProperty）。
- 对象类型：内部转化为reactive代理

**reactive的响应式实现原理是什么？**

- 基于Proxy代理整个对象，惰性递归（访问时才转换），对象的每个属性，拦截 get 和 set 操作。
- 依赖收集：在get时调用track收集依赖
- 触发更新：在set时调用trigger通知更新

**ref和reactive的性能差异?**

- 对象类型：reactive性能更好，ref需要内部转化为reactive代理（性能差异不大）。

## 插槽

- 作用：插槽（Slot）是 Vue 提供的一种机制，用于在组件中预留位置，让父组件可以向子组件传递内容，实现组件的灵活组合和复用。
- 分类：
  - 默认插槽：没有名称的插槽，父组件传递的内容会被渲染在子组件的默认插槽位置。
  - 具名插槽：通过 name 属性命名的插槽，父组件可以通过 v-slot 指令指定内容渲染到对应的具名插槽位置。
  - 作用域插槽：允许子组件在插槽内容中暴露数据给父组件的机制。
- 动态插槽名（Vue3）：

```vue
<template #[dynamicSlotName]>动态插槽内容</template>
```

## watch、watchEffect、computed的区别

| 特性       | `computed`                         | `watch`                                | `watchEffect`                        |
| ---------- | ---------------------------------- | -------------------------------------- | ------------------------------------ |
| 用途       | 派生响应式数据                     | 监听数据变化，执行副作用操作           | 自动收集依赖，执行副作用操作         |
| 返回值     | 只读的 `Ref` 对象                  | 返回停止监听的函数                     | 返回停止监听的函数                   |
| 依赖收集   | 自动收集依赖，惰性计算（缓存结果） | 需显式指定监听源                       | 自动收集依赖，立即执行               |
| 新旧值获取 | 无                                 | 可获取旧值和新值                       | 无旧值，只跟踪最新值                 |
| 执行时机   | 依赖变化时重新计算                 | 默认在组件更新前执行（`flush: 'pre'`） | 默认在组件更新前执行（类似 `watch`） |
| 异步处理   | 不支持                             | 支持异步操作                           | 支持异步操作                         |
| 默认是否立即执行 | 懒执行（被用才会）                                 | 否                                     | 是                                     |

**computed返回异步会发生什么？**

- 返回字符串类型：[object Promise]

**如何停止watch或watchEffect的监听？**

- watch：返回一个停止监听的函数，调用该函数即可停止监听。

```js
const stop = watch(
  () => state.count,
  (newVal, oldVal) => {},
);
// 停止监听
stop();
```

## keep-alive

- include：字符串或正则表达式以及数组，匹配的组件会被缓存。
- exclude：字符串或正则表达式以及数组，匹配的组件不会被缓存。
- max：数字，缓存组件的最大数量，超过后会按照 LRU（最近最少使用）策略移除。

**keep-alive实现原理？**

- 缓存机制：通过Map或Object缓存组件vnode实例，渲染时直接从缓存中取。
- DOM处理：
  - 该组件实例对应的整个 DOM 树会被从真实的文档流 (DOM tree) 中完全移除 (detached) 。这就是为什么在页面检查器里看不到它的 DOM 了。
  - 当这个组件再次被激活时，keep-alive 会从缓存中找到这个实例，直接复用这个组件实例（保留所有状态），并将它对应的 DOM 树重新插入 (attached) 到文档流中。

**注意事项**

- 组件必须设置name选项：否则include/exclude无法匹配
- 避免内存泄漏：及时清理不需要缓存的组件（如通过max或动态include）
- SSR不兼容：keep-alive仅在客户端渲染中生效
- 缓存组件的状态保留：表单内容等会被保留，需手动重置或通过key强制更新

## 异步组件

**原理**

底层基于 ES Module 的动态 import()，打包时会进行代码分割（code splitting），生成独立 chunk。组件真正渲染时才会加载对应 JS 文件，从而减少首屏资源体积。
```js
const AsyncComp = defineAsyncComponent({
  loader: () =>
    import("./MyComponent.vue").catch(() => {
      // 重试逻辑
      return retryImport();
    }),
  loadingComponent: loadingSpinner, // 加载中组件
  errorComponent: ErrorDisplay, // 错误组件
  delay: 1000, // 延迟显示loading（防闪烁）
  timeout: 3000, // 超时时间
});
```

**Suspense组件**

- 统一管理异步组件（如异步组件或异步setup函数）：
  - #default 插槽：异步组件加载完成后显示的内容
  - #fallback 插槽：加载中显示的占位内容

```vue
<template>
  <Suspense>
    <template #default>
      <AsyncComp />
    </template>
    <template #fallback>
      <LoadingSpinner />
    </template>
  </Suspense>
</template>
```

**异步组件的核心作用**

- 按需加载：减少初始包体积，提升首屏加载速度
- 性能优化：结合代码分割（Code Spliting）动态加载非关键组件

## Vue-Router

**创建路由实例**

```js
import { createRouter, createWebHistory } from 'vue-router';
const router = createRouter({
    history: createWebHistory(),  // 或 createWebHashHistory
    routes: [...]
});

```

**组合式API支持**

- useRouter()：获取路由实例（替代this.$router）
- useRoute()：获取当前路由对象（替代this.$route）

**动态路由**

```js
{ path: '/user/:id', component: User };
// 获取参数：route.params.id
```

**嵌套路由**

```js
{
    path: '/parent',
    component: Parent,
    children: [
        { path: 'child', component: Child }
    ]
}

```

**命名路由与编程式导航**

```js
router.push({ name: "user", params: { id: 1 } });
```

**路由模式**

- createWebHistory()：History模式（需服务器支持）
- createWebHashHistory()：Hash模式
- createMemoryHistory()：SSR或测试环境
- history 模式：使用 HTML5 History API，URL 不带 #，需要服务器配置支持。
- hash 模式：URL 带 #，不需要服务器配置，兼容性更好。

**重定向与别名**

```js
{ path: '/home', redirect: '/' }
{ path: '/', alias: 'home' }
```

## 导航守卫

**全局守卫**

- beforeEach：路由跳转前
- afterEach：路由跳转后
- beforeResolve：路由确认前

**路由独享守卫**

- beforeEnter：路由进入前

**组件内守卫**

- Options API
  - beforeRouteEnter：路由进入前，访问不到this
  - beforeRouteUpdate：当前路由组件复用、路由参数等发生变化时触发，可以访问this
  - beforeRouteLeave：离开当前路由时触发，可以访问this
- Composition API
  - onBeforeRouteUpdate：当前路由组件复用、路由参数等发生变化时触发，访问不到this
  - onBeforeRouteLeave：离开当前路由时触发，访问不到this

**高级特性与最佳实践**

- 路由懒加载

  ```js
  const User = () => import("./User.vue");
  const User = defineAsyncComponent(() => import("./User.vue"));
  ```

- 路由元信息（meta）

  ```js
  { path: '/profile', meta: { requireAuth: true } }
  // 在导航守卫中访问：to.meta.requiresAuth
  ```

- 动态路由
  - 添加路由：router.addRoute({ path: '/new', component: New })
  - 删除路由：router.removeRoute('route-name')

**Vue-Router 4.X 与 3.X 的主要区别**

- API命名调整（如new VueRouter() -> createRouter()）
- 组合式API支持（useRouter/useRoute）
- 动态路由API优化（addRoute/removeRoute）

## 状态管理

**Vuex**

- state：存储应用的状态数据。
- getters：计算属性，基于 state 派生出新的数据。
- mutations：同步修改 state 的方法，必须是同步函数。
- actions：可以包含异步操作的方法，提交 mutations 来修改 state。
- modules：将 store 分割成模块，每个模块拥有自己的 state、getters、mutations 和 actions。

**Pinia**

- 无Mutations：直接通过Actions处理同步/异步逻辑
- 扁平化结构：多个Store代替嵌套模块，更易维护
- TypeScript支持：自动推导类型，无需额外配置
- Devtools集成：支持时间旅行调试和状态快照
- 轻量高效：体积更小，API更简洁

**Pinia如何支持TypeScript？**

- Store定义自动推导类型，组件中通过store.xxx直接获得类型提示

**Vuex与pinia的区别**

- Pinia是Vue官方推荐的新状态管理库，支持Composition API和TypeScript
- 核心差异：
  - Pinia无mutations，直接通过actions修改状态（同步/异步均可）
  - Pinia基于模块化设计（每个Store独立），无需嵌套模块
  - 更简洁的API和TypeScript支持

## 性能优化

### 组件级优化策略

**渲染控制**

- v-memo：告诉 Vue 只有在依赖变化时才重新渲染这个节点或组件
- v-show：高频切换（CSS显示隐藏）
- v-if：低频切换（销毁/重建组件）

**组件设计优化**

- 细粒度拆分：隔离高频更新组件（通过分析组件的状态和更新频率，将高频更新的部分拆分成独立组件，减少不必要的渲染），
  怎么分析？可以通过Vue Devtools观察组件的更新频率和性能瓶颈，识别出哪些组件频繁更新，哪些组件状态变化较大，从而决定如何拆分组件。
- 异步组件：延迟加载非关键组件

### 状态管理优化

- shallowRef/shalloReactive：非深度响应式数据
  - shallowRef: 只对 最外层对象本身是响应式的，内部属性不会被递归响应化。一些大对象且不需要关心其内层，比如webSocket
- 使用markRaw避免不必要的响应式

### 资源与加载优化

**代码分割：按需加载组件和库**

- 路由级懒加载（Vue Router）
- 组件级懒加载（defineAsyncComponent）
- 第三方库按需加载：如lodash的单函数引入（import { debounce } from 'lodash'）

**Tree Shaking支持：移除未使用的代码，减少包体积**

- 使用ES模块语法（ESM）
- 避免副作用导入（导入就直接执行的文件）
- package.json "sideEffects": false 帮助打包工具 tree-shaking

**预加载关键资源**

```html
<link rel="preload" href="critical.js" as="script" />
<link rel="preload" href="critical.css" as="style" />
```

### 运行时性能优化

**列表渲染优化**

- v-for使用唯一key：优化虚拟DOM diff算法，减少不必要的重渲染。
- 虚拟列表：仅渲染可视区域内的列表项，提升性能。
- 避免v-for与v-if共用（优先用computed过滤数据）。

**计算与侦听优化**

- computed：缓存计算结果，避免重复计算。
- 避免深度监听大型对象

**事件处理优化**

- 高频事件使用防抖/节流：减少事件处理频率，提升性能。
- 使用事件修饰符（如.passive）提升滚动性能。

### 架构级优化

**服务端渲染（SSR）**

- 提升首屏加载速度和SEO表现，适合内容丰富的应用。

**静态站点生成（SSG）**

- 预渲染静态页面，适合博客、文档等内容较少变动的站点。

**CDN与缓存策略**

- 静态资源添加Content Hash：app.3a88b9e2.js # 文件名包含hash
- 设置长期缓存(Nginx)：

```js
location /assets {
    expires 1y;
    add_header Cache-Control "public";
}

```

### 工具链优化

**现代构建工具**

- Vite：基于ESM的快速构建工具，开发环境启动快，
- 生产构建优化：

  ```js
  // vite.config.js
  export default {
    build: {
      minify: "terser", // 代码压缩
      brtliSize: true, // 压缩分析
      chunkSizeWarningLimit: 1000, // 调整块大小警告
    },
  };
  ```

**性能分析工具**

- Chrome DevTools Performance 面板
- Vue DevTools 性能追踪
- Lighthouse 性能评分

### 面试

**Vue3比Vue2快在哪里？**

- 响应式系统：Vue3基于Proxy实现更高效的响应式系统，减少了性能开销。
- 虚拟DOM优化：Vue3的虚拟DOM算法更高效，减少了不必要的渲染和更新。
- Tree Shaking支持：Vue3支持Tree Shaking，未使用的功能不会被打包进最终代码，减小了包体积。
- 组件设计优化：Vue3引入了Composition API，允许更灵活的组件设计和状态管理，提升了性能。

**v-memo的使用场景（避免执行render）**

- 大型列表渲染优化
- 复杂表单优化
- 重复渲染的子组件

## Vue2和Vue3区别

### 架构设计区别

| 特性           | Vue2                    | Vue3              | 优势                                |
| -------------- | ----------------------- | ----------------- | ----------------------------------- |
| **响应式系统** | `Object.defineProperty` | `Proxy`           | 支持动态属性/数组索引监听，性能更优 |
| **代码组织**   | Options API             | Composition API   | 逻辑复用更灵活，类型推导更友好      |
| **源码组织**   | Flow 类型系统           | TypeScript 重写   | 更好的类型支持和源码可维护性        |
| **包体积**     | 全量引入（22.5kb）      | 按需引入（<10kb） | Tree Shaking 减少 41% 体积          |

### 响应式系统升级

- Vue2：基于 `Object.defineProperty` 实现响应式，无法监听新增属性和数组索引。
- Vue3：基于 `Proxy` 实现响应式，支持监听对象的所有属性和数组索引，性能更优。
  - 原理：Vue3使用Proxy代理（惰性递归）整个对象，拦截get/set操作，实现响应式。相比Vue2的defineProperty需要递归处理每个属性，Proxy更高效。

### Composition API vs Options API

- Options API：通过data、methods、computed等选项组织代码，适合小型组件，但逻辑复用较困难。
- Composition API：通过setup函数组织代码，使用ref/reactive定义响应式数据，适合大型组件，逻辑复用更灵活。
  - 优势：更好的逻辑复用（通过组合函数），更好的类型推导（TypeScript支持），更清晰的代码结构（将相关逻辑放在一起）。

### 性能优化对比

| 优化点               | Vue2                               | Vue3                                       |
| -------------------- | ---------------------------------- | ------------------------------------------ |
| **响应式系统**       | `Object.defineProperty`，性能较低  | `Proxy`，性能更优                          |
| **虚拟DOM算法**      | 基于简单的diff算法                 | 基于更高效的diff算法，减少不必要的渲染     |
| **Tree Shaking支持** | 不支持，整个库被打包               | 支持，未使用的功能不会被打包，减小包体积   |
| **组件设计优化**     | Options API，逻辑复用较困难        | Composition API，逻辑复用更灵活            |
| **静态提升**         | 不支持，所有节点每次渲染都需要处理 | 支持，静态节点提升到编译时，减少运行时开销 |

### 新特性与API

- **Teleport**：允许将组件渲染到 DOM 的任意位置，提升灵活性。
- **Suspense（异步组件）**：统一管理异步组件的加载状态，提升用户体验。
- **Fragments（碎片）**：组件可以返回多个根节点，减少不必要的 DOM 层级。

### 面试

**Vue3的模版编译优化有哪些？**

- 静态节点提升：不会变化的节点只创建一次，避免重复渲染。
- 静态属性优化：静态属性（class、id、style 等）在编译阶段确定，不在运行时计算。
- 事件处理缓存：不依赖响应式数据的事件处理函数只创建一次，减少函数重建开销。

**Vue2项目如何升级Vue3？**

- 使用@/vue/compat过渡
- 逐步替换废弃API（$children,filters等）
- 优先迁移新组件，逐步重构旧组件

## SPA

**定义**：整个应用只有一个HTML文件，通过动态替换DOM内容实现"页面"切换。
**优点**：无刷新跳转，前后端分离开发，减轻服务器渲染压力。
**缺点**：首屏加载慢，SEO不友好，路由管理复杂度

### SPA与MPA对比

| 特性           | SPA                          | MPA (多页面应用)     |
| -------------- | ---------------------------- | -------------------- |
| **页面数量**   | 1 个 HTML                    | 多个 HTML            |
| **页面跳转**   | 前端路由控制，无刷新         | 整页刷新             |
| **数据请求**   | Ajax/Fetch 局部获取数据      | 每次跳转加载完整页面 |
| **开发复杂度** | 高（需前端路由、状态管理等） | 低                   |
| **SEO 支持**   | 差（需额外优化）             | 优                   |

### 面试

**SPA首屏加载优化有哪些方案？**

- 路由懒加载 + 组件懒加载
  - 通过Vue Router的动态import实现路由级懒加载，通过defineAsyncComponent实现组件级懒加载，减少首屏包体积。
- 资源预加载/预取：
  - 使用<link rel="preload">预加载关键资源，使用<link rel="prefetch">预取未来可能需要的资源。
- CDN加速静态资源：
  - 将静态资源部署到CDN，利用其全球分发网络提升加载速度。
- 开启Gzip/Brotli压缩
  - 在服务器（如Nginx）配置启用Gzip或Brotli压缩，减少传输数据量。
  ```js
  // 动态 gzip（不需要 Vite）
  location / {
      gzip on; // 开启 gzip 压缩
      gzip_comp_level 6; // 压缩级别，1-9，6 是一个常用的平衡点
      // 压缩特定类型的文件
      gzip_types text/plain text/css application/javascript application/json image/svg+xml;
      // 让浏览器识别是否支持 gzip
      gzip_vary on;
  }
  ```
- 服务端渲染（SSR）

**如何解决SPA的SEO问题？**

- 预渲染（Prerender）：在构建时生成静态HTML文件，适合内容较少变动的页面。
- 服务端渲染（Nuxtjs）：在服务器端渲染完整的HTML，适合内容丰富且频繁更新的应用。
- 动态渲染（针对爬虫单独处理）：
  - 使用User-Agent检测爬虫，返回预渲染的HTML内容。
  - 适合需要兼顾SEO和用户体验的应用。
- 静态站点生成（SSG）：
  - 预渲染所有页面为静态HTML，适合博客、文档等内容较少变动的站点。

## 其他

### nextTick

- Vue 的 数据更新是异步的，nextTick用于在 DOM 更新完成后执行回调函数。它确保在数据变化后，所有相关的 DOM 更新都已经完成，适合在更新后访问或操作 DOM。
- 使用场景：
  - 在数据更新后需要访问更新后的 DOM 元素。
  - 在组件更新后执行某些操作，如滚动到特定位置。

### Hook 和工具函数怎么区分

```
“我的划分标准是看它是否依赖 Vue 的响应式能力和生命周期。依赖 ref、computed、watch、onMounted 这类能力的，我会抽成 Hook；如果只是纯函数计算、格式化、转换、对象处理，就放到工具包里。这样职责更清晰，也方便测试和复用。”
```
**withDefaults**
- props 自动有默认值
- TS 类型更精确
- 代码更简洁可读