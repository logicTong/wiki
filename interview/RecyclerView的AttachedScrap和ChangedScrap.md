要理解 `ViewHolder` 何时被添加到 `mAttachedScrap` 和 `mChangedScrap`，需要结合 RecyclerView 的**布局流程**和**数据更新机制**，并参考源码中的关键方法（如 `LayoutManager.onLayoutChildren()`、`RecyclerView.dispatchLayout()` 等）。以下是具体场景和源码级分析：


### **Scrap 缓存的核心作用**
`mAttachedScrap` 和 `mChangedScrap` 是 RecyclerView 的**一级缓存**，用于存储**当前屏幕可见但被临时分离（Detached）的 `ViewHolder`**。这些 `ViewHolder` 不会被立即回收，而是暂时“挂起”，等待布局过程中重新附着（Reattach）或更新。  


### **触发添加到 Scrap 缓存的核心场景**
Scrap 缓存的填充主要发生在 **布局过程中**（尤其是 `LayoutManager` 重新布局时）。以下是关键场景和源码逻辑：


#### **场景 1：布局刷新（如滚动、首次布局）**
当 RecyclerView 进行布局（如首次加载、滚动或 `requestLayout()` 触发布局）时，`LayoutManager` 会先将当前所有可见的 `ViewHolder` 分离（Detach），暂存到 Scrap 缓存中，再重新排列它们的位置。这一过程通过 `LayoutManager.detachAndScrapView()` 或 `detachAndScrapAttachedViews()` 实现。


##### **源码：LayoutManager.detachAndScrapView()**
```java
// 来源：RecyclerView.LayoutManager
public void detachAndScrapView(View child, Recycler recycler) {
    final ViewHolder vh = getChildViewHolderInt(child);
    if (vh.isInvalid() && !vh.isRemoved() && !mRecyclerView.mAdapter.hasStableIds()) {
        // ViewHolder 无效且未被移除，且未启用稳定 ID → 不缓存，直接回收
        recycler.recycleView(child); 
    } else {
        // 将 ViewHolder 添加到 Scrap 缓存（mAttachedScrap）
        recycler.scrapView(child); 
    }
}
```
- **`scrapView()` 方法**：最终调用 `Recycler.scrapView()`，将 `ViewHolder` 添加到 `mAttachedScrap`。
  ```java
  // 来源：RecyclerView.Recycler
  void scrapView(View view) {
      final ViewHolder holder = getChildViewHolderInt(view);
      if (holder.hasAnyOfTheFlags(ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_INVALID) 
          || !holder.isUpdated() || canReuseUpdatedViewHolder(holder)) {
          // 若 ViewHolder 未被标记为移除/无效，添加到 mAttachedScrap
          mAttachedScrap.add(holder);
      } else {
          // 若需要更新（如 notifyItemChanged），添加到 mChangedScrap
          mChangedScrap.add(holder);
      }
  }
  ```
- **关键逻辑**：  
  普通布局刷新时，可见的 `ViewHolder` 会被分离并添加到 `mAttachedScrap`（未被标记为需要更新）；若 `ViewHolder` 因数据变化需要更新（如 `notifyItemChanged`），则会被添加到 `mChangedScrap`。  


#### **场景 2：数据更新（如 notifyItemChanged）**
当调用 `Adapter.notifyItemChanged(position)` 时，RecyclerView 会标记对应位置的 `ViewHolder` 为“需要更新”，并在布局时将其分离到 `mChangedScrap`，以便后续重新绑定数据。


##### **源码：RecyclerView 处理数据更新**
当 `Adapter` 通知数据变化时，RecyclerView 会生成 `UpdateOp`（更新操作），并在布局时处理这些操作。在 `LayoutManager.onLayoutChildren()` 中，会遍历所有可见 `ViewHolder`，检查是否需要更新：
```java
// 来源：RecyclerView.LayoutManager
public void onLayoutChildren(Recycler recycler, State state) {
    // 分离所有可见 ViewHolder 到 Scrap 缓存
    detachAndScrapAttachedViews(recycler); 

    // 遍历 Scrap 缓存，处理更新
    for (int i = 0; i < scrapList.size(); i++) {
        ViewHolder vh = scrapList.get(i);
        if (vh.mPosition == position && state.isPreLayout()) { 
            // 预布局阶段（如动画），标记需要更新的 ViewHolder
            if (vh.isInvalid() && !vh.isRemoved() && !mAdapter.hasStableIds()) {
                // 无效且未启用稳定 ID → 不缓存
            } else {
                // 添加到 mChangedScrap（需要更新）
                mChangedScrap.add(vh);
            }
        }
    }
}
```
- **触发条件**：  
  `notifyItemChanged` 会触发 `ViewHolder` 标记为 `FLAG_UPDATE`（通过 `ViewHolder.addFlags()`），布局时该 `ViewHolder` 会被分离到 `mChangedScrap`，等待重新绑定数据（`onBindViewHolder`）。  


#### **场景 3：执行动画（如 Item 增删）**
RecyclerView 的动画机制（如 `DefaultItemAnimator`）依赖 Scrap 缓存。在动画执行前，会将旧状态的 `ViewHolder` 分离到 `mAttachedScrap` 或 `mChangedScrap`，动画结束后再重新附着或回收。


##### **源码：ItemAnimator 与布局协作**
```java
// 来源：RecyclerView.ItemAnimator
public boolean animateRemove(ViewHolder holder) {
    // 将需要删除的 ViewHolder 暂存到 Scrap 缓存（mAttachedScrap）
    dispatchRemoveStarting(holder); 
    return true;
}

// 来源：RecyclerView
void dispatchLayout() {
    // 布局前阶段（Step 1）：处理预布局（Pre-layout），分离 ViewHolder 到 Scrap
    dispatchLayoutStep1(); 
    // 实际布局阶段（Step 2）：重新排列 ViewHolder，可能从 Scrap 中复用
    dispatchLayoutStep2(); 
    // 布局后阶段（Step 3）：处理动画，清理 Scrap 缓存
    dispatchLayoutStep3(); 
}
```
- **关键逻辑**：  
  动画预布局（`dispatchLayoutStep1`）时，会将旧状态的 `ViewHolder` 分离到 `mAttachedScrap`（如删除动画中的旧 View）；动画执行后，这些 `ViewHolder` 会被回收或重新附着。  


### **mAttachedScrap 与 mChangedScrap 的区别**
| 缓存类型          | 触发场景                                                                 | 特点                                                                 |
|-------------------|--------------------------------------------------------------------------|----------------------------------------------------------------------|
| `mAttachedScrap`  | 普通布局刷新（如滚动、首次布局）、动画暂存（未标记更新的 ViewHolder）       | - 存储未被标记为“需要更新”的 ViewHolder<br>- 可直接复用（无需重新绑定） |
| `mChangedScrap`   | 数据更新（如 `notifyItemChanged`）、预布局中需要更新的 ViewHolder          | - 存储标记为“需要更新”的 ViewHolder<br>- 需重新绑定数据后复用          |


### **总结：何时添加？**
- **添加到 `mAttachedScrap`**：  
  布局过程中临时分离的、未被标记为需要更新的 `ViewHolder`（如滚动时调整位置、动画暂存旧状态）。  
- **添加到 `mChangedScrap`**：  
  数据更新（如 `notifyItemChanged`）或预布局中需要重新绑定数据的 `ViewHolder`（标记为 `FLAG_UPDATE`）。  


### **附：关键源码调用链**
```plaintext
RecyclerView.requestLayout() → 
LayoutManager.onLayoutChildren() → 
detachAndScrapAttachedViews(recycler) → 
Recycler.scrapView() → 
根据 ViewHolder 状态添加到 mAttachedScrap 或 mChangedScrap
```

通过这种机制，RecyclerView 确保了布局过程中可见 `ViewHolder` 的快速复用，避免了重复创建和绑定，从而提升了滚动流畅性。