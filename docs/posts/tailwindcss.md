---
title: TailwindCSS 常用样式笔记
description: TailwindCSS实用工具类速查表，包含布局、尺寸、颜色、文本等常用样式类整理
date: 2023-04-28
sticky: 2
tags:
  - CSS
  - 前端开发
  - TailwindCSS
---

# TailwindCSS 常用样式笔记

## 📋 目录

- [TailwindCSS 常用样式笔记](#tailwindcss-常用样式笔记)
  - [📋 目录](#-目录)
  - [布局（Layout）](#布局layout)
  - [尺寸（Width/Height）](#尺寸widthheight)
  - [边距与内距（Margin/Padding）](#边距与内距marginpadding)
  - [边框（Border）](#边框border)
  - [背景（Background）](#背景background)
  - [字体（Text）](#字体text)
  - [弹性布局（Flexbox）](#弹性布局flexbox)
  - [阴影（Shadow）](#阴影shadow)
  - [其他杂项（Other Utilities）](#其他杂项other-utilities)
  - [实战示例（Example）](#实战示例example)
  - [小总结（快速记忆）](#小总结快速记忆)
  - [✨ 备注](#-备注)

## 布局（Layout）

| 类名 | 含义 | 示例 |
|:---|:---|:---|
| `flex` | 弹性盒子 | `flex items-center justify-between` |
| `grid` | 网格布局 | `grid grid-cols-3 gap-4` |
| `block` | 块元素 | `block` |
| `inline-block` | 行内块元素 | `inline-block` |
| `hidden` | 隐藏元素 | `hidden` |

## 尺寸（Width/Height）

| 类名 | 含义 | 示例 |
|:---|:---|:---|
| `w-4`, `w-1/2`, `w-full` | 宽度设置 | `w-full` |
| `h-8`, `h-1/2`, `h-screen` | 高度设置 | `h-screen` |
| `min-w-0`, `min-h-screen` | 最小宽高 | `min-h-screen` |
| `max-w-xs`, `max-h-96` | 最大宽高 | `max-w-4xl` |

## 边距与内距（Margin/Padding）

| 类名 | 含义 | 示例 |
|:---|:---|:---|
| `m-4`, `mx-2`, `my-3` | 外边距 | `mt-4`, `mr-4`, `mb-4`, `ml-4` |
| `p-4`, `px-2`, `py-3` | 内边距 | `px-4`, `py-2` |

## 边框（Border）

| 类名 | 含义 | 示例 |
|:---|:---|:---|
| `border` | 默认 1px 边框 | `border` |
| `border-2`, `border-4` | 指定边框宽度 | `border-2` |
| `border-gray-300` | 设置边框颜色 | `border-gray-300` |
| `rounded`, `rounded-lg`, `rounded-full` | 圆角 | `rounded-full` |

## 背景（Background）

| 类名 | 含义 | 示例 |
|:---|:---|:---|
| `bg-gray-100`, `bg-blue-500` | 背景色 | `bg-red-300` |
| `bg-gradient-to-r` | 线性渐变背景 | `bg-gradient-to-r from-green-400 to-blue-500` |

## 字体（Text）

| 类名 | 含义 | 示例 |
|:---|:---|:---|
| `text-sm`, `text-lg`, `text-2xl` | 字体大小 | `text-3xl` |
| `font-bold`, `font-light`, `font-semibold` | 字重 | `font-semibold` |
| `text-gray-500`, `text-blue-600` | 字体颜色 | `text-gray-400` |
| `text-center`, `text-left`, `text-right` | 文本对齐 | `text-center` |

## 弹性布局（Flexbox）

| 类名 | 含义 | 示例 |
|:---|:---|:---|
| `flex-row`, `flex-col` | 方向 横向 / 纵向 | `flex-col` |
| `justify-start`, `justify-center`, `justify-between` | 主轴对齐 | `justify-between` |
| `items-start`, `items-center`, `items-end` | 交叉轴对齐 | `items-center` |
| `gap-4` | 元素间距 | `gap-2` |

## 阴影（Shadow）

| 类名 | 含义 | 示例 |
|:---|:---|:---|
| `shadow`, `shadow-md`, `shadow-lg` | 阴影大小 | `shadow-xl` |
| `shadow-none` | 移除阴影 | `shadow-none` |

## 其他杂项（Other Utilities）

| 类名 | 含义 | 示例 |
|:---|:---|:---|
| `overflow-hidden`, `overflow-auto`, `overflow-scroll` | 内容溢出处理 | `overflow-hidden` |
| `cursor-pointer` | 鼠标指针变成点击手势 | `cursor-pointer` |
| `transition`, `duration-300` | 动效过渡 | `transition duration-300` |
| `opacity-50`, `opacity-75` | 透明度 | `opacity-75` |
| `z-10`, `z-50` | 层级控制 | `z-50` |

## 实战示例（Example）

```html
<div class="w-full max-w-sm mx-auto mt-10 p-6 bg-white rounded-lg shadow-lg flex flex-col items-center">
  <h1 class="text-2xl font-bold text-gray-800 mb-4">欢迎来到 Tailwind！</h1>
  <p class="text-gray-600 text-center mb-6">这是一个使用 TailwindCSS 快速搭建的小卡片。</p>
  <button class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded transition duration-300">
    点击我
  </button>
</div>
```

## 小总结（快速记忆）

```
布局：flex grid block hidden
尺寸：w- h- min- max-
边距/内距：m- p- x y
边框：border rounded
背景：bg-
字体：text- font-
弹性：justify- items- gap-
溢出处理：overflow-
鼠标指针：cursor-pointer
动画过渡：transition duration-
透明度：opacity-
层级控制：z-
```

## ✨ 备注

如果想做**响应式**（根据屏幕尺寸变化），加前缀：

- `sm:` 小屏幕
- `md:` 中等屏幕
- `lg:` 大屏幕
- `xl:` 更大屏幕
- `2xl:` 超大屏幕

例子：

```html
<div class="w-full md:w-1/2">
```
表示手机上宽度100%，中等屏幕以上宽度50%。 