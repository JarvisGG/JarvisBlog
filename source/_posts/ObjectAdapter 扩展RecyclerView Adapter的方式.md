---
title: ObjectAdapter 扩展RecyclerView Adapter的方式
date: 2017-11-01 10:03:56
tags: Android 开源控件
comments: true
---

## 前言
起因，随着业务的开展，我们的Feed列表有时会增加好多的卡片类型，而过多的卡片就会引发过多的 ViewHolder，ViewType 造成Adapter路基冗杂。那么本篇博客就是基于这个问题做的Adapter 扩展。
<!--more-->

## 扩展对比
### 扩展前
Activity:
``` Java
recyclerView.setAdapter(new OldAdapter(this));
recyclerView.setLayoutManager(new LinearLayoutManager(this));
```
Adapter:
``` Java
public class OldAdapter extends RecyclerView.Adapter {
    
    private Context mContext;
    private LayoutInflater mInflater;
    private List<IndexTypeEnum> mData;
    
    public OldAdapter(Context context) {
        this.mContext = context;
        this.mInflater = LayoutInflater.from(mContext);
        this.mData = new ArrayList<>();
    }

    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View view;
        if (viewType == 1) {
            view = mInflater.inflate(R.layout.item_frist, parent, false);
        } else if (viewType == 2) {
            view = mInflater.inflate(R.layout.item_second, parent, false);
        } else {
            view = mInflater.inflate(R.layout.item_thrid, parent, false);
        }
        return new InnerViewHolder(view);
    }

    @Override
    public void onBindViewHolder(RecyclerView.ViewHolder holder, int position) {
        // ...do
    }

    @Override
    public int getItemCount() {
        return mData.size();
    }

    @Override
    public int getItemViewType(int position) {
        if (position == 0) {
            return 1;
        } else if (position == 1) {
            return 2;
        } else {
            return 3;
        }
    }
    
    public void setData(List<IndexTypeEnum> data) {
        this.mData = data;
        notifyDataSetChanged();
    }

    public void add(IndexTypeEnum indexTypeEnum, int pos) {
        mData.add(pos, indexTypeEnum);
        notifyItemChanged(pos);
    }
    
    public void remove(IndexTypeEnum indexTypeEnum, int pos) {
        mData.remove(pos);
        notifyItemChanged(pos);
    }
    
    public void move(int fromPosition, int toPosition) {
        IndexTypeEnum indexTypeEnum = mData.get(fromPosition);
        mData.set(fromPosition, mData.get(toPosition));
        mData.set(toPosition, indexTypeEnum);
        notifyItemMoved(fromPosition, toPosition);
    }
    
    public static class InnerViewHolder extends RecyclerView.ViewHolder {

        public InnerViewHolder(View itemView) {
            super(itemView);
        }
    }
}
```
添加：
``` Java
adapter.add(1, FRIST);
```
删除：
``` Java
adapter.remove(1);
```
移动：
``` Java
adapter.move(1,2);
```
### 扩展后
Activity:
``` Java
presenterSelector = new ZhihuPresenterSelector(this);
objectAdapter = new ArrayObjectAdapter(presenterSelector);
recyclerView.setObjectAdapter(objectAdapter);
recyclerView.setLayoutManager(new LinearLayoutManager(this));

for (IndexTypeEnum indexTypeEnum : data) {
    objectAdapter.add(indexTypeEnum);
}
```
PresenterSelector:
``` Java
public class ZhihuPresenterSelector extends PresenterSelector {

    private final ArrayList<Presenter> mPresenters = new ArrayList<>();
    private FristPresenter fristPresenter;
    private SecondPresenter secondPresenter;
    private ThridPresenter thridPresenter;

    public ZhihuPresenterSelector(Context context) {
        fristPresenter = new FristPresenter(context);
        secondPresenter = new SecondPresenter(context);
        thridPresenter = new ThridPresenter(context);
        containerPresenter = new ContainerPresenter(context);

        mPresenters.add(fristPresenter);
        mPresenters.add(secondPresenter);
        mPresenters.add(thridPresenter);
    }

    @Override
    public Presenter getPresenter(Object item) {
        IndexTypeEnum indexRow = (IndexTypeEnum) item;
        switch (indexRow) {
            case FRIST:
                return fristPresenter;
            case SECOND:
                return secondPresenter;
            case THRIED:
                return thridPresenter;
            default:
                return fristPresenter;
        }
    }

    @Override
    public Presenter[] getPresenters() {
        return mPresenters.toArray(new Presenter[mPresenters.size()]);
    }
}
```
Presenter:
``` Java
public class FristPresenter extends Presenter {

    private Context mContext;
    private LayoutInflater mInflater;

    public FristPresenter(Context context) {
        this.mContext = context;
        this.mInflater = ((Activity) context).getLayoutInflater();
    }

    @Override
    public ViewHolder onCreateViewHolder(ViewGroup parent) {
        View view = mInflater.inflate(R.layout.item_frist, parent, false);
        return new InnerViewHolder(view);
    }

    @Override
    public void onBindViewHolder(ViewHolder viewHolder, Object item, int position) {

    }

    @Override
    public void onUnBindViewHolder(ViewHolder viewHolder) {

    }

    protected class InnerViewHolder extends ViewHolder {

        public InnerViewHolder(View view) {
            super(view);
        }
    }
}
```
添加：
``` Java
objectAdapter.add(1, indexTypeEnum)
```
删除：
``` Java
objectAdapter.remove(indexTypeEnum1);
objectAdapter.remove(1);
```
移动：
``` Java
objectAdapter.move(1,2);
```
### 小节
这里我们可以看到，优化前Adapter是很冗杂的，但优化后，我们可以按职责详细划分这里的业务逻辑.</br>
1.我们这里通过 PresenterSelector 来总结当前所有的卡片种类（Presenter).</br>
2.将我们不同的卡片业务逻辑划分到对应的 Presenter.</br>
3.关于基本操作全部封装在 ObjectAdapter.</br>