---
title: 前端面试题-场景
date: 2026-05-21 15:00:00
tags: [前端, 面试, 场景]
categories: 前端面试
---

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

