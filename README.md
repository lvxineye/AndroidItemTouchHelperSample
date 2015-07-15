# 实现RecyclerView的拖拽
> 使用Android Support Library中的ItemTouchHelper类实现RecyclerView的拖拽效果。<br/>
> 使用RecyclerView实现基本的拖拽和滑动消失效果是不需要引入三方library的

## ItemTouchHelper
ItemTouchHelper是一个非常强大的工具类用于处理当你添加drag&drop 和 swipe-to-dismiss特性到RecyclerView时所关心的那些问题。它是RecyclerView.ItemDecoration的子类,意味着可以向已经存在的LayoutManager 和 Adapter(!)中添加任何东西，并且也可以处理已经存在的item animations、type-restricted、drop settling animations甚至更多。

## 配置
首先我们需要对RecyclerView进行配置，如果没有准备好，请先更新你的build.gradle，添加RecyclerView依赖。

```
compile 'com.android.support:recyclerview-v7:22.2.0'
``` 

在任意RecyclerView.Adapter 和 LayoutManager都可以使用ItemTouchHelper

## 使用ItemTouchHelper 和 ItemTouchHelper.Callback
为了使用ItemTouchHelper，你必须得先创建ItemTouchHelper.Callback。这是一个接口用于监听“move” 和 “swipe”的状态。也可用于你要控制所选的view的状态和重写默认动画。如果你只是想使用基本实现你可以使用系统提供的一个帮助类SimpleCallback。

- 实现基本的drag & drop 和 swipe-to-dismiss必须实现以下主要的回调函数:

```
> getMovementFlags(RecyclerView, ViewHolder)
> onMove(RecyclerView, ViewHolder, ViewHolder)
> onSwiped(ViewHolder, int)
```

- 使用两个帮助函数:

```
> isLongPressDragEnabled()
> isItemViewSwipeEnabled()
```

- 详细介绍：

```
@Override
public int getMovementFlags(RecyclerView recyclerView, 
    RecyclerView.ViewHolder viewHolder) {
    int dragFlags = ItemTouchHelper.UP | ItemTouchHelper.DOWN;
    int swipeFlags = ItemTouchHelper.START | ItemTouchHelper.END;
    return makeMovementFlags(dragFlags, swipeFlags);
}
```
**ItemTouchHelper**允许非常简单的判断屏幕事件的走向。必须重写**getMovementFlags()**来说明支持哪个方向的拖拽和滑动。使用帮助类**ItemTouchHelper.makeMovementFlags(int, int)**来管理returned标志。这样就可以在同一个方向进行拖拽和滑动了。


```
@Override
public boolean isLongPressDragEnabled() {
    return true;
}
```
**ItemTouchHelper**可以只支持*drag*而不支持*swipe(or vice versa)*，所以你必须明确指出你希望支持的类型。为了支持长按 RecyclerView item 对其进行拖动，**isLongPressDragEnabled()**函数实现应该返回true. 此外在开始拖拽时**ItemTouchHelper.startDrag(RecyclerView.ViewHolder)**将会被调用。

```
public boolean isItemViewSwipeEnabled() {
    return true;
}
```
如果在*viwe*中可以随意滑动，**isItemViewSwipeEnabled()**函数返回true. 此外手动拖动时**ItemTouchHelper.startSwipe(RecyclerView.ViewHolder)**会被调用。

**onMove()**和**onSwiped()**负责主要数据的更新。故首先我们需要创建一个允许传递事件回调的接口。

```
public interface ItemTouchHelperAdapter {
 
    void onItemMove(int fromPosition, int toPosition);
 
    void onItemDismiss(int position);
}
```
最简单的方式就是下面这样，让我们的RecyclerListAdapter实现ItemTouchHelperAdapter。

```
public class RecyclerListAdapter extends 
        RecyclerView.Adapter<ItemViewHolder> 
        implements ItemTouchHelperAdapter {
	//....
}
```

```
@Override
public void onItemDismiss(int position) {
    mItems.remove(position);
    notifyItemRemoved(position);
}

@Override
public void onItemMove(int from, int to) {
    Collections.swap(mItems, from, to);
    notifyItemMoved(from, to);
}
```

**notifyItemRemoved()** 和 **notifyItemMoved()**的调用是非常重要，因此Adapter是可以感受到这些变化的,同样重要的是要注意当我们改变item的Position的时候view的index也时刻在改变，并不是在“drop”事件结束后再改变。

现在我们回过头来创建我们自己的**SimpleItemTouchHelperCallback**并且必须override **onMove()** 和 **onSwiped()**。首先添加一个构造函数和Adapter变量。

```
private final ItemTouchHelperAdapter mAdapter;
 
public SimpleItemTouchHelperCallback(
        ItemTouchHelperAdapter adapter) {
    mAdapter = adapter;
}
```
之后 override the remaining events and notify the adapter:

```
@Override
public boolean onMove(RecyclerView recyclerView, 
        RecyclerView.ViewHolder viewHolder, 
        RecyclerView.ViewHolder target) {
 	//...       
}
```
```
mAdapter.onItemMove(viewHolder.getAdapterPosition(), 
            target.getAdapterPosition());
    return true;
}
```

```
@Override
public void onSwiped(RecyclerView.ViewHolder viewHolder, 
        int direction) {
    mAdapter.onItemDismiss(viewHolder.getAdapterPosition());
}
```

写完之后回调类应该像下面这样：

```
public class SimpleItemTouchHelperCallback extends ItemTouchHelper.Callback {
 
    private final ItemTouchHelperAdapter mAdapter;
 
    public SimpleItemTouchHelperCallback(ItemTouchHelperAdapter adapter) {
        mAdapter = adapter;
    }
 
    @Override
    public boolean isLongPressDragEnabled() {
        return true;
    }
 
    @Override
    public boolean isItemViewSwipeEnabled() {
        return true;
    }
 
    @Override
    public int getMovementFlags(RecyclerView recyclerView, ViewHolder viewHolder) {
        int dragFlags = ItemTouchHelper.UP | ItemTouchHelper.DOWN;
        int swipeFlags = ItemTouchHelper.START | ItemTouchHelper.END;
        return makeMovementFlags(dragFlags, swipeFlags);
    }
 
    @Override
    public boolean onMove(RecyclerView recyclerView, ViewHolder viewHolder, 
            ViewHolder target) {
        mAdapter.onItemMove(viewHolder.getAdapterPosition(), target.getAdapterPosition());
        return true;
    }
 
    @Override
    public void onSwiped(ViewHolder viewHolder, int direction) {
        mAdapter.onItemDismiss(viewHolder.getAdapterPosition());
    }
 
}
```
随着我们Callback的完成，再创建ItemTouchHelper并且调用**attachToRecyclerView(RecyclerView)**

```
ItemTouchHelper.Callback callback = 
    new SimpleItemTouchHelperCallback(adapter);
ItemTouchHelper touchHelper = new ItemTouchHelper(callback);
touchHelper.attachToRecyclerView(recyclerView);
```

## 参考
- [拖拽RecyclerView](http://www.devtf.cn/?p=795)
- [Android-ItemTouchHelper-Demo](https://github.com/iPaulPro/Android-ItemTouchHelper-Demo)
