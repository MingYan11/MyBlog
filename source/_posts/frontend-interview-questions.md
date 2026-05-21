---
title: 前端面试题整理
date: 2026-05-21 15:00:00
tags: [前端, 面试, JavaScript, Vue, CSS, HTML, TypeScript, 工程化]
categories: 前端
---

# 项目

## Spinman 后台管理系统

### 世界地图

```
因为后台只返回国家code，所以前端使用ISO获取国家全名，在调用链接去获取国家经纬度。
```

### monorepo（单一代码仓库）

```
把多个相关的前端项目或包（Package）放在同一个仓库管理，而不是每个项目独立一个仓库（Polyrepo）。
```

## Spinman 游戏平台

### hook

#### socket

```js
// wssUrl: wss://api.spinman.app
// wsApiPath: /game-zone-a/nest-ws-main-api
// socket.io 是 WebSocket 的实时通信库
function useCreateSocket(namespace?: string) {
    function createSocket(namespace?: string) {
      // 初始化 socket 实例
      return io(`${wssUrl}/${namespace}`, {
        auth: {
          adminToken: ''
        },
        path: wsApiPath,
        transports: ['websocket']
      });
  }

  /**
   * 创建后台余额管理 socket
   */
  function createNotifySocket() {
    return createSocket(WssNamespace.AdminBalance) as AdminBalanceSocket;
  }
}
```

```js
// 示例
const socket = shallowRef<AdminBalanceSocket | null>(null);
socket.value = createNotifySocket();
```

#### queue

```js
export function useQueue(
    callback: (item:T) => void,
    maxLoadNum = 1,
    maxSizeNum = 200,
    interval = 200,
    )
  {
    const queue = ref<T[]>()
    const { pause, resume, isActive } = useIntervalFn(() => {
      if(queue.value.length <= 0) {
        pause()
        return
      }
      const batch = queue.value.splice(0, Math.min(queue.value.length, maxLoadNum))
      batch.forEach((item:T) => callback?.(item))
    },
    interval,
    { immediate: false })

    function push(data:T | T[]) {
      if(!data || (Array.isArray(data) && data.length === 0))
        return
      if(Array.isArray(data)) {
        queue.value.push(...data)
      } else {
        queue.value.push(item)
      }

      if(!isActive)
        resume()

      if(queue.value.length > maxSizeNum) {
        queue.value = queue.value.slice(-Math.floor(maxSizeNum / 2))
      }
    }
}
```

```js
//调用示例：
const myBets = ref();
const myBet = useQueue((item) => updateList(item));
function updateList(item) {
  myBet.value.unshift(item);

  setTimeout(() => {
    myBet.value.pop();
  }, 450);
}
```

#### 分页 usePaginationList

```js
export function usePaginationList(
  getListRequest,
  options
) {
  const {
    transformList,
    transformItem,
    defaultPageSize: _defaultPageSize,
    isAutoLoad = true,
    mode = 'page',
    search = {},
    rateLimitDelayMs = 300
  } = options

  const defaultSearch = JSON.pare(JSON.stringify(search))

  // 是否自适应页面大小，页面变化时掉整pagesize
  const isAdaptivePageSize = typeof defaultPageSize === 'function'
  const finallyPageSize = computed(() =>{
    if(!_defaultPageSize) {
      return 10
    }
    if(typeof _defaultPageSize !== 'number') {
      const pageSize = toValue(_defaultPageSize)
      if(typeof pageSize !== 'number') {
        return 10
      }
      return _defaultPageSize
    }
    return _defaultPageSize
  })

  const params = ref({
    currentPage: 1,
    pageSize: finallyPageSize.value
  })

  const list = ref()
  const loading = ref(false)
  const total = ref(0)
  const totalPages = ref(0)

  const onDebounceSearch = useDebounceFn(onSearch, rateLimitDelayMs)
  const onThrottleSearch = useDebounce(onSearch, rateLimitDelayMs, true)
  const onDebounceReset = useDebounceFn(onReset, rateLimitDelayMs)
  const onThrottleReset = useDebounce(onReset, rateLimitDelayMs, true)

  const hasPrePage = computed(() => params.value.currentPage > 1)
  const hasNextPage = computed(() => params.value.currentPage > totalPages.value )

  if(isAutoLoad) {
    if(isAdaptivePageSize) {
      watch(() => finallyPageSize.value,() => {
        onSearch()
      },{ immediate:true })
    } else {
      getList()
    }
  }



  async function getList(
    {
      shouldResetList = false
    }
  ) {
    loading.value = true
    try {
      const normalizedParams = {
        ...toValue(params),
        ...toValue(search)
      }
      const { data, code } = await getListRequest(normalizedParams)

      if(data && code === 0) {
        if(transformList) {
          data.list = transformList.(data.list)
        }
        if(transformItem) {
          data.list = data.list.map(item => transformItem(item))
        }
        if(toValue(mode) === 'append' && !shouldResetList) {
          list.value.push(data.list)
        } else {
          list.value = data.list
        }
        totalPages.value = data?.total ?? data?.total ? Math.ceil(data?.total / finallyPageSize.value) : 0
        total.value = data?.total || 0
      }
    } catch(error) {
      console.error(error)
    } finally {
      loading.value = false
    }
  }

  function resetSearchParams() {
    if(isRef(search)) {
      if(isReadonly(search)) {
        console.warn('')
        return
      }
      search.value = defaultSearch
    }
  }

  function resetParams() {
    params.value = {
      currentPage: 1,
      pageSize: finallyPageSize.value
    }
  }

  function onSearch() {
    resetParams()
    getList({
        shouldResetList:true
      })
  }

  function onReset() {
    resetParams()
    resetSearchParams()
    getList({
      shouldResetList:true
    })
  }

  function toPrePage() {
    if(!hasPrePage)
      return
    params.value.currentPage -= 1
    getList()
  }

  function toNextPage() {
    if(!hasNextPage)
      return
    params.value.currentPage += 1
    getList()
  }

  return {
    total,
    totalPages,
    list,
    getList,
    onSearch,
    onReset,
    loading,
    toPrePage,
    toNextPage,
    onDebounceSearch,
    onThrottleSearch,
    onDebounceReset,
    onThrottleReset
  }
}
```

### 游戏

```
游戏封装为为游戏容器、控制区域（下注、难度选择）、面板区域（游戏主区域）、结果区域
状态：游戏准备中、游戏开始（点了下注按钮）、游戏进行中、游戏结束、游戏暂停（某些游戏有）
动画：css、gif、精灵动画、动画库（anime.js）
音频：使用pixi.js的Sound类封装了一个hook
```

**游戏优化**

- 对象池管理器：对于频繁创建和销毁的对象（游戏中掉落的方块），使用对象池来复用对象，减少垃圾回收的开销，提高性能。


#### 动画

```js
/**
 * requestAnimationFrame
 * 与屏幕刷新率同步
 * 离开页面自动暂停，不消耗 CPU
 * 高精度，适合动画
 *  */
let left = 0
const frameInterval = 1000 / fps.value
let lastTime = performance.now()
function animate(now: number) {
const delta = now - lastTime
if (delta < frameInterval) {
    animateId.value = requestAnimationFrame(animate)
    return
  }
  left += 2
  box.style.transform = `translateX(${left}px)`
  lastTime = now
  if (left < 1000) {
    animateId.value = requestAnimationFrame(animate)
  } else {
    animateId.value = null
  }
}
animateId.value = requestAnimationFrame(animate)
```

#### mines游戏

```
介绍：选择格子可能是钻石或炸弹
游戏恢复：每次下注时会有个任务id，存储在store，用于恢复游戏，不传后端也会去查询
游戏初始化：点击下注后让所有格子随机变色发送请求获取所有格子数据（格子id，任务id，格子是否使用）
点击某个格子时触发放大的动画效果，发送请求携带格子id和任务id，返回是否为炸弹、赢多少然后结束放大动画;若是炸弹则游戏结束展开所有格子
用户提现主动结束
```

#### goal游戏

```
介绍：选择格子足球踢过去到格子上，遇到炸弹则游戏结束
踢足球动画实现：初始足球位置位于场地中间正下方，在初始化时给每个格子绑定了ref用于获取格子位置
足球x位置：(点击方块left + 方块宽度 / 2) - (场地left + 场地宽度 / 2)
足球y位置：(点击方块top + 方块高度 / 2) - (场地top + 场地高度 + 足球偏移位置 - 足球高度 / 2)
使用anime.js动画transform移动x和y位置，rotate做旋转
```

## 导出PDf防分页截断实现：

```
我当时是先做分页点计算，再做 PDF 渲染。先遍历导出区域里的元素，计算每个元素的 top 和 height，并换算到 PDF 坐标系里。分页时如果发现某个不可拆分的元素会跨页，把这个元素的起始位置作为下一页起点。对于表格行、图片、卡片这类不能拆的内容设成最小分页单元（通过约定类名）。最后把整块内容转成一张 canvas，通过 jsPDF.addImage 按不同纵向偏移写入每一页，每页若有多余内容部分则使用jsPDF.rect填充白色遮盖掉。
```

**如何换算到pdf坐标系（pt）**

```
// 计算比例
rate = PDF内容区宽度 / DOM导出区域实际宽度
pdfHeight = domHeight × rate
pdfTop = (elementTop - rootTop) × rate
```

# 八股文

## HTML

### 说说对 html 语义化的理解

- 定义：HTML 语义化是指使用具有明确意义的 HTML 标签来描述页面内容和结构，使得网页更易于理解和维护。
- 优点：
  - 提升可读性和可维护性。
  - 有助于搜索引擎优化（SEO）。
  - 利于无障碍访问，辅助技术可以更好地理解页面结构。

### 前端页面有哪三层构成，分别是什么？

- 结构层：由 HTML 组成，定义了网页的内容和结构。
- 表现层：由 CSS 组成，负责网页的样式和布局。
- 行为层：由 JavaScript 组成，负责网页的交互和动态效果。

### 行级元素和块级元素分别有哪些及怎么转换？

- 行级元素：span、a、img、input、label 等，默认情况下不会换行，宽度由内容决定。
  - 和其它元素都会在一行显示
  - 垂直方向外边距无效
  - 宽度由内容决定，可以设置宽度但不生效
  - 行级元素只能容纳文本或者其它行内元素
- 块级元素：div、p、h1-h6、ul、li 等，默认情况下会换行，占满父元素的宽度。
  - 总是在新行上开始，就是每个块级元素独占一行，默认从上到下排列
  - 宽度默认占满父元素的宽度，可以设置宽度
  - 块级元素可以容纳其它行级元素和块级元素。
- 转换方法：
  - 行级元素转换为块级元素：display: block。
  - 块级元素转换为行级元素：display: inline inline-block flex。
  
### H5有哪些新元素和新特性

- 语义话标签：header、nav、section、article、aside、footer等
- 视频video、音频audio
- 画布canvas
- 表单控件，calemdar、date、time、email
- 地理：geolocation API
- 本地离线存储，localStorage长期存储数据，浏览器关闭后数据不丢失，sessionStorage的数据在浏览器关闭后自动删除
- 拖拽释放

### iframe的作用以及优缺点

- 作用：iframe（内联框架）用于在一个网页中嵌入另一个独立的网页，可以实现跨域内容的展示、第三方服务的集成等功能。
- 通信：HTML5 提供了 window.postMessage 方法。
- 优点：
  - 隔离性：iframe 中的内容与父页面相互独立，避免了样式和脚本的冲突。
  - 跨域访问：可以通过 iframe 嵌入来自不同域的内容，解决跨域问题。
  - 复用性：可以在多个页面中复用同一个 iframe，提高开发效率。
  - 加载是异步的，页面可以在不等待 iframe 加载完成的情况下进行展示。
- 缺点：
  - 性能问题：过多的 iframe 会增加页面的加载时间和资源消耗。
  - SEO 问题：搜索引擎可能无法正确索引 iframe 中的内容，影响 SEO。
  - 响应式设计问题：iframe 的内容可能无法适应不同屏幕尺寸，导致布局问题。
  - 用户体验问题：iframe 中的内容可能与父页面的样式不一致，影响用户体验。

### meta viewport 是做什么用的，怎么写？
- 作用：Viewport，适配移动端，可以控制视口的大小和比例
```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1">
```
- content属性：
  - width ：宽度(数值/device-width)
  - height ：高度(数值/device-height)
  - initial-scale ：初始缩放比例
  - maximum-scale ：最大缩放比例
  - minimum-scale ：最小缩放比例
  - user-scalable ：是否允许用户缩放（yes/no）

## CSS

### 盒模型
- 定义：CSS 盒模型是指在网页布局中，每个元素都被看作一个矩形盒子，由内容区、内边距（padding）、边框（border）和外边距（margin）组成。
- 标准盒模型：content-box，元素的宽度和高度只包括内容区，不包括内边距和边框。
- IE 盒模型：border-box，元素的宽度和高度包括内容区、内边距和边框。
- 切换盒模型：box-sizing。
- 外边距合并：
  - 两个相邻元素的 margin-top / margin-bottom 取 最大值，而不是相加。
  - 当父元素没有 border / padding / 内容时，父元素的 margin-top 与第一个子元素的 margin-top 发生折叠，父元素的 margin-bottom 与最后一个子元素的 margin-bottom 也会发生折叠。

### CSS所有选择器及其优先级，哪些可以继承

- 选择器：
  - 通用选择器（*）
  - 标签选择器（div）
  - 类选择器（.class）
  - ID 选择器（#id）
  - 属性选择器（[type="text"]）
  - 伪类选择器（:hover、:nth-child()）
  - 伪元素选择器（::before、::after）
  - 子元素选择器（ul > li{}）
  - 后代选择器（div p{}）
  - 相邻兄弟选择器（div + p{}）
  - 通用兄弟选择器（div ~ p{}）
- 优先级：
  - 内联样式（style） > ID 选择器 > 类选择器、属性选择器、伪类选择器 > 标签选择器、伪元素选择器 > 通用选择器
  - !important 可以覆盖所有其他规则，但不建议滥用。
- 继承：
  - 继承的属性：color、font-family、font-size、line-height、text-align 等文本相关属性。
  - 不继承的属性：margin、padding、border、width、height 等布局相关属性。
  - 所有元素可继承的属性：visibility、cursor等。

**伪类和伪元素的区别**

- 伪类（:hover、:nth-child()）：用于选择元素的特定状态或位置，不创建新的元素。
- 伪元素（::before、::after）：用于在元素的内容之前或之后创建一个虚拟元素，可以设置样式和内容。

### CSS 布局方式

**定位**

- 静态定位（static）：默认值，元素按照正常的文档流进行布局。
- 相对定位（relative）：元素相对于其正常位置进行偏移，但仍占据原来的空间。
- 绝对定位（absolute）：元素相对于最近的定位祖先元素进行定位，如果没有定位祖先，则相对于初始包含块（浏览器窗口（viewport）或者HTML 文档根元素 <html>）进行定位。
- 固定定位（fixed）：元素相对于浏览器窗口进行定位，即使页面滚动也不会移动。
- 粘性定位（sticky）：元素在跨越特定阈值前表现为相对定位，超过阈值后表现为固定定位。

**浮动**

- 浮动（float）：元素向左或向右浮动，允许文本和内联元素环绕它。常用于实现多列布局，但可能导致父元素高度塌陷。
- 清除浮动：
  ```css
  /** 第一种 */
  .clearfix::after {
    content: "";
    display: block;
    clear: both;
  }
  
  /** 
   * 第二种
   * 父元素设置 触发 BFC（块级格式化上下文），从而清除浮动。
   */
  .container {
    overflow: hidden; /* 或 auto */
  }
  ```
  ```html
  <!-- 第三种 -->
  <div style="clear: both;"></div>
  ```

### BFC

- 定义：BFC（Block Formatting Context）是一个独立的渲染区域，元素在其中按照特定的规则进行布局和渲染。
![alt text](./images/bfc.png)

**布局特点**

- 垂直方向的 margin 不折叠
- 浮动元素被包含
- BFC 内的元素布局独立
- 不会与外部 BFC 发生浮动重叠
- 可以用来实现多列布局（如多栏等）

**作用**

- 清除内部浮动
- 阻止外边距折叠
- 自适应两栏布局

**如何创建BFC**

| CSS 属性                                        | 说明            |
| --------------------------------------------- | ------------- |
| `overflow: hidden / auto / scroll`            | 最常用创建 BFC 的方式 |
| `display: flow-root`                          | 现代方法，直接创建 BFC |
| `float: left / right`                         | 浮动元素自身创建 BFC  |
| `position: absolute / fixed`                  | 绝对定位元素创建 BFC  |
| `display: inline-block`                       | 行内块元素创建 BFC   |
| `display: table-cell / table-caption / table` | 表格相关元素创建 BFC  |

### 实现水平垂直居中方式

| 方法                     | 水平居中 | 垂直居中 | 适用场景                 |
| ---------------------- | ---- | ---- | -------------------- |
| `text-align: center`   | ✅    | ❌    | 行内 / inline-block 元素 |
| `margin: 0 auto`       | ✅    | ❌    | 块级元素，已知宽度            |
| `line-height`          | ✅    | ✅    | 单行文本                 |
| `absolute + transform` | ✅    | ✅    | 块级元素、多行内容            |
| Flex                   | ✅    | ✅    | 现代布局，通用              |
| Grid                   | ✅    | ✅    | 现代布局，复杂结构            |

### less 和 sass 的区别

- Sass的安装需要Ruby环境，是在服务端处理的，而Less是需要引入less.js来处理Less代码输出css到浏览器，也可以在开发环节使用Less，然后编译成css文件，直接放到项目中

  ![alt text](./images/less%20vs%20sass.png)

## JavaScript

### 数据类型

- 基本数据类型：
  - String
  - Number
  - Boolean
  - Null
  - Undefined
  - Symbol (ES6)
    BigInt (ES2020)
- 引用数据类型：主要是: Object (包括普通对象、Array、Function、Date等)。
  - 特点：
    - 值本身（即对象实体）存储在堆内存 (Heap)中，而变量的值是指向该堆内存地址的引用，该引用存储在栈内存中；
    - 变量之间的赋值是引用的复制。

**null和undefined区别**

- null：表示“无值”或“空值”，是一个对象类型的值，通常用于表示变量应该有一个对象，但目前为空。
- undefined：表示变量已声明但未初始化，或者对象属性不存在，类型为 undefined。

### 类型判断

- typeof：适用于基本数据类型，但对于 null 和数组等对象类型会返回 "object"。
- instanceof：适用于对象类型，判断一个对象是否是某个构造函数的实例，但对于基本数据类型无效。
- Object.prototype.toString.call()：适用于所有数据类型，返回一个表示对象类型的字符串，如 "[object Array]"、"[object Object]"。
- Array.isArray()：专门用于判断数组类型，返回 true 或 false。

### 变量声明: var, let 与 const

- var：函数作用域，变量提升，允许重复声明。
- let：块作用域，不允许重复声明，存在暂时性死区。
- const：块作用域，不允许重复声明，必须初始化，值不可变（对于对象类型，引用不可变，但对象内容可变），存在暂时性死区。
- 暂时性死区：在块作用域内，使用 let 或 const 声明的变量在声明之前是不可访问的，访问会抛出 ReferenceError 错误。

### 函数

- 函数声明：function foo() {}，存在函数提升，可以在声明之前调用。
- 函数表达式：const foo = function() {}，不存在函数提升，必须在声明之后调用。
- 箭头函数：const foo = () => {}，没有自己的 this、arguments、super 和 new.target，适合简化函数表达式，但不适合用作构造函数。
- 函数默认返回 undefined，如果没有显式返回值。

### 数组方法

- 会改变原数组的方法：push、pop、shift、unshift、splice、sort、reverse。
- 不会改变原数组的方法：concat、slice、map、filter、reduce、forEach、find、findIndex、includes、indexOf、join、toString。

| 遍历方式                                         | 支持异步（await）                     | 支持中断（break/return）                   | 备注                                   |
| -------------------------------------------- | ------------------------------- | ------------------------------------ | ------------------------------------ |
| **for 循环** (`for(let i=0;i<arr.length;i++)`) | ✅ 支持，配合 `await`                 | ✅ 支持 `break` / `continue`            | 最原始、最灵活的方式                           |
| **for...of**                                 | ❌ 不支持异步       | ✅ 支持 `break` / `continue` / `return` | 可直接迭代数组、Set、Map 等可迭代对象               |
| **forEach**                                  | ❌ 不支持异步 `await`                 | ❌ 不支持 `break` / `return`             | 异步需要用 `Promise.all` 或者 `for...of` 替代 |
| **map**                                      | ❌ 不支持异步 `await`（会立即返回新数组）       | ❌ 不支持 `break`                        | 返回新数组，常用于转换；异步要配合 `Promise.all`      |
| **filter**                                   | ❌ 不支持异步 `await`                 | ❌ 不支持 `break`                        | 用于筛选元素，异步可配合 `Promise.all`           |
| **reduce**                                   | ❌ 不直接支持异步 `await`（需手动链 Promise） | ❌ 不支持 `break`                        | 用于累加、聚合；异步需写成 async reduce           |
| **some**                                     | ❌ 不支持异步 `await`                 | ✅ 支持中断（返回 true 停止遍历）                 | 遍历到满足条件即可停止                          |
| **every**                                    | ❌ 不支持异步 `await`                 | ✅ 支持中断（返回 false 停止遍历）                | 遍历到不满足条件即可停止                         |
| **for await...of**                           | ✅ 支持（用于异步迭代器）                   | ✅ 支持 `break` / `continue` / `return` | 用于异步数组或异步生成器（async generator）        |


### DOM 操作与事件模型

- 高效的DOM节点操作：使用 DocumentFragment 批量操作 DOM，减少重排和重绘。
  ```js
  // 假设有一个 ul
  const ul = document.querySelector('ul');
  // 创建 fragment
  const fragment = document.createDocumentFragment();
  // 批量创建 li
  for (let i = 1; i <= 5; i++) {
    const li = document.createElement('li');
    li.textContent = 'Item ' + i;
    fragment.appendChild(li); // 先放在 fragment 中
  }
  // 一次性插入页面
  ul.appendChild(fragment);
  ```
- 事件委托：将事件监听器添加到父元素上，通过事件冒泡机制处理子元素的事件，减少事件监听器数量，提高性能。
- 事件捕获与冒泡：事件传播的两个阶段，捕获阶段从根元素到目标元素，冒泡阶段从目标元素回到根元素。可以通过 addEventListener 的第三个参数指定事件监听器是在捕获阶段还是冒泡阶段触发。
- 捕获阶段：事件从根元素向目标元素传播，先触发父元素的事件监听器，再触发子元素的事件监听器。
- 目标阶段：事件触发在目标元素上，触发目标元素的事件监听器。
- 冒泡阶段：事件从目标元素向根元素传播，先触发子元素的事件监听器，再触发父元素的事件监听器。

### 本地存储方案对比

- localStorage：同步 API，存储容量较大（约 5MB），适合存储较大数据，但可能导致性能问题。
- sessionStorage：同步 API，存储容量较大（约 5MB），数据在页面会话结束时被清除，适合存储临时数据。
- IndexedDB：异步 API，存储容量较大（取决于浏览器），适合存储大量结构化数据，支持事务和索引，但使用较复杂。
- Cookie：同步 API，存储容量较小（约 4KB），数据随每个 HTTP 请求发送，适合存储少量数据，如用户认证信息，但可能存在安全和性能问题。

### script、script async 和 script defer 的区别

- script：默认行为，浏览器会立即下载并执行脚本，阻塞页面渲染，适合需要立即执行的脚本。
- script async：浏览器会异步下载脚本，下载完成后立即执行，可能会在页面渲染完成之前执行，适合独立的脚本，如分析工具。
- script defer：浏览器会异步下载脚本，但会等到页面渲染完成后再执行，适合依赖 DOM 的脚本，如主应用逻辑。

**注意：没有 src 属性的脚本，async 和 defer 属性会被忽略。**

### this 指向

- 全局上下文：在全局作用域中，this 指向全局对象（浏览器中是 window）。
- 函数上下文：在普通函数中，this 的值取决于函数的调用方式。默认情况下，非严格模式下 this 指向全局对象，严格模式下 this 为 undefined。
  - 对象方法：当函数作为对象的方法调用时，this 指向调用该方法的对象。
  - 构造函数：当函数作为构造函数使用（使用 new 关键字调用）时，this 指向新创建的对象。
- 箭头函数：箭头函数没有自己的 this，this 的值由外层作用域决定，通常指向定义时所在的上下文。

### call()、apply()、bind()区别

- call()：立即调用函数，并且可以传入参数列表。
- apply()：立即调用函数，但参数以数组形式传入。
- bind()：返回一个新的函数，绑定 this 和参数，但不立即调用。

### 原型与原型链

- 原型：每个 JavaScript 对象都有一个内部属性 [[Prototype]]，指向另一个对象，这个对象被称为原型。通过原型可以实现属性和方法的共享。
- 原型链：当访问一个对象的属性或方法时，如果该对象没有这个属性或方法，JavaScript 引擎会沿着原型链向上查找，直到找到该属性或方法或者到达原型链的末尾（null）。原型链是 JavaScript 实现继承的一种机制。
- 继承：通过原型链实现对象之间的继承关系，子对象可以访问父对象的属性和方法。
- 准则
  - Person.prototype.constructor == Person // **准则1：原型对象（即Person.prototype）的constructor指向构造函数本身**
  - person01.**proto** == Person.prototype // **准则2：实例（即person01）的**proto**和原型对象指向同一个地方**
- **总结**
  - prototype 是“生产共享方法的地方”
  - [[Prototype]] 是“对象真实的原型指针（内部查找链路）”
  - __proto__ 是“访问 [[Prototype]] 的接口（查找路线）”
  - constructor 是“原型对象指回构造函数的引用指针”

### 继承

- 原型链继承：通过将子类的原型指向父类的实例来实现继承，但存在引用类型属性共享和无法传递参数的问题。
  ```js
    function Parent() {
      this.name = 'Parent';
      this.hobbies = ['coding'];
    }
    Parent.prototype.sayHello = function() {
      console.log('Hello from Parent');
    }
    function Child() {}
    Child.prototype = new Parent();
    const child1 = new Child();
    const child2 = new Child();
    child1.name = 'Child1';
    child1.hobbies.push('music');
    console.log(child1.name); // Child1
    console.log(child2.name); // Parent
    console.log(child2.hobbies); // ['coding', 'music']（引用类型属性共享问题）
    child1.sayHello(); // Hello from Parent
    child2.sayHello(); // Hello from Parent

    // child1.__proto__ -> Child.prototype -> Child.prototype.__proto__ -> Parent.prototype

  ```
 
- 构造函数继承：在子类构造函数中调用父类构造函数来实现继承，解决了引用类型属性共享的问题，但无法继承父类原型上的方法。
  ```js
    function Parent() {
      this.name = 'Parent';
    }
    Parent.prototype.sayHello = function() {
      console.log('Hello from Parent');
    }
    function Child() {
      Parent.call(this); // 调用父类构造函数
    }
    const child1 = new Child();
    const child2 = new Child();
    child1.name = 'Child1';
    console.log(child1.name); // Child1
    console.log(child2.name); // Parent（不共享引用类型属性）
    child1.sayHello(); // TypeError: child1.sayHello is not a function（无法继承父类原型上的方法）
  ```
- 组合继承：结合原型链继承和构造函数继承的优点，既解决了引用类型属性共享的问题，又能继承父类原型上的方法，但存在调用父类构造函数两次的问题。
  ```js
    function Parent() {
      this.name = 'Parent';
    }
    Parent.prototype.sayHello = function() {
      console.log('Hello from Parent');
    }
    function Child() {
      Parent.call(this); // 调用父类构造函数
    }
    Child.prototype = new Parent(); // 创建一个父类实例作为子类原型，调用父类构造函数一次
    Child.prototype.constructor = Child; // 修正 constructor 指向
    const child1 = new Child();
    const child2 = new Child();
    child1.name = 'Child1';
    console.log(child1.name); // Child1
    console.log(child2.name); // Parent（不共享引用类型属性）
    child1.sayHello(); // Hello from Parent
    child2.sayHello(); // Hello from Parent
  ```
- 寄生组合继承：通过创建一个中间函数来避免调用父类构造函数两次，实现了更高效的继承方式。
  ```js
    function Parent() {
      this.name = 'Parent';
    }
    Parent.prototype.sayHello = function() {
      console.log('Hello from Parent');
    }
    function Child() {
      Parent.call(this); // 调用父类构造函数
    }
    Child.prototype = Object.create(Parent.prototype); // 创建一个中间对象，避免调用父类构造函数两次
    Child.prototype.constructor = Child; // 修正 constructor 指向
    const child1 = new Child();
    const child2 = new Child();
    child1.name = 'Child1';
    console.log(child1.name); // Child1
    console.log(child2.name); // Parent（不共享引用类型属性）
    child1.sayHello(); // Hello from Parent
    child2.sayHello(); // Hello from Parent
  ```

### 闭包

- 定义：闭包是指一个函数能够访问并操作其外部作用域中的变量，即使外部函数已经执行完毕。
- 作用：闭包可以让函数访问并操作函数外部的变量，即使外部函数已经执行完毕。闭包常用于数据封装、模拟私有变量、实现函数工厂等场景。
- 注意事项：闭包可能导致内存泄漏，因为闭包会保持对外部变量的引用，导致这些变量无法被垃圾回收器回收。因此，在使用闭包时要注意避免不必要的引用，及时释放资源。

### 内存泄漏和内存溢出

| 特性     | 内存泄漏                 | 内存溢出                |
| ------ | -------------------- | ------------------- |
| 定义     | 已不再使用的内存 **未被回收**    | 程序分配内存 **超过环境限制**   |
| 是否立即报错 | 不一定，可能慢慢积累           | 会立即报错或崩溃            |
| 典型原因   | 全局变量、闭包、未移除事件、DOM 引用 | 无限递归、超大对象分配、长时间泄漏累积 |
| 表现     | 内存持续增加、性能下降          | 程序崩溃、报错、堆空间耗尽       |


### Promise

- 定义：Promise 是一种用于处理异步操作的对象，代表一个可能在未来完成或失败的操作及其结果。
- 状态：Promise 有三种状态：pending（等待中）、fulfilled（已完成）和 rejected（已拒绝）。状态只能从 pending 转变为 fulfilled 或 rejected，且一旦状态改变就不能再改变。
- 方法：
  - Promise.all(): 接收一个 Promise 数组，返回一个新的 Promise，当数组中的所有 Promise 都 fulfilled 时，返回一个包含所有结果的数组；如果有任何一个 Promise rejected，则返回一个 rejected 的 Promise。
  - Promise.race(): 接收一个 Promise 数组，返回一个新的 Promise，当数组中的任意一个 Promise 状态改变时，返回该 Promise 的状态和结果。
  - Promise.allSettled(): 接收一个 Promise 数组，返回一个新的 Promise，当数组中的所有 Promise 都 settled（已完成或已拒绝）时，返回一个包含所有结果的数组。
  - Promise.any(): 接收一个 Promise 数组，返回一个新的 Promise，当数组中的任意一个 Promise fulfilled 时，返回该 Promise 的结果；如果所有 Promise 都 rejected，则返回一个 rejected 的 Promise。

### async/await

- 定义：async/await 是基于 **Promise 的语法糖**，用于处理异步操作，使异步代码看起来像同步代码。
- async 函数：声明一个函数为异步函数，返回一个 Promise 对象。
- await 表达式：用于等待一个 Promise 对象的结果，必须在 async 函数内部使用。
- 错误处理：可以使用 try/catch 块来捕获 async 函数中的错误，或者在调用 async 函数时使用 .catch() 方法来处理错误。
- **await 不会阻塞主线程，只是暂停 async 函数内部的执行，async 函数永远返回 Promise**

### 高阶函数

- 定义：高阶函数是指接受一个或多个函数作为参数，或者返回一个函数作为结果的函数。
- 作用：高阶函数可以用于函数组合、函数柯里化、函数节流等场景，增强函数的功能和灵活性。
- 示例：Array.prototype.map()、Array.prototype.filter()、Array.prototype.reduce() 等都是高阶函数，因为它们接受一个函数作为参数来处理数组中的每个元素。

### es6 新特性

- let 和 const 变量声明
- 箭头函数
- 模板字符串
- 解构赋值
- 扩展运算符（...）
- 默认参数
- Promise
- 模块化（import/export）
- 类和继承
- Symbol 和 Set/Map 数据结构
- 生成器函数（Generator）
- 对象和数组的扩展（如 Object.assign()、Array.from() 等）
- 数组方法（如 find()、findIndex()、includes()、map()、filter()、reduce() 等）
- Proxy 和 Reflect 对象
- 迭代器和 for...of 循环

### 防抖和节流

- 防抖：一段时间内只执行最后一次（搜索）
- 节流：每隔固定时间执行一次（滚动、窗口调整大小等）

```js
function debounce(fn, delay = 300) {
  let timer = null;

  return function (...args) {
    if (timer) clearTimeout(timer);
    timer = setTimeout(() => {
      fn.apply(this, args);
      timer = null;
    }, delay);
  };
}

function debounce(fn, delay = 300, immediate = false) {
  let timer = null;

  return function (...args) {
    if (timer) clearTimeout(timer);
    if (immediate) {
      if (!timer) {
        fn.apply(this, args);
      }
      timer = setTimeout(() => {
        timer = null;
      }, delay);
    } else {
      timer = setTimeout(() => {
        fn.apply(this, args);
        timer = null;
      }, delay);
    }
  };
}

function throttle(fn, delay = 300) {
  let lastTime = 0;

  return function (...args) {
    now = new Date();
    if (now - lastTime >= delay) {
      fn.apply(this, args);
      lastTime = now;
    }
  };
}
```

| 时间(ms) | 事件触发 | 防抖+立即执行                             | 节流                    |
| -------- | -------- | ----------------------------------------- | ----------------------- |
| 0        | 点击一次 | ✅立即执行                                | ✅立即执行              |
| 500      | 点击一次 | ❌不执行（还在等待 delay）                | ❌不执行（间隔未到）    |
| 1200     | 点击一次 | ❌不执行（前一次触发后 delay 已过才重置） | ✅执行（间隔到 1000ms） |
| 1600     | 点击一次 | ✅执行（因为上一次防抖执行 reset timer）  | ❌不执行（间隔未到）    |

### 浅拷贝和深拷贝

- 浅拷贝：只复制对象的第一层属性，如果属性值是引用类型，则复制的是引用地址，修改其中一个对象会影响另一个对象。
  - 常见的浅拷贝方法有 Object.assign()、扩展运算符（...）等。
- 深拷贝：复制对象的所有层级属性，如果属性值是引用类型，则会创建一个新的对象，修改其中一个对象不会影响另一个对象。
  - 常见的深拷贝方法有 JSON.parse(JSON.stringify())、递归函数、第三方库（如 lodash 的 \_.cloneDeep()）等。

### 跨域
**什么是跨域**
- 定义：跨域是指在浏览器中，当前网页的域（协议、域名、端口）与请求资源的域不一致时，浏览器出于安全考虑阻止访问该资源的行为。

**解决方案**

- JSONP：通过动态创建 script 标签来实现跨域请求，适用于 GET 请求，但不支持 POST 请求。
- CORS：通过服务器设置 Access-Control-Allow-Origin 头来允许跨域请求，支持多种 HTTP 方法，但需要服务器端配合。
- 代理服务器：在同域下设置一个代理服务器，将跨域请求转发到目标服务器，适用于开发环境，但需要额外的服务器配置。
- WebSocket：通过建立持久化连接来实现跨域通信，适用于实时数据传输，但需要服务器端支持。

### 垃圾回收机制

- 什么是垃圾回收：垃圾回收是指自动释放不再使用的内存资源的过程，JavaScript 引擎通过垃圾回收机制来管理内存，避免内存泄漏和内存溢出。
- 标记清除算法：JavaScript 引擎使用标记清除算法来进行垃圾回收，该算法通过标记所有可达的对象，然后清除未被标记的对象来释放内存。
- 内存泄漏：内存泄漏是指程序中不再使用的对象仍然被引用，导致无法被垃圾回收器回收，最终可能导致内存溢出。常见的内存泄漏场景包括全局变量、闭包、定时器和事件监听器等。

### 事件循环

- 定义：事件循环是 JavaScript 运行时的一种机制，用于处理异步操作和事件。它通过一个循环不断检查消息队列（Message Queue）和调用栈（Call Stack），确保异步任务在适当的时机执行。
  - 消息队列（浏览器/Node.js）：存放异步任务。
  - 调用栈（js）：存放同步任务，单线程。
- 执行顺序：同步任务->微任务（如 Promise.then()）->宏任务（如 setTimeout()）。当调用栈为空时，事件循环会先执行所有微任务，然后再执行一个宏任务。

### 设计模式

- 单例模式：确保一个类只有一个实例，并提供一个全局访问点。
- 工厂模式：定义一个用于创建对象的接口，让子类决定实例化哪个类。
- 观察者模式：定义对象之间的一对多依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都会得到通知并自动更新。
- 模块模式：将相关的功能封装在一个模块中，提供公共接口，隐藏内部实现细节。

### JavaScript V8引擎

 - V8 是 Google 开发的一个开源 JavaScript 引擎，用来将 JS 代码编译、执行，支持浏览器（Chrome）和 Node.js。
 - 特点：
   - 高性能：使用 JIT 编译，将 JS 直接编译为本地机器码
   - 垃圾回收：自动回收不再使用的内存
   - 内嵌优化：通过内联缓存（inline caching）、优化编译器等技术提高执行速度
   - 跨平台：不依赖浏览器，Node.js 也能使用

## Vue

### 生命周期

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

### 组件通信

- 父向子：props、属性透传（$attr）、ref/expose（实例、DOM）。
- 子向父：$emit 触发事件。
- v-model：可以父向子、子向父。
- 兄弟：
  - 父组件作为中介
  - 使用全局事件总线（Event Bus）或状态管理库（如 Vuex）来实现跨组件通信。
- 跨层级通信：provide/inject、Vuex/Pinia、Event Bus（vueuse的useEventBus）。

### 事件修饰符

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

### 指令

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

### ref和reactive区别

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

### 插槽

- 作用：插槽（Slot）是 Vue 提供的一种机制，用于在组件中预留位置，让父组件可以向子组件传递内容，实现组件的灵活组合和复用。
- 分类：
  - 默认插槽：没有名称的插槽，父组件传递的内容会被渲染在子组件的默认插槽位置。
  - 具名插槽：通过 name 属性命名的插槽，父组件可以通过 v-slot 指令指定内容渲染到对应的具名插槽位置。
  - 作用域插槽：允许子组件在插槽内容中暴露数据给父组件的机制。
- 动态插槽名（Vue3）：

```vue
<template #[dynamicSlotName]>动态插槽内容</template>
```

### watch、watchEffect、computed的区别

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

### keep-alive

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

### 异步组件

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

### Vue-Router

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

### 导航守卫

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

### 状态管理

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

### 性能优化

#### 组件级优化策略

**渲染控制**

- v-memo：告诉 Vue 只有在依赖变化时才重新渲染这个节点或组件
- v-show：高频切换（CSS显示隐藏）
- v-if：低频切换（销毁/重建组件）

**组件设计优化**

- 细粒度拆分：隔离高频更新组件（通过分析组件的状态和更新频率，将高频更新的部分拆分成独立组件，减少不必要的渲染），
  怎么分析？可以通过Vue Devtools观察组件的更新频率和性能瓶颈，识别出哪些组件频繁更新，哪些组件状态变化较大，从而决定如何拆分组件。
- 异步组件：延迟加载非关键组件

#### 状态管理优化

- shallowRef/shalloReactive：非深度响应式数据
  - shallowRef: 只对 最外层对象本身是响应式的，内部属性不会被递归响应化。一些大对象且不需要关心其内层，比如webSocket
- 使用markRaw避免不必要的响应式

#### 资源与加载优化

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

#### 运行时性能优化

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

#### 架构级优化

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

#### 工具链优化

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

#### 面试

**Vue3比Vue2快在哪里？**

- 响应式系统：Vue3基于Proxy实现更高效的响应式系统，减少了性能开销。
- 虚拟DOM优化：Vue3的虚拟DOM算法更高效，减少了不必要的渲染和更新。
- Tree Shaking支持：Vue3支持Tree Shaking，未使用的功能不会被打包进最终代码，减小了包体积。
- 组件设计优化：Vue3引入了Composition API，允许更灵活的组件设计和状态管理，提升了性能。

**v-memo的使用场景（避免执行render）**

- 大型列表渲染优化
- 复杂表单优化
- 重复渲染的子组件

### Vue2和Vue3区别

#### 架构设计区别

| 特性           | Vue2                    | Vue3              | 优势                                |
| -------------- | ----------------------- | ----------------- | ----------------------------------- |
| **响应式系统** | `Object.defineProperty` | `Proxy`           | 支持动态属性/数组索引监听，性能更优 |
| **代码组织**   | Options API             | Composition API   | 逻辑复用更灵活，类型推导更友好      |
| **源码组织**   | Flow 类型系统           | TypeScript 重写   | 更好的类型支持和源码可维护性        |
| **包体积**     | 全量引入（22.5kb）      | 按需引入（<10kb） | Tree Shaking 减少 41% 体积          |

#### 响应式系统升级

- Vue2：基于 `Object.defineProperty` 实现响应式，无法监听新增属性和数组索引。
- Vue3：基于 `Proxy` 实现响应式，支持监听对象的所有属性和数组索引，性能更优。
  - 原理：Vue3使用Proxy代理（惰性递归）整个对象，拦截get/set操作，实现响应式。相比Vue2的defineProperty需要递归处理每个属性，Proxy更高效。

#### Composition API vs Options API

- Options API：通过data、methods、computed等选项组织代码，适合小型组件，但逻辑复用较困难。
- Composition API：通过setup函数组织代码，使用ref/reactive定义响应式数据，适合大型组件，逻辑复用更灵活。
  - 优势：更好的逻辑复用（通过组合函数），更好的类型推导（TypeScript支持），更清晰的代码结构（将相关逻辑放在一起）。

#### 性能优化对比

| 优化点               | Vue2                               | Vue3                                       |
| -------------------- | ---------------------------------- | ------------------------------------------ |
| **响应式系统**       | `Object.defineProperty`，性能较低  | `Proxy`，性能更优                          |
| **虚拟DOM算法**      | 基于简单的diff算法                 | 基于更高效的diff算法，减少不必要的渲染     |
| **Tree Shaking支持** | 不支持，整个库被打包               | 支持，未使用的功能不会被打包，减小包体积   |
| **组件设计优化**     | Options API，逻辑复用较困难        | Composition API，逻辑复用更灵活            |
| **静态提升**         | 不支持，所有节点每次渲染都需要处理 | 支持，静态节点提升到编译时，减少运行时开销 |

#### 新特性与API

- **Teleport**：允许将组件渲染到 DOM 的任意位置，提升灵活性。
- **Suspense（异步组件）**：统一管理异步组件的加载状态，提升用户体验。
- **Fragments（碎片）**：组件可以返回多个根节点，减少不必要的 DOM 层级。

#### 面试

**Vue3的模版编译优化有哪些？**

- 静态节点提升：不会变化的节点只创建一次，避免重复渲染。
- 静态属性优化：静态属性（class、id、style 等）在编译阶段确定，不在运行时计算。
- 事件处理缓存：不依赖响应式数据的事件处理函数只创建一次，减少函数重建开销。

**Vue2项目如何升级Vue3？**

- 使用@/vue/compat过渡
- 逐步替换废弃API（$children,filters等）
- 优先迁移新组件，逐步重构旧组件

### SPA

**定义**：整个应用只有一个HTML文件，通过动态替换DOM内容实现"页面"切换。
**优点**：无刷新跳转，前后端分离开发，减轻服务器渲染压力。
**缺点**：首屏加载慢，SEO不友好，路由管理复杂度

#### SPA与MPA对比

| 特性           | SPA                          | MPA (多页面应用)     |
| -------------- | ---------------------------- | -------------------- |
| **页面数量**   | 1 个 HTML                    | 多个 HTML            |
| **页面跳转**   | 前端路由控制，无刷新         | 整页刷新             |
| **数据请求**   | Ajax/Fetch 局部获取数据      | 每次跳转加载完整页面 |
| **开发复杂度** | 高（需前端路由、状态管理等） | 低                   |
| **SEO 支持**   | 差（需额外优化）             | 优                   |

#### 面试

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

### 其他

#### nextTick

- Vue 的 数据更新是异步的，nextTick用于在 DOM 更新完成后执行回调函数。它确保在数据变化后，所有相关的 DOM 更新都已经完成，适合在更新后访问或操作 DOM。
- 使用场景：
  - 在数据更新后需要访问更新后的 DOM 元素。
  - 在组件更新后执行某些操作，如滚动到特定位置。

#### Hook 和工具函数怎么区分

```
“我的划分标准是看它是否依赖 Vue 的响应式能力和生命周期。依赖 ref、computed、watch、onMounted 这类能力的，我会抽成 Hook；如果只是纯函数计算、格式化、转换、对象处理，就放到工具包里。这样职责更清晰，也方便测试和复用。”
```

**withDefaults**
- props 自动有默认值
- TS 类型更精确
- 代码更简洁可读

## TS

**内置工具类型**

| 工具类型 | 作用 | 等价写法 |
|----------|------|----------|
| `Partial<T>` | 所有属性变为可选 | `{ [P in keyof T]?: T[P] }` |
| `Required<T>` | 所有属性变为必填 | `{ [P in keyof T]-?: T[P] }` |
| `Readonly<T>` | 所有属性变为只读 | `{ readonly [P in keyof T]: T[P] }` |
| `Pick<T, K>` | 选取 T 中指定属性 K | `{ [P in K]: T[P] }` |
| `Omit<T, K>` | 排除 T 中指定属性 K | `Pick<T, Exclude<keyof T, K>>` |
| `Record<K, T>` | 创建键为 K 类型，值为 T 类型的对象 | `{ [P in K]: T }` |
| `Exclude<T, U>` | 从联合类型 T 中排除 U 类型 | `T extends U ? never : T` |
| `Extract<T, U>` | 从联合类型 T 中提取 U 类型 | `T extends U ? T : never` |

**any和unknown的主要区别是什么**

  - any禁用所有类型检查，相当于退化到JS；unknown要求在使用前显式断言或类型检查，确保操作的安全性。


**为什么使用元组而不是普通数组**

  - 元组精确维护元素的位置和类型，例如[string, number]确保第一个元素是字符串，第二个是数字，防止越界访问和类型错位。


**void和undefined在函数返回值中的区别？**


  - void表示函数没有有效返回值（可返回undefined和null），而undefined要求必须显式返回undefined值 

**双重断言为什么危险**

  - as any as T 绕过类型系统检查，可能导致运行时错误。应优先使用类型守卫或正确的类型声明

**协变和逆变的实践意义？**

  - 协变允许子类型赋值给父类型，逆变允许父类型赋值给子类型。

**type和interface的区别？**

| 特性        | `interface`                  | `type`                       |
| --------- | ---------------------------- | ---------------------------- |
| 定义对象      | ✅ 可以                         | ✅ 可以                         |
| 定义基本类型别名  | ❌ 不能                         | ✅ 可以，例如 `type Name = string` |
| 定义联合/交叉类型 | ❌ 不支持直接                      | ✅ 支持，例如 `type AorB = A 或 B` |
| 扩展        | ✅ `extends` 可以扩展其他 interface | ✅ 可以通过交叉类型 `&` 扩展            |
| 合并声明      | ✅ 同名 interface 会自动合并         | ❌ type 不支持同名合并，会报错           |

**如何选择使用type还是interface？**

  - 定义对象类型时优先使用interface，定义基本类型别名、联合/交叉类型时使用type。

**readonly和const有什么区别？**

  - readonly用于类型层面，表示属性不可修改，但对象本身可变；const用于值层面，表示变量不可重新赋值，但对象属性可修改。

**何时需要编写 .d.ts文件？**

  - 当引入没有类型声明的第三方库时。
  - 扩展已有模块类型
  - 声明全局变量/类型。优先使用@types包，其次考虑自定义声明文件。

**keyof typeof组合使用有什么价值？**

  - keyof typeof可以获取对象的键名类型，常用于限制函数参数为对象的某个属性。例如：

  ```ts
  const Colors = {
    Red: 'red',
    Green: 'green',
    Blue: 'blue',
  } as const;
  type Color = keyof typeof Colors; // 'Red' | 'Green' | 'Blue'
  ```

**satisfies和as断言有什么区别？**

  - as：强制类型转换（可能掩盖错误）
  - satisfies：验证类型兼容性（不改变类型推断）

**declare**

  - 声明全局变量
  - 声明模块
  - 声明函数/类类型

## HTTP及浏览器

### GET和POST有什么区别？

| 特性        | GET                                 | POST                             |
| --------- | ----------------------------------- | -------------------------------- |
| 请求参数位置    | URL 中（query string，例如 `?key=value`） | 请求体（body）中                       |
| 数据长度限制    | 有长度限制（浏览器和服务器限制，通常约 2KB～8KB）        | 理论上无限制（受服务器配置影响）                 |
| 安全性       | 不安全，参数在 URL 可见，容易被缓存或记录             | 相对安全，参数不显示在 URL，但仍需 HTTPS 保护敏感数据 |
| 幂等性       | 是（多次请求结果相同）                         | 不一定（可能会改变服务器状态，如提交表单）            |
| 用途        | 获取资源、查询数据                           | 提交数据、创建或修改资源                     |
| 缓存        | 可以被浏览器缓存                            | 默认不缓存                            |
| 浏览器历史记录   | 会被记录                                | 通常不记录在历史记录中                      |
| 适合传输的数据类型 | 简单数据，ASCII 文本                       | 大量数据、复杂对象（如 JSON、文件上传）           |

### https是怎么保证安全的，为什么比http安全？

- 加密传输：HTTPS使用SSL/TLS协议加密数据，防止中间人攻击和数据泄露。
- 认证机制：通过数字证书验证服务器身份，防止钓鱼攻击。
- 数据完整性：使用消息认证码（MAC）确保数据未被篡改。

### post请求为什么会多发送一次option请求？

- 预检请求：当发送跨域POST请求时，浏览器会先发送一个OPTIONS请求（预检请求）来检查服务器是否允许该跨域请求。
- 服务器响应：如果服务器响应允许该请求，浏览器才会继续发送实际的POST请求。

### HTTP的keep-alive是干什么的？

- 连接复用：允许在同一TCP连接上发送多个HTTP请求，减少连接建立和关闭的开销。
- 提升性能：减少延迟和资源消耗，提升页面加载速度。

### 同样是重定向307，303，302的区别？

| 状态码 | 描述 | 适用场景 |
| --- | --- | --- |
| 302 Found | 临时重定向，原请求方法不变 | 旧版HTTP，现代浏览器通常将POST请求重定向为GET |
| 303 See Other | 临时重定向，原请求方法改为GET | POST请求重定向后应使用GET请求获取资源 |
| 307 Temporary Redirect | 临时重定向，原请求方法不变 | POST请求重定向后仍使用POST请求获取资源 |

### HTTP的缓存机制

| 缓存类型                                  | 是否发请求  | 响应状态                      | 代表字段                                  | 优点               | 缺点                        | 适用场景 / 选择建议                       |
| ------------------------------------- | ------ | ------------------------- | ------------------------------------- | ---------------- | ------------------------- | --------------------------------- |
| **强缓存（Strong Cache）**                 | ❌ 不发请求 | 200 from cache            | `Cache-Control`, `Expires`            | 性能极高，减少服务器压力，加载快 | 数据更新不及时，需合理设置过期时间         | 静态资源（JS/CSS/图片），配合文件名 hash 使用     |
| **协商缓存（Negotiation Cache）**           | ✅ 会发请求 | 304 Not Modified / 200 OK | `ETag` / `Last-Modified`              | 数据准确，可避免重复下载     | 仍需发请求，略慢于强缓存；ETag 生成有性能开销 | HTML 页面、API 数据（非实时）、部分动态资源        |


### 浏览器从输入URL到看到页面发生的全过程

1. URL解析：浏览器解析URL，提取协议、域名、路径等信息。
2. DNS解析：浏览器通过DNS服务器将域名解析为IP地址。
3. TCP连接：浏览器与服务器建立TCP连接（可能使用HTTPS的SSL/TLS握手）。
4. HTTP请求：浏览器发送HTTP请求（如GET）到服务器。
5. 服务器处理：服务器接收请求，返回响应内容。
6. 渲染页面：浏览器解析 HTML 生成 DOM，同时解析 CSS 生成 CSSOM，生成渲染树，并将页面渲染出来。

### 什么是重绘和回流及怎么减少重绘和回流？

- 重绘（Repaint）：元素的外观发生变化（如颜色、背景等），但布局不变。
- 回流（Reflow）：元素的尺寸或位置发生变化，导致浏览器重新计算布局。
- 减少重绘和回流：
  - 避免频繁操作样式：尽量一次性修改多个样式属性，而不是逐一修改，以减少浏览器的重绘和回流次数。
  - 利用CSS3动画：CSS3动画和过渡不会触发回流，因为它们是通过GPU进行渲染的，这可以大大提高性能。
  - 避免使用table布局：table布局在发生变化时可能需要多次计算，这会增加回流次数。尽量使用flexbox或grid等现代布局技术。
  - 批量修改DOM：如果需要添加、删除或修改多个DOM节点，可以考虑使用DocumentFragment或离线节点，这样可以在一次回流中完成所有操作。
  - 使用绝对定位：绝对定位的元素不会触发其父元素及后续元素的回流，因为它们已经脱离了正常的文档流。
  - 避免使用内联样式：内联样式会增加重绘和回流的可能性，因为它们会直接修改元素的样式。尽量使用外部CSS文件来管理样式。

### 如何实现浏览器内多个标签页之间的通信？

- localStorage：通过监听storage事件实现跨标签页通信。
- BroadcastChannel API：创建一个频道，多个标签页可以通过该频道发送和接收消息。
- SharedWorker：创建一个共享的Worker，多个标签页可以通过该Worker进行通信。
- Service Worker：通过postMessage与Service Worker通信，Service Worker再将消息广播到其他标签页。

### 简述tcp三次握手和4次挥手的过程

- TCP三次握手：
  1. 客户端发送SYN包（SYN=1）请求建立连接。
  2. 服务器响应SYN-ACK包（SYN=1, ACK=1）表示同意连接。
  3. 客户端发送ACK包（ACK=1）确认连接建立成功。
- TCP四次挥手：
  1. 客户端发送FIN包（FIN=1）请求关闭连接。
  2. 服务器响应ACK包（ACK=1）确认收到关闭请求。
  3. 服务器发送FIN包（FIN=1）表示同意关闭连接。
  4. 客户端响应ACK包（ACK=1）确认收到关闭请求，连接关闭完成。

### HTTP1.1和HTTP2的区别？

- HTTP2 支持 多路复用（单 TCP 连接并发请求）
- HTTP2 使用 二进制帧，HTTP1.1 是文本协议
- HTTP2 支持 header 压缩
- HTTP2 可以 服务器推送

## Git

### 如何解决冲突？

- 手动解决：打开冲突文件，按照标记（<<<<<<<、=======、>>>>>>>）手动编辑，保留需要的代码，删除标记后保存。
**使用命令**
- git rebase
- git merge
  - 区别：rebase会将当前分支的提交移到目标分支的最新提交之后，保持提交历史线性；merge会创建一个新的合并提交，保留分支的历史结构。

### git pull设置为rebase
- 命令行：git config --global pull.rebase true
- 作用：避免不必要的合并提交，保持提交历史清晰。

## vite和webpack

### 什么是 Vite 和 Webpack？它们的区别是什么？

- Vite 是一个基于 ES 模块的现代前端构建工具，提供快速的开发体验和高效的生产构建。
- Webpack 是一个功能强大的模块打包工具，支持各种资源类型的处理和复杂的构建配置。

| 对比项     | Webpack    | Vite               |
| ------- | ---------- | ------------------ |
| 启动方式    | 先整体打包再启动   | 启动 Dev Server，按需加载 |
| 开发环境    | 基于 Bundle  | 基于 ESM             |
| 冷启动速度   | 慢（打包整个项目） | 极快 (基于 ESM 按需加载)                |
| 热更新 HMR | 重新构建模块树    | 精确模块更新             |
| 打包工具    | Webpack 本身 | Rollup（生产环境）       |
| 配置复杂度   | 较复杂        | 简洁                 |
| 浏览器要求   | 兼容性更强      | 依赖现代浏览器 ESM        |
| 大型老项目   | 更适合        | 改造成本较高             |
| 开发体验    | 一般         | 非常好                |

### Vite 的热更新为什么比 Webpack 快？

- Vite 开发模式不打包整个项目，只按需加载模块。
- 使用浏览器原生 ES 模块，改动文件时只重新加载变化的模块，而 Webpack 会重新打包依赖模块。

### Webpack 的 Entry、Output、Loader、Plugin 分别是什么？

- Entry：入口文件，Webpack 从这里开始构建依赖图。
- Output：输出配置，指定打包后的文件名和路径。
- Loader：转换器，处理非 JavaScript 文件（如 CSS、图片等）。
- Plugin：插件，扩展 Webpack 功能（如优化、压缩、环境变量等）。

### Webpack 的模块热替换（HMR）原理是什么？

- Webpack Dev Server 监听文件变化。
- 发生变化时，构建改动模块生成新的代码片段。
- 通过 WebSocket 通知浏览器更新模块。
- 浏览器只替换改变的模块，不刷新整个页面。

### Vite 开发模式的依赖预构建（依赖优化）是怎么做的？

- Vite 会把 node_modules 中的依赖使用 esbuild 预先编译为 ESM，提升浏览器加载速度。
- 原因：浏览器不支持直接加载 CommonJS，需要转换为 ESM。

### Webpack 打包流程是什么？

- 初始化：读取配置，初始化 Compiler 对象。
- 编译：从 Entry 开始，递归构建依赖图，应用 Loader。
- 输出：将模块内容组合成 Chunk，生成资源文件，应用 Plugin，写入 Output

### Tree Shaking 是什么？Vite 和 Webpack 都支持吗？

- Tree Shaking：移除未使用的代码，减小打包体积。
- Webpack 和 Vite 都支持，但前提是必须使用 ES Module（import/export），CommonJS 不支持。

### vite生产环境优化方法

- 代码分割：使用动态 import 配合手动拆包，减少首屏体积。
- 资源压缩：压缩 JavaScript、CSS 和图片资源，必要时移除 console。
- Tree Shaking：确保使用 ES Module，移除未使用代码。
- 懒加载与按需加载：路由懒加载、组件懒加载、第三方库按需引入。
- 静态资源优化：结合 CDN、gzip 或 brotli 压缩以及浏览器缓存提升加载性能。

### webpack生产环境优化方法

- 代码分割：使用 SplitChunksPlugin 和动态 import 做拆包，减少重复代码和首屏体积。
- 资源压缩：使用 TerserPlugin 压缩 JavaScript，使用 CssMinimizerPlugin 压缩 CSS。
- Tree Shaking：确保使用 ES Module，移除未使用代码。
- 缓存优化：给静态资源加 contenthash，配合持久化缓存提升构建和加载性能。
- 构建性能优化：缩小 loader 处理范围，必要时结合多进程构建、CDN 和 gzip 压缩。

## 包管理工具

| 特性                  | npm                          | pnpm                               |
| ------------------- | ---------------------------- | ---------------------------------- |
| **安装方式**            | 将每个依赖直接拷贝到项目的 `node_modules` | 使用全局 store（缓存）并通过硬链接（symlink）安装到项目 |
| **磁盘占用**            | 较大，每个项目重复存储依赖                | 较小，全局缓存依赖，多个项目共享相同版本               |
| **安装速度**            | 相对较慢                         | 快（利用缓存和硬链接，减少重复下载和磁盘写入）            |
| **node_modules 结构** | 深层嵌套，容易产生“依赖膨胀”              | 扁平化且严格，peerDependencies 检查更严格      |
| **lock 文件**         | `package-lock.json`          | `pnpm-lock.yaml`                   |
| **monorepo 支持**     | 支持有限                         | 原生支持 workspace，依赖可在项目间共享           |
| **依赖冲突处理**          | 深层 node_modules 存在多版本冲突      | 严格报错，保证依赖唯一性                       |
| **缓存机制**            | 每个项目缓存下载包的 tar 文件            | 全局 store，所有项目共享依赖，减少重复下载           |
| **适合场景**            | 小型项目或单仓库                     | 多项目 monorepo、大型项目、高性能安装需求          |


## 手写题

### 手写简单发布订阅模式（EventBus）

```js
class PubSub {
  constructor() {
    this.events = {};
  }
  $on(event, callback) {
    if (!this.events[event]) {
      this.events[event] = [];
    }
    this.events[event].push(callback);
  }
  $emit(event, data) {
    if (this.events[event]) {
      this.events[event].forEach(callback => callback(data));
    }
  }
  $off(event) {
    this.events[event] = {};
  }
}
```

### 手写并发请求限制

```js
function limitConcurrency(tasks, limit) {
  return new Promise((resolve) => {
    const results = [];
    let index = 0;
    let activeCount = 0;
    function next() {
      if (index === tasks.length && activeCount === 0) {
        resolve(results);
        return;
      }
      while (activeCount < limit && index < tasks.length) {
        const currentIndex = index++;
        activeCount++;
        tasks[currentIndex]().then(result => {
          results[currentIndex] = result;
          activeCount--;
          next();
        });
      }
    }
    next();
  });
}
```

### 手写 new 操作符实现

```js
function myNew(Constructor, ...args) {
  // 创建新对象，原型指向构造函数的prototype
  const obj = Object.create(Constructor.prototype); 
  // 执行构造函数，传入新对象作为this
  const result = Constructor.apply(obj, args); 
  // 返回构造函数返回的对象或新对象
  return typeof result === 'object' && result !== null ? result : obj; 
}
```

## 场景

### 如何渲染大量数据列表？

```js
/**
 *  虚拟列表
 *  通过滚动距离以及每列高度计算偏移量（offsetY）和数据开始索引和结束索引
 *  */
const onScroll = (e) => {
  /**
   * 这里可以使用requestAnimationFrame进行优化减少触发频率以及保证状态更新发生在下一次重绘之前减少抖动、空白。
   */
  pendingScrollTop = e.target.scrollTop
}
<div class="container" @scroll.passive="onScroll">
    <div :style="{ height: totalHeight + 'px', position: 'relative' }">
      <div class="items" :style="{ transform: `translateY(${offsetY}px)` }">
        <div
          v-for="(item, index) in visibleItems"
          :key="startIndex + index"
          class="item"
          :style="{ height: itemHeight + 'px' }"
        >
          {{ item }}
        </div>
      </div>
    </div>
</div>
```

```vue
/** vueuse 方式 */
<script setup lang="ts">
const items = ref(
  Array.from({ length: 1000000 }, (_, i) => ({
    id: i,
    label: `Item ${i + 1}`,
  })),
);

const { list, containerProps, wrapperProps } = useVirtualList(items, {
  itemHeight: 30,
  overscan: 10,
});
</script>
<template>
  <div class="container" v-bind="containerProps">
    <div v-bind="wrapperProps">
      <div v-for="item in list" :key="item.data.id" class="item">
        {{ item.data.label }}
      </div>
    </div>
  </div>
</template>
```

# AI  

## Skills（AI技能）

**三大核心**

- 元数据：简短技能描述，帮助AI决定是否使用该技能，消耗token少。
- 行动指南：详细的执行步骤，即“提示词”的主体。
- 资源文件：用于执行具体任务的外部文件，按需加载。

## MCP

- 让 AI 能标准化调用外部能力的一套协议（大模型和外部工具之间的统一插线板）。

## Agent

- 具备自主决策能力的AI系统，能够根据环境和目标选择合适的技能执行任务。

## Agent、MCP、Skills之间的关系

- Skills：提供具体的能力和操作步骤。
- MCP：负责接工具。
- Agent：根据环境和目标选择合适的技能，通过MCP调用外部能力，实现自主决策和任务执行。
