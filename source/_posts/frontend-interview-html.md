---
title: 前端面试题-HTML
date: 2026-05-21 15:00:00
tags: [前端, 面试, HTML]
categories: 前端面试
---

## 说说对 html 语义化的理解

- 定义：HTML 语义化是指使用具有明确意义的 HTML 标签来描述页面内容和结构，使得网页更易于理解和维护。
- 优点：
  - 提升可读性和可维护性。
  - 有助于搜索引擎优化（SEO）。
  - 利于无障碍访问，辅助技术可以更好地理解页面结构。

## 前端页面有哪三层构成，分别是什么？

- 结构层：由 HTML 组成，定义了网页的内容和结构。
- 表现层：由 CSS 组成，负责网页的样式和布局。
- 行为层：由 JavaScript 组成，负责网页的交互和动态效果。

## 行级元素和块级元素分别有哪些及怎么转换？

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
  
## H5有哪些新元素和新特性

- 语义话标签：header、nav、section、article、aside、footer等
- 视频video、音频audio
- 画布canvas
- 表单控件，calemdar、date、time、email
- 地理：geolocation API
- 本地离线存储，localStorage长期存储数据，浏览器关闭后数据不丢失，sessionStorage的数据在浏览器关闭后自动删除
- 拖拽释放

## iframe的作用以及优缺点

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

## meta viewport 是做什么用的，怎么写？
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
