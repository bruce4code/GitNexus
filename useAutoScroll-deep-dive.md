# 深度解析 React 中优雅的自动滚动：useAutoScroll Hook 实现

> 在 AI 聊天应用中，有一个看似简单但实现起来充满细节的交互：**AI 正在流式输出时，用户上滚查看历史消息，自动滚动应该暂停；当用户滚回底部时，自动滚动恢复。** 本文从 GitNexus 项目中的 `useAutoScroll` hook 出发，剖析这个交互背后的精妙设计。

## 问题定义

先明确需求：

- AI 回答是流式输出的，消息内容持续增长
- 用户可能在任何时刻上滚查看历史消息
- 用户上滚时，**不能**抢夺用户的视口焦点
- 用户滚回底部后，**自动恢复**跟随
- 整个过程必须流畅，不能有闪烁或卡顿

## 整体架构

```typescript
export function useAutoScroll<T>(
  chatMessages: T[],        // 消息列表——内容变化的驱动力
  isChatLoading: boolean,   // 是否正在加载
  bottomThreshold = 100,    // "底部"判定阈值
): UseAutoScrollResult {
```

返回三个东西：
1. `scrollContainerRef` → 可滚动的外层容器
2. `messagesContainerRef` → 内容容器（用于 ResizeObserver）
3. `isAtBottom` → 是否在底部，供 UI 控制浮动按钮
4. `scrollToBottom` → 手动滚动到底部

## 核心设计：State 与 Ref 的分工

这是整个实现最精髓的决策。

### 错误示范

新手容易这样写：

```tsx
const [isAtBottom, setIsAtBottom] = useState(true);

// 滚动事件监听
const handleScroll = () => {
  setIsAtBottom(/* 判断是否在底部 */);
};

// ResizeObserver 回调
const observer = new ResizeObserver(() => {
  if (isAtBottom) {  // ← 闭包陷阱！
    scrollToBottom();
  }
});
```

问题在哪？`isAtBottom` 被 ResizeObserver 回调闭包捕获，永远是最初的值。常见的解法是把 `isAtBottom` 加入 `useEffect` 依赖：

```tsx
useEffect(() => {
  const observer = new ResizeObserver(() => {
    if (isAtBottom) { ... }
  });
  // ...
}, [isAtBottom]); // ← 每次 isAtBottom 变化都重建 Observer
```

这会频繁销毁和重建 Observer，浪费性能。

### 正解：各司其职

```typescript
const [isAtBottom, setIsAtBottom] = useState(true);     // 用于 UI 渲染
const shouldStickToBottomRef = useRef(true);             // 用于回调逻辑
```

| 变量 | 用途 | 特征 |
|------|------|------|
| `isAtBottom` (state) | 控制浮动按钮显隐、触发 React 重渲染 | 变化时需重新渲染 UI |
| `shouldStickToBottomRef` (ref) | 回调中判断"是否要跟随滚动" | 需跨闭包读最新值，变化不应触发重渲染 |

ResizeObserver 回调里读 ref：

```typescript
const observer = new ResizeObserver(() => {
  if (shouldStickToBottomRef.current) {  // ← 永远是最新的
    scrollToBottom('auto');
  } else {
    syncScrollState();
  }
});
```

Observer 只需创建一次，依赖数组空了也不怕——读到的永远是当前最新的值。

## 智能的滚动方向检测

```typescript
const syncScrollState = () => {
  const nearBottom = isNearBottom(element, bottomThreshold);
  const currentScrollTop = element.scrollTop;

  if (nearBottom) {
    shouldStickToBottomRef.current = true;         // 在底部 → 跟随
  } else if (currentScrollTop < lastScrollTopRef.current - USER_SCROLL_EPSILON) {
    shouldStickToBottomRef.current = false;        // 明确上滚 → 停止跟随
  }
  // 其他情况（向下滚但还没到底）→ 保持现有状态
};
```

关键细节：

1. **只有"在底部"和"明确上滚"会改变状态**。用户正在向下滚动但还没到底时，状态不变——这样即使 AI 输出导致内容变长，也不会误判。

2. **`USER_SCROLL_EPSILON = 5`** 是一个防抖阈值。滚动惯性或像素对齐可能导致 `scrollTop` 有几像素的抖动，5px 以下的变化被认为不是用户主动操作。

3. **同步更新 `lastScrollTopRef`**，用于方向判断。

## 三重 rAF 节流体系

这个 hook 在三个地方使用了 `requestAnimationFrame` 节流，每一层都有特定目的。

### 第一层：滚动事件节流

```typescript
const handleScroll = () => {
  if (scrollFrameIdRef.current !== null) {
    cancelAnimationFrame(scrollFrameIdRef.current);  // 取消上次排队的
  }
  scrollFrameIdRef.current = requestAnimationFrame(() => {
    scrollFrameIdRef.current = null;
    syncScrollState();
  });
};
```

这是**首尾取消模式**：每次滚动事件先取消上一次 rAF，再排新的。效果是只有**最后一次滚动事件的下一帧**才执行检查，避免了高频滚动导致的状态抖动。

### 第二层：ResizeObserver 节流

```typescript
const observer = new ResizeObserver(() => {
  if (shouldStickToBottomRef.current) {
    if (resizeFrameId !== null) {
      cancelAnimationFrame(resizeFrameId);
    }
    resizeFrameId = requestAnimationFrame(() => {
      resizeFrameId = null;
      scrollToBottom('auto');
    });
  } else {
    syncScrollState();
  }
});
```

当用户在底部时，内容增长触发的滚动操作也经过 rAF 节流，保证与浏览器帧同步。当用户不在底部时，只更新状态（`syncScrollState`）但不滚动。

### 第三层：流式更新的 rAF 调度（外部配合）

在 `useAppState.tsx` 中，流式消息的 React state 更新也经过 rAF 去重：

```typescript
const scheduleMessageUpdate = () => {
  if (pendingUpdate) return;      // 去重
  pendingUpdate = true;
  rafHandle = requestAnimationFrame(() => {
    pendingUpdate = false;
    rafHandle = null;
    updateMessage();
  });
};
```

这保证了每秒最多 60 次 React state 更新，且与浏览器帧对齐。

## ResizeObserver 的妙用

为什么选择 `ResizeObserver` 而不是 `MutationObserver`？

```typescript
observer.observe(content);  // 监听消息容器
```

消息内容变化导致容器尺寸变化 → ResizeObserver 触发 → 判断是否要滚动。

`ResizeObserver` 相比 `MutationObserver` 的优势：
- **语义更精确**：我们关心的是"内容变大了多少"，而非"具体哪个 DOM 节点变了"
- **性能更好**：不会因为子节点的属性变化而触发
- **自身去重**：同一帧内的多次尺寸变化会合并触发

## useLayoutEffect 消除闪烁

```typescript
useLayoutEffect(() => {
  if (!shouldStickToBottomRef.current) return;
  scrollToBottom('auto');
}, [chatMessages.length, isChatLoading, scrollToBottom]);
```

`useLayoutEffect` 在**浏览器绘制之前**同步执行。这意味着：
- `useEffect`：先绘制（用户可能看到新消息出现在视口中间），再滚动到（用户看到闪烁）
- `useLayoutEffect`：先在 JS 中滚动到底部，再绘制（用户直接看到已滚好的状态）

在滚动场景下，这消除了视觉闪烁的最后一毫秒。

## 完整状态机

```
                    ┌─────────────────────────┐
                    │   shouldStickToBottom     │
                    │        = true             │
                    │   isAtBottom = true       │
                    │   (跟随模式)              │
                    └───────────┬──────────────┘
                                │
                    AI 流式输出，内容增长
                    ResizeObserver → 自动滚到底
                                │
                    ─── 用户上滚查看历史 ───
                                │
                    ┌───────────▼──────────────┐
                    │   shouldStickToBottom     │
                    │        = false            │
                    │   isAtBottom = false      │
                    │   (浏览模式)              │
                    └───────────┬──────────────┘
                                │
                    AI 持续输出，内容增长
                    ResizeObserver 只检查不滚动
                    浮动 "回到底部" 按钮出现
                                │
                    ─── 用户滚回底部 ───
                    ─── 或点击浮动按钮 ───
                                │
                    ┌───────────▼──────────────┐
                    │   shouldStickToBottom     │
                    │        = true             │
                    │   isAtBottom = true       │
                    │   (跟随模式)              │
                    └───────────┬──────────────┘
                                │
                    自动跟随恢复，按钮消失
```

## 总结

这个 `useAutoScroll` hook 展示了几个重要的 React 模式：

1. **Ref 与 State 的分治**：UI 渲染用 state，回调逻辑用 ref。避免闭包陷阱的同时，也避免了 Observer 的不必要重建。
2. **rAF 的三层节流**：滚动事件、ResizeObserver、React state 更新各自独立节流，互不干扰。
3. **ResizeObserver 的语义化选择**：用尺寸变化而非 DOM 变化来驱动滚动逻辑。
4. **useLayoutEffect 的精确时机**：在绘制前完成滚动，消除闪烁。
5. **小阈值的大作用**：5px 防抖阈值让方向检测更鲁棒。

这些模式在很多场景下都可以复用——无论是聊天窗口、日志查看器，还是实时数据大屏，核心思路都是相通的。