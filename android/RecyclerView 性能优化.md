# RecyclerView 性能优化

## 数据处理和视图加载分离

我们知道，从远端拉取数据肯定是要放在异步的，在我们拉取下来数据之后可能就匆匆把数据丢给了 VH 处理，其实，数据的处理逻辑我们也应该放在异步处理，这样 Adapter 在 notify change 后，ViewHolder 就可以简单无压力地做数据与视图的绑定逻辑，比如：

```java
mTextView.setText(Html.fromHtml(data).toString());
```

这里的 `Html.fromHtml(data)` 方法可能就是比较耗时的，存在多个 `TextView` 的话耗时会更为严重，这样便会引发掉帧、卡顿，而如果把这一步与网络异步线程放在一起，站在用户角度，最多就是网络刷新时间稍长一点。


## 数据优化

分页拉取远端数据，对拉取下来的远端数据进行缓存，提升二次加载速度；对于新增或者删除数据通过 `DiffUtil` 来进行局部刷新数据，而不是一味地全局刷新数据。


## 布局优化

### 减少过渡绘制

减少布局层级，例如使用 `ConstraintLayout`或者用一个自定义 View 来代替 ItemView。


### 减少 xml 文件 inflate 时间

这里的 xml 文件不仅包括 layout 的 xml，还包括 drawable 的 xml，xml 文件 inflate 出 ItemView 是通过耗时的 IO 操作，尤其当 Item 的复用几率很低的情况下，随着 Type 的增多，这种 inflate 带来的损耗是相当大的，此时我们可以用代码去生成布局，即 `new View()` 的方式，只要搞清楚 xml 中每个节点的属性对应的 API 即可。


### 减少 View 对象的创建

一个稍微复杂的 Item 会包含大量的 View，而大量的 View 的创建也会消耗大量时间，所以要尽可能简化 ItemView；设计 ItemType 时，对多 ViewType 能够共用的部分尽量设计成自定义 View，减少 View 的构造和嵌套。


## 其他

其他并不代表不重要，而是我不能把他们进行分类哈，其中可能某些操作会对你的 RecyclerView 有很大的优化。

* 升级 `RecycleView` 版本到 25.1.0 及以上使用 Prefetch 功能，可参考 [RecyclerView 数据预取](https://juejin.im/entry/58a3f4f62f301e0069908d8f)。

* 如果 Item 高度是固定的话，可以使用 `RecyclerView.setHasFixedSize(true);` 来避免 `requestLayout` 浪费资源；

* 设置 `RecyclerView.addOnScrollListener(listener);` 来对滑动过程中停止加载的操作。

* 如果不要求动画，可以通过 `((SimpleItemAnimator) rv.getItemAnimator()).setSupportsChangeAnimations(false);` 把默认动画关闭来提神效率。

* 对 `TextView` 使用 `String.toUpperCase` 来替代 `android:textAllCaps="true"`。

* 对 `TextView` 使用 `StaticLayout` 或者 `DynamicLayout` 的自定义 `View` 来代替它。

* 通过重写 `RecyclerView.onViewRecycled(holder)` 来回收资源。

* 通过 `RecycleView.setItemViewCacheSize(size);` 来加大 `RecyclerView` 的缓存，用空间换时间来提高滚动的流畅性。

* 如果多个 `RecycledView` 的 `Adapter` 是一样的，比如嵌套的 `RecyclerView` 中存在一样的 `Adapter`，可以通过设置 `RecyclerView.setRecycledViewPool(pool);` 来共用一个 `RecycledViewPool`。

* 对 `ItemView` 设置监听器，不要对每个 Item 都调用 `addXxListener`，应该大家公用一个 `XxListener`，根据 `ID` 来进行不同的操作，优化了对象的频繁创建带来的资源消耗。

* 通过 getExtraLayoutSpace 来增加 RecyclerView 预留的额外空间，如下所示：
  ```
  new LinearLayoutManager(this) {
      @Override
      protected int getExtraLayoutSpace(RecyclerView.State state) {
          return size;
      }
  };
  ```


## 结语

我正在打造一个帮助 Android 开发者们拿到更好 offer 的面试库————[安卓 offer 收割基](https://github.com/Blankj/AndroidOfferKiller)，欢迎 star，觉得不错的可以持续关注，有兴趣的可以一起加入进来和我一同打造。