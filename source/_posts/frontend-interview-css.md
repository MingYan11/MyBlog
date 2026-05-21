---
title: 前端面试题-CSS
date: 2026-05-21 15:00:00
tags: [前端, 面试, CSS]
categories: 前端面试
---

## 盒模型
- 定义：CSS 盒模型是指在网页布局中，每个元素都被看作一个矩形盒子，由内容区、内边距（padding）、边框（border）和外边距（margin）组成。
- 标准盒模型：content-box，元素的宽度和高度只包括内容区，不包括内边距和边框。
- IE 盒模型：border-box，元素的宽度和高度包括内容区、内边距和边框。
- 切换盒模型：box-sizing。
- 外边距合并：
  - 两个相邻元素的 margin-top / margin-bottom 取 最大值，而不是相加。
  - 当父元素没有 border / padding / 内容时，父元素的 margin-top 与第一个子元素的 margin-top 发生折叠，父元素的 margin-bottom 与最后一个子元素的 margin-bottom 也会发生折叠。

## CSS所有选择器及其优先级，哪些可以继承

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

## CSS 布局方式

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

## BFC

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

## 实现水平垂直居中方式

| 方法                     | 水平居中 | 垂直居中 | 适用场景                 |
| ---------------------- | ---- | ---- | -------------------- |
| `text-align: center`   | ✅    | ❌    | 行内 / inline-block 元素 |
| `margin: 0 auto`       | ✅    | ❌    | 块级元素，已知宽度            |
| `line-height`          | ✅    | ✅    | 单行文本                 |
| `absolute + transform` | ✅    | ✅    | 块级元素、多行内容            |
| Flex                   | ✅    | ✅    | 现代布局，通用              |
| Grid                   | ✅    | ✅    | 现代布局，复杂结构            |

## less 和 sass 的区别

- Sass的安装需要Ruby环境，是在服务端处理的，而Less是需要引入less.js来处理Less代码输出css到浏览器，也可以在开发环节使用Less，然后编译成css文件，直接放到项目中

  ![alt text](./images/less%20vs%20sass.png)
