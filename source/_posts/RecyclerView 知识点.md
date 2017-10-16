---
title: RecyclerView 知识点
data: 2017-09-25
tags: Android View
comments: true
---

## 前言
好久没写博客了，到新公司差不多一个月了，之前做Android TV 开发，现在开始做手机端了，写手机端App 或许是一个很久的执念吧。
就最近的页面需求，好好研究了一下ViewGroup，RecyclerView 的绘制，这里打算立个flag，记一下自己踩的坑。
## 最近的一个需求
最近碰到一个需求，如视频所示：
<!--more-->
<video width="494" height="878" controls>
    <source src="../demo.mp4">
</video>

如图所示动画过程：
1.上下滚动是，底图是跟着浮动的（item从头到尾滑动recycerview的高度，但是图片需要滚动全屏距离）.
2.点击item，列表展开.
3.底图放大到全屏.

## 涉及到的知识点
### itemDecoration 是什么？
这里我们先看看源码：
``` Java
public static abstract class ItemDecoration {
        
        public void onDraw(Canvas c, RecyclerView parent, State state) {
            onDraw(c, parent);
        }

        @Deprecated
        public void onDraw(Canvas c, RecyclerView parent) {
        }

        public void onDrawOver(Canvas c, RecyclerView parent, State state) {
            onDrawOver(c, parent);
        }

        @Deprecated
        public void onDrawOver(Canvas c, RecyclerView parent) {
        }

        @Deprecated
        public void getItemOffsets(Rect outRect, int itemPosition, RecyclerView parent) {
            outRect.set(0, 0, 0, 0);
        }
        public void getItemOffsets(Rect outRect, View view, RecyclerView parent, State state) {
            getItemOffsets(outRect, ((LayoutParams) view.getLayoutParams()).getViewLayoutPosition(),
                    parent);
        }
    }
```


