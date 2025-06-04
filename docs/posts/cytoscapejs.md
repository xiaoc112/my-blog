---
title: Cytoscape.js 详细学习笔记
description: Cytoscape.js图论库的使用教程，包括节点、边的设置，样式配置，布局算法等核心内容
date: 2023-04-29
sticky: 1
tags:
  - JavaScript
  - 可视化
  - 图论
---

# Cytoscape.js 详细笔记

## 📋 目录

- [Cytoscape.js 详细笔记](#cytoscapejs-详细笔记)
  - [📋 目录](#-目录)
  - [简介](#简介)
  - [基本概念](#基本概念)
  - [安装](#安装)
    - [npm 安装](#npm-安装)
    - [CDN 引入](#cdn-引入)
  - [初始化](#初始化)
  - [元素（Elements）](#元素elements)
    - [添加节点和边](#添加节点和边)
  - [样式（Style）](#样式style)
    - [常用属性](#常用属性)
  - [布局（Layout）](#布局layout)
    - [常见布局](#常见布局)
  - [事件（Events）](#事件events)
  - [常用APIs总结](#常用apis总结)
  - [示例代码](#示例代码)
  - [常见问题](#常见问题)
    - [容器没有大小，图显示不出来？](#容器没有大小图显示不出来)
    - [修改元素样式后没有变化？](#修改元素样式后没有变化)
    - [布局变化没有自动更新？](#布局变化没有自动更新)
- [✨ 小结](#-小结)

## 简介

Cytoscape.js 是一个纯 JavaScript 的图论（Graph Theory）库，可以用于构建复杂的节点+边的关系图。
支持大规模数据、交互式操作、动画、各种布局算法。

官网：[https://js.cytoscape.org/](https://js.cytoscape.org/)

适用场景：
- 知识图谱
- 社交网络
- 流程图
- 网络拓扑

## 基本概念

| 概念 | 说明 |
|:---|:---|
| Node（节点） | 图中的点 |
| Edge（边） | 节点之间的连接 |
| Elements | 节点和边的合集 |
| Style | 样式表，控制节点/边的颜色、形状等 |
| Layout | 布局算法，决定节点的位置 |

## 安装

### npm 安装

```bash
npm install cytoscape
```

### CDN 引入

```html
<script src="https://unpkg.com/cytoscape/dist/cytoscape.min.js"></script>
```

## 初始化

```javascript
const cy = cytoscape({
  container: document.getElementById('cy'), // 容器元素

  elements: [], // 元素列表

  style: [], // 样式

  layout: { name: 'grid' }, // 布局
});
```

## 元素（Elements）

### 添加节点和边

```javascript
elements: [
  { data: { id: 'a' } },
  { data: { id: 'b' } },
  { data: { id: 'ab', source: 'a', target: 'b' } }
]
```

- `id`: 元素 ID（必须唯一）
- `source`: 边的起点（节点 id）
- `target`: 边的终点（节点 id）

动态添加元素：

```javascript
cy.add({ data: { id: 'c' } });
cy.add({ data: { id: 'ac', source: 'a', target: 'c' } });
```

## 样式（Style）

类似 CSS，定义节点和边的样式：

```javascript
style: [
  {
    selector: 'node',
    style: {
      'background-color': '#0074D9',
      'label': 'data(id)'
    }
  },
  {
    selector: 'edge',
    style: {
      'width': 3,
      'line-color': '#ccc',
      'target-arrow-color': '#ccc',
      'target-arrow-shape': 'triangle',
      'curve-style': 'bezier'
    }
  }
]
```

### 常用属性

- 节点：`background-color`, `label`, `shape`, `width`, `height`
- 边：`line-color`, `width`, `target-arrow-shape`, `curve-style`

## 布局（Layout）

### 常见布局

| 布局名 | 说明 |
|:---|:---|
| grid | 网格布局 |
| circle | 圆形布局 |
| concentric | 同心圆布局 |
| breadthfirst | 层级布局 |
| cose | 力导向布局（常用） |

设置布局：

```javascript
cy.layout({ name: 'cose' }).run();
```

动态重新布局：

```javascript
cy.elements().layout({ name: 'circle' }).run();
```

## 事件（Events）

监听节点点击：

```javascript
cy.on('tap', 'node', function(evt){
  const node = evt.target;
  console.log('Clicked node:', node.id());
});
```

监听边点击：

```javascript
cy.on('tap', 'edge', function(evt){
  const edge = evt.target;
  console.log('Clicked edge:', edge.id());
});
```

更多事件：
- `mouseover`, `mouseout`
- `dragfree`, `position`

## 常用APIs总结

| API | 说明 |
|:---|:---|
| `cy.add()` | 添加元素 |
| `cy.remove()` | 删除元素 |
| `cy.getElementById(id)` | 获取指定元素 |
| `ele.data(key, value)` | 修改数据 |
| `ele.style(property, value)` | 修改样式 |
| `cy.layout({...}).run()` | 重新布局 |
| `cy.zoom()`, `cy.pan()` | 缩放和平移视图 |
| `cy.center()` | 将元素居中显示 |
| `cy.fit()` | 缩放到合适大小 |

## 示例代码

```html
<div id="cy" style="width: 600px; height: 400px;"></div>
<script src="https://unpkg.com/cytoscape/dist/cytoscape.min.js"></script>
<script>
  const cy = cytoscape({
    container: document.getElementById('cy'),
    elements: [
      { data: { id: 'a' } },
      { data: { id: 'b' } },
      { data: { id: 'ab', source: 'a', target: 'b' } }
    ],
    style: [
      { selector: 'node', style: { 'background-color': '#0074D9', 'label': 'data(id)' } },
      { selector: 'edge', style: { 'width': 2, 'line-color': '#ccc', 'target-arrow-shape': 'triangle' } }
    ],
    layout: { name: 'grid' }
  });
</script>
```

## 常见问题

### 容器没有大小，图显示不出来？

容器元素必须有 **固定宽度高度**！否则 Cytoscape.js 无法渲染。

### 修改元素样式后没有变化？

修改样式后需要触发渲染，比如：

```javascript
node.style('background-color', '#f00');
cy.style().update();
```

### 布局变化没有自动更新？

布局要 `.run()` 才生效：

```javascript
cy.layout({ name: 'cose' }).run();
```

---

# ✨ 小结

Cytoscape.js 本质是：
- 创建元素（节点、边）
- 设置样式
- 布局排布
- 响应交互

掌握这4个核心点，基本就可以开始做各种酷炫的关系图了！ 