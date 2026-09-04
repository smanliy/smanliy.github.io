---
title: 更新导致的渲染慢，你会如何排查和解决 | debug
published: 2026-09-04
description: "从原因到解决解释"
image: ""
tags: ['vue','React']
category: 技术
draft: false
---
# 更新导致的渲染慢，你会如何排查和解决

## 一、谁变了，谁被影响

> 组件执行更新逻辑 ≠ 真实 DOM 一定会变化。  
> 要把 **计算、比较、提交、绘制** 拆开，分层排查渲染卡顿。

### 1. 状态变化：Source（变化源头）

触发源：

- `props`
- `state`
- `context`
- Vue 响应式数据

> 这是更新的起点，某个数据被修改了。

### 2. 识别范围：Scope（找出哪些组件会收到更新信号）

- React：**自上而下遍历组件树**，从更新的父组件往下传导
- Vue：**响应式依赖追踪**，找到当初读取过这个数据的副作用或组件

> 卡顿源头坑点 1：识别范围过大，很多**完全不需要更新的组件被划入更新范围**，这是多余重渲染的根源。

### 3. 更新计算

执行组件渲染函数，生成新的页面描述，也就是 Virtual DOM。

> 卡顿源头坑点 2：`render` 函数里有很重的同步计算或循环，这一步 JS 会产生长任务，火焰图中 `Scripting` 耗时高。

### 4. 计算差异：Diff

新旧虚拟 DOM 对比，找出真正变化的节点。

> 卡顿源头坑点 3：超长列表 Diff 对比开销巨大，JS 耗时高。  
> 经过 Diff 后，很多组件虽然执行了 render，最后也可能判定没有变化，不需要修改 DOM。

### 5. 浏览器提交：真实 DOM 阶段

把 Diff 结果交给浏览器：

- 更新 DOM
- 修改样式
- 重排 Layout
- 重绘 Paint

> 卡顿源头坑点 4：DOM 节点太多，回流布局开销大，火焰图中 `Rendering / Layout` 耗时高。

## 二、常见原因

| 代号 | 问题名称 | 高频框架 | 核心现象 |
| :--- | :--- | :--- | :--- |
| 高 | 状态放太高 | React / Vue | 一小块数据更新，整个页面组件树刷新 |
| 宽 | 订阅范围太宽 | Vue / React Context | 一个字段改动，一大堆订阅组件被唤醒 |
| 新 | 对象身份不稳 | React | `memo` 缓存失效，子组件无理由重渲染 |
| K | 列表 key 不稳 | React / Vue | 列表增删排序时渲染异常、性能差 |
| 环 | 副作用循环 | React / Vue | 重复执行 `effect` / `watch`，甚至无限更新循环 |
| 量 | 工作量本身太大 | 全部框架 | 更新逻辑本身合理，但是 DOM 或计算量过载 |

## 三、定位问题

### 1. DevTools：查更新链路

作用：定位**多余重渲染**。

React：

- 使用 `React DevTools -> Profiler`
- 开启组件更新高亮
- 查看一次交互后，哪些组件触发了重渲染
- 判断是不是父组件更新，导致一堆无关子组件跟着刷新

Vue：

- 使用 `Vue DevTools`
- 追踪响应式依赖
- 查看组件更新记录

### 2. Chrome Performance：查真实耗时成本

Profiler 能看到组件有没有更新，但看不出：

- 更新的 JS 代码是不是超过 50ms 长任务
- 时间是耗在脚本 `Scripting`
- 还是耗在 DOM 布局 `Rendering / Layout`

Chrome Performance 火焰图可以进一步判断：

1. 如果黄色 `Scripting` 长条明显，说明 JS 计算、render、Diff 耗时长。
2. 如果紫色 `Rendering / Layout` 长条明显，说明 DOM 节点太多或布局开销太大。

| 板块名称 | 主颜色 | 核心工作 | 卡顿说明 |
| :--- | :--- | :--- | :--- |
| Scripting | 黄色 | 执行 JS、框架更新、Diff、业务逻辑 | 长任务超过 50ms，JS 阻塞主线程，是最常见的更新卡顿源 |
| Rendering | 紫色 | 样式重算、Layout 回流 | DOM 频繁改动、强制同步布局，布局开销大 |
| Painting | 绿色 | 像素绘制、重绘 | 大面积背景、图片频繁修改造成绘制压力 |
| Composite | 深蓝色 | 图层 GPU 合成 | 开销通常较低，一般不是主要卡顿来源 |

点开黄色 `Scripting` 长条后，下方会展开调用栈，可以看到更细的任务：

- 黄色：JS 执行，包括业务代码和框架代码
- 灰色：浏览器内置任务，例如 GC 垃圾回收

> 突然出现一大块灰色，通常说明频繁创建和销毁大量对象，导致 GC 卡顿。  
> 例如每次渲染都新建大数组、大对象。

## 四、优化顺序

### 1. 状态下沉

现象：局部状态定义在高层父组件，导致下面一大堆无关子组件刷新。

解决：把 `state` 下移到真正使用它的子组件内部，上层不再持有这份状态。

```js
// 优化前：state 在顶层
const [search, setSearch] = useState('')

return (
  <>
    <Search value={search} onChange={setSearch} />
    <BigList /> {/* 无辜重渲染 */}
  </>
)
```

```js
// 优化后：state 下沉
// Search 组件内部自己维护 search state
// BigList 不再被 search 更新触发重渲染
function Search() {
  const [search, setSearch] = useState('')

  return <input value={search} onChange={e => setSearch(e.target.value)} />
}
```

### 2. 拆分订阅范围 / 拆分 Context

现象：一个 `Context` 或全局仓库里放了很多互不相关的数据；修改其中一个字段，所有消费组件都刷新。

解决：

- React：拆分成多个独立 `Context`
- Vue / Pinia：拆分成更小的 store 模块
- 组件只订阅自己真正用到的数据

### 3. 修复引用不稳定

现象：父组件每次渲染都会生成新对象、新数组、新函数，传给子组件后导致 `memo` 浅比较失败。

解决：

- 函数使用 `useCallback`
- 对象、数组使用 `useMemo`

```js
// 优化前：每次父组件渲染都会生成全新函数
<Child onClick={() => doSomething()} />
```

```js
// 优化后：稳定函数引用
const handleClick = useCallback(() => {
  doSomething()
}, [])

<Child onClick={handleClick} />
```

### 4. 修复列表 key

现象：数组增删、排序后出现大规模异常重渲染，甚至渲染错乱。

解决：使用业务唯一 `id`，不要用数组下标作为 key。

```js
{list.map(item => (
  <Item key={item.id} data={item} />
))}
```

### 5. 断掉副作用循环

现象：`useEffect` / `watch` 修改状态，又再次触发自身执行，形成重复更新甚至无限循环。

解决：

- 修正依赖数组
- 删除不必要依赖
- 增加守卫条件
- 必要时加执行锁，防止循环触发

```js
useEffect(() => {
  if (!loading) return

  setLoading(false)
}, [loading])
```

### 6. 最后兜底：memo

> `memo` 属于治标不治本。  
> 更新信号依旧会传到子组件，只是通过浅比较判断 props 没变后，跳过子组件 render。

```js
const Child = React.memo(ChildComp)
```

一般优化顺序应该是：

1. 先确认是不是状态范围过大
2. 再确认订阅范围是否过宽
3. 再修复引用不稳定
4. 再检查列表 key
5. 再排查副作用循环
6. 最后才考虑 `memo`