要深入分析 `RecyclerView.tryGetViewHolderForPositionByDeadline` 的逻辑，需要结合 Android 源码（以 API 33 为例），重点关注其**多级缓存查找流程**和**复用策略**。以下是核心逻辑的源码级拆解：


### **方法定义与核心变量**
方法定义在 `RecyclerView.Recycler` 类中（RecyclerView 的内部类，负责管理 ViewHolder 缓存）：
```java
ViewHolder tryGetViewHolderForPositionByDeadline(int position, boolean dryRun, long deadlineNs) {
    // 省略边界检查和预布局处理...
}
```
核心目标：根据 `position` 查找或创建 ViewHolder，优先复用缓存以提升性能。


### **核心逻辑步骤（源码流程）**
以下是方法中获取 ViewHolder 的关键步骤（附源码注释）：


#### **1. 预处理：检查 position 合法性 & 稳定 ID**
```java
// 步骤1：检查 position 合法性（避免越界）
if (position < 0 || position >= mState.getItemCount()) {
    return null;
}

// 步骤2：获取 itemId（若启用稳定 ID，优先用 ID 匹配缓存）
final boolean stableIds = mAdapter.hasStableIds();
long itemId = stableIds ? mAdapter.getItemId(position) : NO_ID;
```
若 Adapter 启用了稳定 ID（`setHasStableIds(true)`），后续缓存查找会优先通过 `itemId` 匹配，避免因位置变化导致的缓存失效。


#### **2. 第一级缓存：Scrap 缓存（屏幕内临时分离的 ViewHolder）**
Scrap 缓存存储**当前屏幕可见但被临时分离的 ViewHolder**（如布局刷新、动画时），优先从这里查找。  
源码中通过 `mAttachedScrap`（未标记移除的 ViewHolder）和 `mChangedScrap`（需要更新的 ViewHolder）查找：
```java
// 尝试从 mAttachedScrap 中查找（未标记移除的 ViewHolder）
for (int i = 0; i < scrapCount; i++) {
    final ViewHolder holder = mAttachedScrap.get(i);
    if (holder.mPosition == position && !holder.isInvalid() 
        && (stableIds ? holder.getItemId() == itemId : true)) {
        return holder; // 直接返回匹配的 ViewHolder
    }
}

// 若 mAttachedScrap 未找到，尝试 mChangedScrap（需更新的 ViewHolder）
for (ViewHolder holder : mChangedScrap) {
    if (holder.mPosition == position && !holder.isInvalid() 
        && (stableIds ? holder.getItemId() == itemId : true)) {
        return holder;
    }
}
```
- **关键逻辑**：通过 `position` 或 `itemId` 匹配，直接复用无需重新绑定数据。
- **适用场景**：布局过程中临时分离的 ViewHolder（如滚动时的局部刷新）。


#### **3. 第二级缓存：CachedViews（最近移除的 ViewHolder）**
`mCachedViews` 存储**最近被移除屏幕的 ViewHolder**（默认容量 2），用于快速恢复“可能立即重新进入屏幕”的 View（如快速滑动后的回退）。  
源码中遍历 `mCachedViews` 查找匹配项：
```java
int cachedSize = mCachedViews.size();
for (int i = 0; i < cachedSize; i++) {
    final ViewHolder holder = mCachedViews.get(i);
    if (holder.mPosition == position && !holder.isInvalid() 
        && (stableIds ? holder.getItemId() == itemId : true)) {
        // 命中缓存，从 mCachedViews 中移除并返回
        mCachedViews.remove(i);
        return holder;
    }
}
```
- **关键逻辑**：缓存按插入顺序存储，优先匹配 `position` 或 `itemId`。
- **注意**：若 `mCachedViews` 已满（默认 2），旧数据会被移除并降级到 `RecycledViewPool`。


#### **4. 第三级缓存：ViewCacheExtension（用户自定义缓存）**
`ViewCacheExtension` 是用户可自定义的缓存扩展（需重写 `getViewForPositionAndType`），源码中仅在特定条件下调用：
```java
if (mViewCacheExtension != null) {
    // 尝试从自定义缓存中获取（需用户实现）
    final View view = mViewCacheExtension.getViewForPositionAndType(this, position, type);
    if (view != null) {
        final ViewHolder holder = getChildViewHolder(view);
        // 验证 holder 有效性（如未被回收）
        return holder;
    }
}
```
- **适用场景**：需自定义复杂缓存策略时（如预加载特定类型的 View）。
- **注意**：默认未启用，需用户主动实现。


#### **5. 第四级缓存：RecycledViewPool（按类型分类的通用缓存）**
`RecycledViewPool` 存储**按 `itemType` 分类的 ViewHolder**（每个类型默认缓存 5 个），用于跨 RecyclerView 复用（如嵌套滚动场景）。  
源码中通过 `itemType` 查找对应缓存池：
```java
// 根据 itemType 获取缓存池
RecycledViewPool.Pool pool = mRecyclerPool.getPoolForType(type);
if (pool != null) {
    // 从池中获取 ViewHolder（可能已被重置）
    ViewHolder holder = pool.getRecycledView();
    if (holder != null) {
        // 重置 ViewHolder 状态（如清除动画、标记为未绑定）
        holder.resetInternal();
        return holder;
    }
}
```
- **关键逻辑**：ViewHolder 会被重置（`resetInternal()`），需重新绑定数据（`bindViewHolder`）。
- **适用场景**：长列表滚动时，不同位置但相同类型的 View 复用。


#### **6. 未命中缓存：创建并绑定新 ViewHolder**
若四级缓存均未命中，最终会调用 Adapter 创建并绑定 ViewHolder：
```java
// 创建新 ViewHolder（调用 Adapter.onCreateViewHolder）
ViewHolder holder = mAdapter.createViewHolder(RecyclerView.this, type);
// 绑定数据（调用 Adapter.onBindViewHolder）
bindViewHolder(holder, position);
return holder;
```
- **性能代价**：创建和绑定是耗时操作，需尽量通过缓存避免。


### **关键源码总结**
| 缓存级别       | 缓存结构               | 匹配条件               | 是否需要重新绑定 |
|----------------|------------------------|------------------------|------------------|
| Scrap          | `mAttachedScrap`/`mChangedScrap` | `position` 或 `itemId` | 否（仅 `mChangedScrap` 需更新） |
| CachedViews    | `mCachedViews`（列表） | `position` 或 `itemId` | 否               |
| ViewCacheExtension | 用户自定义           | 自定义逻辑             | 否               |
| RecycledViewPool | `RecycledViewPool`（按类型） | `itemType`             | 是（需重新绑定） |


### **补充：截止时间（deadlineNs）的作用**
方法名中的 `ByDeadline` 指**带超时控制的查找**，源码中通过 `deadlineNs` 判断是否继续查找：
```java
// 若当前时间超过截止时间，提前返回 null（避免阻塞主线程）
if (deadlineNs != FOREVER_NS && System.nanoTime() > deadlineNs) {
    return null;
}
```
该机制用于保证布局过程的流畅性，避免在缓存查找上耗时过长（如极端卡顿场景）。


### **总结**
`tryGetViewHolderForPositionByDeadline` 的核心是**多级缓存的优先级查找**：  
优先复用屏幕内临时分离的 Scrap 缓存 → 最近移除的 CachedViews → 用户自定义缓存 → 通用 RecycledViewPool → 最后创建新实例。通过这种策略，RecyclerView 实现了高效的 View 复用，确保了滚动的流畅性。