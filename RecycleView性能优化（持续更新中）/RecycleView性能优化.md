## 卡顿原因

### RecyclerView: notifyDataSetChanged

数据需要全局刷新时，可以使用notifyDataSetChanged；对于增加或减少数据，可以使用如下方法实现局部刷新。

```java
void onNewDataArrived(List<News> news) {
    List<News> oldNews = myAdapter.getItems();
    DiffResult result = DiffUtil.calculateDiff(new MyCallback(oldNews, news));
    myAdapter.setNews(news);
    result.dispatchUpdatesTo(myAdapter);
}
```

### 嵌套RecycleView

常见于竖直滚动的RecycleView嵌套一个横向滚动的RecycleView。对于单个RecycleView而言，都拥有独立的itemView对象池，对于嵌套的情况，可以设置共享对象池，如下：

```java
class OuterAdapter extends RecyclerView.Adapter<OuterAdapter.ViewHolder> {
    RecyclerView.RecycledViewPool mSharedPool = new RecyclerView.RecycledViewPool();

    ...

    @Override
    public void onCreateViewHolder(ViewGroup parent, int viewType) {
        // inflate inner item, find innerRecyclerView by ID…
        LinearLayoutManager innerLLM = new LinearLayoutManager(parent.getContext(),
                LinearLayoutManager.HORIZONTAL);
        innerRv.setLayoutManager(innerLLM);
        innerRv.setRecycledViewPool(mSharedPool);
        return new OuterAdapter.ViewHolder(innerRv);

    }
    
```

更近一步可以调用 `setInitialPrefetchItemCount(int)` 来优化嵌套时预加载性能，例如横向RecycleView上有3.5个item需要显示，可以调用`LinearLayoutManager.setInitialPrefetchItemCount(4)`，默认的数值是2。

### RecyclerView: Too much inflation / Create taking too long

减少view type的数量，如果只是样式上相差不大，可以再bind方法里进行改变，因为创建不同的view type需要额外的inflate调用导致卡顿。

### RecyclerView: Bind taking too long

[onBindViewHolder(VH, int)](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.Adapter.html#onBindViewHolder(VH, int))的实现应该非常简单，通过读取POJO类的数据来设置ViewHolder，不应该有其他额外的逻辑实现。

### RecyclerView or ListView: layout / draw taking too long

### Layout性能

通过Systemtrace工具可以检测Layout的性能，如果耗时太长或者调用次数过多，需要考察一下是否过度使用RelativeLayout或者嵌套多层LinearLayout，每层Layout都会导致其child多次的measure/layout，其时间复杂度是O(n²)。有如下方法可以优化

- 重新组织层级
- 自定义控件重写layout方法
- 使用 [ConstraintLayout](https://developer.android.com/training/constraint-layout/index.html)

### Bitmap传递

Android以OpenGL Texture的形式来展示bitmap，当bitmap第一次展示时，它会以width * height大小的Texture形式传递到GPU上，所以要保证bitmap的大小不会大于其展示大小，要知道上传过程是阻塞主线程的。一般传递一张1920 * 1080的Texture不会超过10ms。

### 对象分配和垃圾回收

虽然Android 5.0上使用ART来减少GC停顿时间，但仍然会造成卡顿。尽量避免在循环内创建对象导致GC。要知道，创建对象需要分配内存，而这个时机会检查内存是否足够来决定需不需要进行GC。

## RecycleView的优化技巧

### TextView

避免使用textAllCaps，直接使用String.toUpperCase

### 单个监听器

对于每个ViewHolder都要设置监听器怎么办？

全局new一个Listener，通过view.getId与view.getTag取到对应View的id和数据，避免对每个新创建的Viewholder都new出一个监听器。

### 值相同避免再次刷新

例如TextView的text相同，就不需要再调用setText方法，对比的损耗往往小于绘制。

### 避免半透明绘制

尽量不要对ViewHolder使用有透明度改变的绘制。

### 自定义View

使用StaticLayout和DynamicLayout代替TextView，使用自定义View代替LayoutInflater.inflate(xml)文件，或者使用ConstraintLayout降低层级深度。

### 缓存池

缓存自定义View，下载好网络上的POJO类数据避免重复下载。

### 预加载

```java
// 通过复写指定预加载的像素值。
LinearLayoutManager.getExtraLayoutSpace();
```

和

```java
// 设置预加载itemview数目。
RecycleView.setItemViewCacheSize(size);
```

### 参考资料

Android 7.0源码

[Android 官方文档Performance -- Slow Rendering](https://developer.android.com/topic/performance/vitals/render.html)

[Recycler View – pre cache views](https://androiddevx.wordpress.com/2014/12/05/recycler-view-pre-cache-views/)

[RecyclerView item optimizations Or how StaticLayout can help you](https://medium.com/@programmerr47/recyclerview-item-optimizations-cae1aed0c321)

[RecyclerView Prefetch](https://medium.com/google-developers/recyclerview-prefetch-c2f269075710)

[优化ListView有哪些方法](https://www.zhihu.com/question/19703384)

