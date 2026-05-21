---
title: 前端面试题-手写题
date: 2026-05-21 15:00:00
tags: [前端, 面试, 手写题]
categories: 前端面试
---

## 手写简单发布订阅模式（EventBus）

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

## 手写并发请求限制

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

## 手写 new 操作符实现

## hooks

### socket

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

### queue

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

### 分页 usePaginationList

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