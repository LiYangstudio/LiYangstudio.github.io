---
layout: post
title: 【RecyclerView】为RecyclerView设置监听的设计过程
desc: 我的博客系统介绍
keywords: 'blog,Machine Learning,AI'
date: 2017-1-18T00:00:00.000Z
categories:
- blog
tags:
- blog
icon: fa-blog
---
# 概述

RecyclerView由于天生没有像ListView那样具有setOnItemClickListener或者setOnItemLongClickListener能够很简单对Item进行监听的方法。

<!-- more -->


所以网上出现了多种实现方式：

* 1.利用addOnItemTouchListener的方式，将事件传递给GestureDetectorCompat，进行手势处理。

* 2.直接在自己的adapter里面对需要的View进行监听

* 3.另外一些比较复杂和奇葩的方法- -

附上一个外链：[三种方式实现RecyclerView监听](http://blog.devwiki.net/index.php/2016/07/24/three-ways-click-recyclerview-item.html)



我个人对上面三种方式的点评：

* 1.个人认为通过GestureDetectorCompat实现单击和长按事件有点小题大作，杀鸡用牛刀的感觉

* 2.手动对需要的View进行监听，这样就没有封装性了。 而且每次都要自己写一次，很麻烦；并且代码逻辑放在了Adapter里面，不是通过接口的方式，结构上就不合理。

* 3.其他复杂或者奇葩的方法，感觉是比较类似于奇淫巧技，适合研究，但不适合使用的。



# 探索过程

* 1.项目当初是通过短暂思考并且设计了一个简单实现的ClickableAdapter，就投放到项目中去使用。 这里可以称之为第一版。

* 2.业务逻辑越来越复杂，导致了原本ClickableAdapter出现了一个BUG，是由设计失误造成的。 经过不断的尝试，终于衍生出了第二版。



## 第一版设计思路：

* 1.在onBindViewHolder中，将position通过View的setTag方法，与View本身绑定起来。

* 2.然后对ItemView进行View.setOnClickListener

* 3.View.setOnClickListener回调中有View本身传回来，然后拿出View中存储的TAG，

* 4.再通过封装接口将position回传给外面



## 第二版设计思路：

* 1.在onCreateViewHolder中，将第一个参数ViewGroup parent保存为成员变量。（通过查询RecyclerView源代码发现，这个ViewGroup就是RecyclerView本身）

* 2.在onBindViewHolder中，对ItemView进行View.setOnClickListener

* 3.View.setOnClickListener回调中有View本身传回来，然后通过recyclerView.getChildAdapterPosition(itemView)拿到我们的position

* 4.再通过封装的接口将position回传给外面

* 5.额外考虑1：RecyclerView在设计的时候，其Adapter本身是不持有RecyclerView本身的引用的，所以这个封装破坏了原本RecyclerView的设计思路

* 6.额外考虑2：由于在成员变量持有RecyclerView，存在潜在的内存泄露问题。为了保险起见，使用弱引用。



# 源代码分析

第一版的设计是存在不合理性的：

* 1.TAG冲突问题

* 2.在某特殊场景使用下，会导致position位置**不准确**问题（后续会详细解释）

需要从代码中来查看



## ClickableAdapter（第一版）

### 源代码

```

/**
 * 封装了长按和点击的RecyclerView的Adapter(使用TAG实现)
 *
 * @author Hans
 * @date 16.7.11
 * <p/>
 * Warning By Hans 2016.7.22 :
 * 目前发现了一个BUG... 通过长按删除并通过[RecyclerView.Adapter.notifyItemRemoved局部刷新的方法刷新]之后,
 * 我们会根据position在List中对应的数据项目,但是由于View已经被移除了
 * 而他们的position是根据他们的一开始列表的位置进行设置的,所以会造成原本的数据移除得并不正确....
 * </p>
 * 解决思路1:
 * 找一个办法对删除的position之后的holder中的itemView重新设置TAG(重设位置)
 * 解决进度:未解决
 * </p>
 * 解决思路2:
 * 不使用RecyclerView.Adapter.notifyItemRemoved局部刷新,使用RecyclerView.Adapter.notifyDataSetChange全部重绘
 * 解决进度:可解决(但没有充分利用高效性)
 * </p>
 * 解决思路3:(update by Hans 2016.7.25)
 * 通过其他方法解决,目前已想出通过RecyclerView本身获取item的ViewHolder位置定位 position. 不使用TAG方式{@link ClickableRecyclerAdapter}
 * 解决进度:可解决(充分利用 RecyclerView局部刷新的高效性)
 */
public abstract class ClickableAdapter<V extends RecyclerView.ViewHolder> extends RecyclerView.Adapter<V> implements View.OnClickListener, View.OnLongClickListener {
    protected OnItemClickListener mOnItemClickListener;
    protected OnItemLongClickListener mOnItemLongClickListener;

    private static final int TAG_KEY = 0x12345678;

    @Override
    public void onBindViewHolder(V holder, int position) {
        if (mOnItemClickListener != null)
            holder.itemView.setOnClickListener(this);
        if (mOnItemLongClickListener != null)
            holder.itemView.setOnLongClickListener(this);
        holder.itemView.setTag(TAG_KEY, position);
        onBindVH(holder, position);
    }

    abstract public void onBindVH(V holder, int position);


    public void setOnItemClickListener(OnItemClickListener listener) {
        mOnItemClickListener = listener;
    }

    public void setOnLongClickListener(OnItemLongClickListener listener) {
        mOnItemLongClickListener = listener;
    }


    @Override
    public void onClick(View v) {
        int position = (int) v.getTag(TAG_KEY);
        mOnItemClickListener.onItemClick(v, position);
    }

    @Override
    public boolean onLongClick(View v) {
        int position = (int) v.getTag(TAG_KEY);
        return mOnItemLongClickListener.onItemLongClick(v, position);

    }

    public interface OnItemClickListener {
        void onItemClick(View view, int position);
    }

    public interface OnItemLongClickListener {
        boolean onItemLongClick(View view, int position);
    }
}
```

### 实现的可行性分析

实现是通过setTag方式实现的，那么setTag的内部机理实际上也不难。

点两下源代码就能找到内部实现的数据结构：

```

/**
 * Map used to store views' tags.
 */
private SparseArray<Object> mKeyedTags;
```

简单来说就是一个高效的HashMap<Interge,Object> （这个不在本文讨论之列）



所以从上面来看，这样子实现是可行的。**但由此引出了第一个问题**



#### 问题一（TAG重复问题）

也就是说这个TAG本身，是可能存在冲突的（即被覆盖）。 对于团队合作，如果团队成员不熟悉其实现原理，就有一定的可能会设置相同的TAG。（虽然概率非常小，但是还是会存在）



#### 问题二（删除Item）

在这个使用场景中：删除Item，并调用adapter的notifyItemRemoved进行局部刷新；会导致刷新之后的item的position错位问题



在理解这个BUG之前，要明确几点：

* 1.itemView的事件监听实在onBindViewHolder中完成的

* 2.notifyItemRemoved是对RecylerView布局位置操作，并不会重新绘制；也就是不会重新调用onBindViewHolder

* 3.notifyDataSetChanged是全部重新绘制的，也就是说会重新调用onBindViewHolder



在明确上面这几点的前提下，那么就很能理解上面使用场景出现的BUG了。

简单举个例子：

* 1.原本数据列表是0 1 2 3 4并设置了单击事件监听

* 2.单击数据2（在用户角度是三号位）的item回传的position是几？（答案是：2，因为TAG也是2）

* 3.现在删除了数据2的item（在用户角度来看是三号位），并调用notifyItemRemoved(2).

* 4.现在数据列表是0 1 3 4

* 5.那么现在需要思考点击3号位（在用户角度来看依旧是三号位）回传的position是几？

* 6.预期传回来的值是2（用户三号位是程序员的二号位），但是传回来的却是3

* 7.这就是由于position是在onBindViewHolder时期通过TAG方式与View本身绑定的，在调用notifyItemRemoved之后，它们的TAG还是原来数据中的位置



解决方式：

* 1.不使用RecyclerView局部刷新特性，直接调用notifyDataSetChanged；要求列表重新绘制，本质是为了调用onBindViewHolder进行position位置重新分配

* 2.重新思考通过TAG方式实现的ClickableAdapter的局限性



### 第二版的准备

* 1.对于强迫症来说，不能用局部刷新特性，那跟咸鱼有什么区别（ListView）

* 2.第二版不能够通过TAG来传递位置已成定局



## ClickableRecyclerAdapter

设计思路上面已经说了，所以直接贴源代码

### 源代码

```

/**
 * Created by Hans on 16/7/25.
 * 封装了长按和点击的RecyclerView的Adapter(通过RecyclerView拿item的ViewHolder从而定位position)
 * <p/>
 * 主要是为了解决{@link ClickableAdapter}在7.22发现的BUG,即在该使用场景下:[删除Item并调用局部刷新];从而导致了监听位置position不准确的问题
 * </p>
 * 设计思路:
 * 1.通过源代码发现onCreateViewHolder第一个参数parent回传的是RecyclerView.this本身
 * 2.该封装的ClickableRecyclerAdapter持有RecyclerView引用
 * 3.并在监听事件的时候,通过recyclerView.getChildAdapterPosition(v); 拿到位置
 * <p/>
 * 可能存在的问题:
 * 1.RecyclerView.Adapter本身设计是不持有RecyclerView本身的.(此封装,破坏了RecyclerView原有的设计理念)
 * <p/>
 * 2.可能存在内存泄露的问题. 因为View都是持有Context本身的引用的,所以Adapter持有RecyclerView,就持有了Context.
 * 为了解决这个问题,通过弱引用的方式能够解决.
 */
public abstract class ClickableRecyclerAdapter<V extends RecyclerView.ViewHolder> extends RecyclerView.Adapter<V> implements View.OnClickListener, View.OnLongClickListener {
    protected OnItemClickListener mOnItemClickListener;
    protected OnItemLongClickListener mOnItemLongClickListener;
    private WeakReference<RecyclerView> mRecyclerView;

    @Override
    public void onBindViewHolder(V holder, int position) {
        if (mOnItemClickListener != null)
            holder.itemView.setOnClickListener(this);
        if (mOnItemLongClickListener != null)
            holder.itemView.setOnLongClickListener(this);
        onBindVH(holder, position);
    }

    abstract public void onBindVH(V holder, int position);

    @Override
    public V onCreateViewHolder(ViewGroup parent, int viewType) {
        mRecyclerView = new WeakReference<RecyclerView>((RecyclerView) parent);
        return onCreateVH(parent, viewType);
    }

    public abstract V onCreateVH(ViewGroup parent, int viewType);

    public void setOnItemClickListener(OnItemClickListener listener) {
        mOnItemClickListener = listener;
    }

    public void setOnLongClickListener(OnItemLongClickListener listener) {
        mOnItemLongClickListener = listener;
    }


    @Override
    public void onClick(View v) {
        RecyclerView recyclerView = mRecyclerView.get();
        if (recyclerView != null) {
            int position = recyclerView.getChildAdapterPosition(v);
            mOnItemClickListener.onItemClick(v, position);
        }
    }

    @Override
    public boolean onLongClick(View v) {
        RecyclerView recyclerView = mRecyclerView.get();
        if (recyclerView != null) {
            int position = recyclerView.getChildAdapterPosition(v);
            return mOnItemLongClickListener.onItemLongClick(v, position);
        }
        return true;
    }

    public interface OnItemClickListener {
        void onItemClick(View view, int position);
    }

    public interface OnItemLongClickListener {
        boolean onItemLongClick(View view, int position);
    }
}
```



### 实现可行性分析

下面对实现进行分析

#### ViewGroup一定是RecyclerView本身？

通过查看RecyclerView的源代码：

```

holder = mAdapter.createViewHolder(RecyclerView.this, type);
```

可以确定onCreateViewHolder传过来的第一个参数ViewGroup，就是RecyclerView本身。（该方法在RecyclerView的代码中仅有此处被调用）



#### RecyclerView是通过itemView拿到ViewHolder位置从而定位position的？

在上面代码的实现中：

```

int position = recyclerView.getChildAdapterPosition(v);
```

看起来不像是通过ViewHolder定位。



可以点进去源代码发现：

```

/**
 * Return the adapter position that the given child view corresponds to.
 *
 * @param child Child View to query
 * @return Adapter position corresponding to the given view or {@link #NO_POSITION}
 */
public int getChildAdapterPosition(View child) {
    final ViewHolder holder = getChildViewHolderInt(child);
    return holder != null ? holder.getAdapterPosition() : NO_POSITION;
}

static ViewHolder getChildViewHolderInt(View child) {
    if (child == null) {
        return null;
    }
    return ((LayoutParams) child.getLayoutParams()).mViewHolder;
}
```

可以发现其内部调用了 **包级静态方法getChildViewHolder**拿到ViewHolder在拿到position的



# 心得体会

在我负责的模块中，从ClickableAdapter替换到ClickableRecyclerAdaper应该除了重写额外的一个方法，并且替换监听类的包就可以重新直接跑起来了。



然而并不是我想的那样子。



## 前后源代码

### 替换前

```

    @Override

    public void onBindVH(RecyclerView.ViewHolder holder, int position) {

        int type = getItemViewType(position);

        if (type == EMPTY_URL_TYPE) {

            if (mPhotoList.size() < mMaxNum) {

                holder.itemView.setVisibility(View.VISIBLE);

                ((ImageView) holder.itemView).setImageResource(mLastItemResourceId);

            } else {//如果超过MaxNum就不显示了

                holder.itemView.setVisibility(View.GONE);

            }

        } else {

            String path = mPhotoList.get(position);

            CommonImageLoader.displayImage(ImageDownloader.Scheme.FILE.wrap(path),

                    (ImageView) holder.itemView,

                    CommonImageLoader.DOUBLE_CACHE_OPTIONS);

        }

    }



   @Override

    public int getItemViewType(int position) {

        if (position == mPhotoList.size()) {//如果是当前最后一位

            return EMPTY_URL_TYPE;

        }

        return super.getItemViewType(position);

    }

```



### 替换后

```

@Override
public void onBindVH(RecyclerView.ViewHolder holder, int position) {
    int type = getItemViewType(position);
    if (type == EMPTY_URL_TYPE) {
        ((ImageView) holder.itemView).setImageResource(mLastItemResourceId);
    } else if (type == PIC_URL_TYPE) {
        String path = mPhotoList.get(position);
        CommonImageLoader.displayImage(ImageDownloader.Scheme.FILE.wrap(path),
                (ImageView) holder.itemView,
                CommonImageLoader.DOUBLE_CACHE_OPTIONS);
    } else {
        // do nothing
        //不是上述两种类型的,不做处理
    }
}

@Override
public int getItemViewType(int position) {
    if (position == mPhotoList.size() && mPhotoList.size() < mMaxNum) {//如果是当前最后一位 且图片还没有选满(说明这个位置的类型是显示+号的类型)
        return EMPTY_URL_TYPE;
    }
    if (position < mMaxNum) {//如果当前小于最大选择图片数量的position(说明 这个位置的类型是 显示图片的类型)
        return PIC_URL_TYPE;
    }
    return super.getItemViewType(position);//返回默认0
}
```

## 分析

写的不好的原因有两个：

* 1.代码写的不好的地方：ItemViewType本身就是用来控制itemView显示与否，显示什么布局而生；我这里进行了double处理，就是即用了itemViewType，又通过VISIBLE来设置显示与否。 这样的混用会导致逻辑混乱！！

* 2.上文提到notifyItemRemoved是不会调用onBindVH的。所以当我自己通过根据具体场景设置了GONE之后，而后又因为删除item原本被GONE的View要显示出来，但是onBindVH是不会再被调用了，也就是你的View一旦在onBindVH方法中被GONE了之后，就再也看不到了。（逻辑不会再次调用了）

## 启示

* 1.明确Adapter的getViewItemType的作用（而不是通过在onBindVH中去处理显示逻辑）

    * 1.显示不同的布局样式

    * 2.是否显示某个布局样式

* 2.布局显示与否的逻辑放在getItemViewType中。这样就算是notifyItemRemoved，getItemViewType是要被调用的。





**- Hans 2016.7.25**
