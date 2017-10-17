---
title: RecyclerView 知识点
data: 2017-09-25
tags: Android View
comments: true
---

## 前言
好久没写博客了，到新公司差不多一个月了，之前做Android TV 开发，现在开始做手机端了，写手机端App 或许是一个很久的执念吧。
就最近的页面需求，好好研究了一下ViewGroup，RecyclerView 的绘制，这里打算立个flag，记一下自己踩的坑。
先立个自己的思路：
***这里再提议嘴，一般我们处理一个view的UI交互，动效，最好来封装控件，从我们的mvp，mvvm等等模式里抽离动画，交互的具体代码！！！！！！！***
## 最近的一个需求
最近碰到一个需求，如视频所示：
<!--more-->
<video width="494" height="878" controls>
    <source src="https://raw.githubusercontent.com/JarvisGG/JarvisBlog/master/source/video/demo.mp4">
</video>

如图所示动画过程：
1.上下滚动是，底图是跟着浮动的（item从头到尾滑动recycerview的高度，但是图片需要滚动全屏距离）.
2.点击item，列表展开.
3.底图放大到全屏.

## 涉及到的知识点
### itemDecoration 是什么
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
这里我们看到是直接在**recyclerview的画布上用Canvas**画出来的，这地方很重要。
### ItemView 的绘制
我们可以看源码
``` Java
@Override
protected int getChildDrawingOrder(int childCount, int i) {
    return super.getChildDrawingOrder(childCount, i);
}
```
可以通过控制这个方法来决定recyclerview itemview的绘制顺序
``` Java
@Override
protected void dispatchDraw(Canvas canvas) {
    super.dispatchDraw(canvas);
}
```
这里是具体执行recyclerview 绘制itemview的地方
之前我在做recyclerview itemview滑到屏幕中间的时候放大，由于itemview的绘制顺序，放大的itemview会被后面的itemview盖住，这里我通过复写这个方法来完成。
``` Java

@Override
public void onChildAttachedToWindow(View child) {
    this.post(() -> {
        if (child instanceof ContainerCardView) {
            mContainerCardView.put((ContainerCardView) child, TopRecyclerView.this.getChildAdapterPosition(child));
        }
    });
    super.onChildAttachedToWindow(child);
}

@Override
protected void dispatchDraw(Canvas canvas) {
    // 第一遍绘制所有view
    super.dispatchDraw(canvas);
    // 开始绘制特定view，保证层级最高
    List<ContainerCardView> crashChildViews = new ArrayList<>();
    crashChildViews.addAll(mContainerCardView.keySet());

    for (int i = 0; i < crashChildViews.size(); i++) {
        super.drawChild(canvas, crashChildViews.get(i), this.getDrawingTime());
        crashChildViews.get(i).setTag("draw");
    }
}
```
itemview的layout
``` Java
@Override
public void onLayoutChildren(Recycler recycler, State state) {
    super.onLayoutChildren(recycler, state);
}
```
### recyclerview（ViewGroup） 以及 itemview 重绘
对于重绘我们应该多做注意：
1.频繁重绘，会导致vsync信号间隔，gpu来不及绘制framequeue里面的frames，导致丢帧。
2.itemview 重绘会导致Android TV焦点丢失。
3.itemview 重绘，假如外层的viewgroup是wrapcontent，将会导致递归到根节点（宽高给死的viewgroup）的所有view重绘。假如其中有layout位置动画，将会导致动画失效。
4.避免子view重绘制导致上层跟着重绘制，最简单的方式是给定上层viewgroup固定的框高，任你子view变化莫测，我岿然不动，这样就可以保存一些layout的动画状态。

## 具体的实现方式
### 拆分动画
#### recyclerview 展开，收缩动画：
``` Java
/**
    * 展开动画
    * @param childView
    * @param selectedPosition
    */
public void excuteExtendAnim(View childView, int selectedPosition) {
    this.mTargetView = childView;
    this.mSelectPosition = selectedPosition;
    
    int screenPosition[] = new int[2];
    childView.getLocationOnScreen(screenPosition);

    List<Animator> excuteChildAnim = new ArrayList<>();
    List<View> excuteChilds = new ArrayList<>();
    for (int i = 0; i < getChildCount(); i++) {
        View child = this.getChildAt(i);
        int position = getChildAdapterPosition(child);
        if (position < selectedPosition) {
            excuteChilds.add(child);
            final ObjectAnimator objectAnimator = ObjectAnimator.ofFloat(child, View.TRANSLATION_Y, 0, -screenPosition[1] + mParentScreenPosition[1]);
            objectAnimator.setDuration(mDuration);
            excuteChildAnim.add(objectAnimator);
        } else if (position > selectedPosition) {
            excuteChilds.add(child);
            final ObjectAnimator objectAnimator = ObjectAnimator.ofFloat(child, View.TRANSLATION_Y, 0, Utils.getScreenSizeY(mContext) - screenPosition[1] - childView.getHeight());
            objectAnimator.setDuration(mDuration);
            excuteChildAnim.add(objectAnimator);
        }
    }
    excuteItemAnim(excuteChildAnim, false);
}

/**
    * 收缩动画
    */
public void excuteShrinkAnim() {
    List<Animator> excuteChildAnim = new ArrayList<>();
    for (int i = 0; i < getChildCount(); i++) {
        View child = this.getChildAt(i);
        int position = getChildAdapterPosition(child);
        if (position != mSelectPosition) {
            float startY = child.getTranslationY();
            final ObjectAnimator objectAnimator = ObjectAnimator.ofFloat(child, View.TRANSLATION_Y, startY, 0);
            objectAnimator.setDuration(200);
            excuteChildAnim.add(objectAnimator);
        }
    }
    excuteItemAnim(excuteChildAnim, true);
}

private void excuteItemAnim(List<Animator> excuteChildAnim, boolean isShrink) {

    AnimatorSet set = new AnimatorSet();
    set.playTogether(excuteChildAnim);
    set.setInterpolator(new DecelerateInterpolator());
    set.start();
    notifyAnimStateStart(null, !isShrink);
    set.addListener(new Animator.AnimatorListener() {
        @Override
        public void onAnimationStart(Animator animator) {
        }

        @Override
        public void onAnimationEnd(Animator animator) {
            notifyAnimStateEnd(animator, !isShrink);
        }

        @Override
        public void onAnimationCancel(Animator animator) {
            notifyAnimStateCancel(animator, !isShrink);
        }

        @Override
        public void onAnimationRepeat(Animator animator) {
            notifyAnimStateRepeat(animator, !isShrink);
        }
    });
}
```
#### recyclerview 滚动，itemview 的imageview 的移动动画：
recyclerview：
``` Java
/**
 * 定义接口
 */
public interface ScrollStateCallback {
    public void onScrollStateChanged(RecyclerView recyclerView, int newState);
    public void onScrolled(int headerHeight, int totalHeight, int dx, int dy);
}
／**
 * 将实现这个接口的cardview 添加到集合
 *／
@Override
public void onChildAttachedToWindow(View child) {
    super.onChildAttachedToWindow(child);
    if (child instanceof ScrollStateCallback) {
        mScrollStateCallback.add((ScrollStateCallback) child);
    }
    initChildOperator(child);
}
／**
 * 将实现这个接口的cardview 移除集合
 *／
@Override
public void onChildDetachedFromWindow(View child) {
    super.onChildDetachedFromWindow(child);
    if (child instanceof ScrollStateCallback) {
        mScrollStateCallback.remove(child);
    }
    detachChildOperator(child);
}

private void initView(Context context) {
    this.mContext = context;
    this.addOnScrollListener(new OnScrollListener() {
        @Override
        public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
            super.onScrollStateChanged(recyclerView, newState);
            notifyScrollStateChanged(recyclerView, newState); // 开始调用方法
        }

        @Override
        public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
            super.onScrolled(recyclerView, dx, dy);
            notifyScrolled(recyclerView, dx, dy);// 开始调用Scroll方法
        }
    });
}

private void notifyScrolled(RecyclerView recyclerView, int dx, int dy) {
    for (ScrollStateCallback callback : mScrollStateCallback) {
        callback.onScrolled(mHeaderHeight, mTotalHeight, dx, dy);
    }
}

private void notifyScrollStateChanged(RecyclerView recyclerView, int newState) {
    for (ScrollStateCallback callback : mScrollStateCallback) {
        callback.onScrollStateChanged(recyclerView, newState);
    }
}

```
cardview 继承 ScrollStateCallback:
``` Java
@Override
public void onScrollStateChanged(RecyclerView recyclerView, int newState) {

}

@Override
public void onScrolled(int mHeaderHeight, int mTotalHeight, int dx, int dy) {
    excuteAdCardAnim(mHeaderHeight, mTotalHeight, dx, dy);
}

/**
 * 数学公式推倒
 */
public void excuteAdCardAnim(int mHeaderHeight, int mTotalHeight, int dx, int dy) {
    int topOffset = this.getTop();
    int a = topOffset + mHeaderHeight;
    int b = topOffset * mTotalHeight;
    int c = mTotalHeight - mHeaderHeight;
    int top = a - b / c - mHeaderHeight;
    if (top > 0) {
        top = topOffset;
    }
    // 这里要注意我刚才提出的重回问题，不然这块儿的动画会因为重绘制失效
    this.mImageView.dragPosition(
            mImageView.getLeft(),
            top - topOffset,
            mImageView.getRight(),
            top + mImageView.getHeight() - topOffset);
}
```
#### 关键的动画已经基本完成，现在存在一个问题，我上访的titlebar也要动画，消失，这里就不列出代码了，很简单，问题是我的recyclerview 的目标itemview 需要也保持移动来保证跟后面的imageview保持一致：
``` Java
@Override
public void onAnimationStart(Ad.Creative creative, boolean isShrink) {
    this.mCreative = creative;
    mZHFloatAdFullView.setCreative(mCreative);
    AnimatorSet set = new AnimatorSet();
    List<Animator> animatorList = new ArrayList<>();

    if (isShrink) {

        ValueAnimator downAnim = ValueAnimator.ofInt(0, mLastTopHeight);
        downAnim.addUpdateListener(new AdLogoViewAnimatorUpdateListener(mLastTopHeight, mLastItemHeight));
        animatorList.add(downAnim);

        ObjectAnimator parentAnim = ObjectAnimator.ofFloat(this, View.TRANSLATION_Y, 0);
        animatorList.add(parentAnim);

        ZHFloatAdCardView cardView = mZHFloatAdRecyclerView.getCurrentClickView();
        ObjectAnimator clickAnim = ObjectAnimator.ofFloat(cardView, View.TRANSLATION_Y, 0);
        animatorList.add(clickAnim);

    } else {
        int currentTop = mZHFloatAdRecyclerView.getCurrentAnimItemLogoViewTop();
        mLastTopHeight = currentTop;
        mLastItemHeight = mZHFloatAdRecyclerView.getCurrentAnimItemMargetTop();

        ValueAnimator upAnim = ValueAnimator.ofInt(currentTop, 0);
        upAnim.addUpdateListener(new AdLogoViewAnimatorUpdateListener(mLastTopHeight, mLastItemHeight));
        animatorList.add(upAnim);

        ObjectAnimator parentAnim = ObjectAnimator.ofFloat(this, View.TRANSLATION_Y,  -mHeaderHeight);
        animatorList.add(parentAnim);

        ZHFloatAdCardView cardView = mZHFloatAdRecyclerView.getCurrentClickView();
        ObjectAnimator clickAnim = ObjectAnimator.ofFloat(cardView, View.TRANSLATION_Y, -currentTop);
        animatorList.add(clickAnim);

    }

    set.playTogether(animatorList);
    set.setInterpolator(new AccelerateDecelerateInterpolator());
    set.addListener(new Animator.AnimatorListener() {
        @Override
        public void onAnimationStart(Animator animator) {

        }

        @Override
        public void onAnimationEnd(Animator animator) {
            Log.e("topOffset ------> ", ZHFloatAdFloatView.this.getTop()+"");
        }

        @Override
        public void onAnimationCancel(Animator animator) {

        }

        @Override
        public void onAnimationRepeat(Animator animator) {

        }
    });
    set.setDuration(sAdNormalAnimDuration);
    set.start();
}

class AdLogoViewAnimatorUpdateListener implements ValueAnimator.AnimatorUpdateListener {

    int totalTop;
    int itemTop;

    public AdLogoViewAnimatorUpdateListener(int totalTop, int itemTop) {
        this.totalTop = totalTop;
        this.itemTop = itemTop;
    }

    @Override
    public void onAnimationUpdate(ValueAnimator valueAnimator) {
        int top = (int)valueAnimator.getAnimatedValue();
        if (mZHFloatAdRecyclerView.getCurrentClickAdCardType() == ZHFloatAdCardView.ADCardViewType.FLOAT) {
            mAdLogoView.dragPosition(
                    mAdLogoView.getLeft(),
                    top,
                    mAdLogoView.getRight(),
                    top + mAdLogoView.getHeight());
        }
    }
}
```
### 手势控制
展开第二级页面，我们要监控手势，往上滑，就控制滑动到webview，在webview状态想上滑，就滑到顶部，漏出imageview。
``` Java
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {

    if (isOvering) {
        return true;
    }

    if (mCurrent == 0) {
        return true;
    }
    if (mWebView.getScrollY() == 0) {
        switch (ev.getAction()) {
            case ACTION_DOWN:
                lastY = y = ev.getY();
                break;
            case ACTION_MOVE:
                y = ev.getY();
                deltaY = (int) (lastY - y);
                if (deltaY < -mTouchThreshold) {
                    isOvering = true;
                    mZHFloatAdFloatView.excuteScrollAnim(mContext, this, false,
                            () -> {
                        isOvering = false;
                        setCurrent(0);
                    });
                }
                break;
            case ACTION_UP:
                break;
            default:
                break;
        }
    }
    return super.onInterceptTouchEvent(ev);
}

@Override
public boolean onTouchEvent(MotionEvent event) {

    if (isOvering) {
        return true;
    }

    if (mCurrent == 0) {
        switch (event.getAction()) {
            case ACTION_DOWN:
                lastY = y = event.getY();
                return true;
            case ACTION_MOVE:
                y = event.getY();
                deltaY = (int) (lastY - y);
                if (deltaY > mTouchThreshold) {
                    isOvering = true;
                    mZHFloatAdFloatView.excuteScrollAnim(mContext, this, true,
                            () -> {
                                isOvering = false;
                                setCurrent(1);
                            });
                }
                break;
            case ACTION_UP:
                break;
            default:
                break;
        }
    }
    return super.onTouchEvent(event);
}
```
这里Touch事件的传递方式就不说了。自行google。
### webview的基本配置
``` Java
private void initWebSettings() {
    WebSettings settings = mWebView.getSettings();
    //支持JS
    settings.setJavaScriptEnabled(true);
    //支持插件
    settings.setPluginState(WebSettings.PluginState.ON);
    //设置适应屏幕
    settings.setUseWideViewPort(true);
    settings.setLoadWithOverviewMode(true);
    //支持缩放
    settings.setSupportZoom(false);
    //隐藏原生的缩放控件
    settings.setDisplayZoomControls(false);
    //支持内容重新布局
    settings.setLayoutAlgorithm(WebSettings.LayoutAlgorithm.NORMAL);
    settings.supportMultipleWindows();
    settings.setSupportMultipleWindows(true);
    //设置缓存模式
    settings.setDomStorageEnabled(true);
    settings.setDatabaseEnabled(true);
    settings.setCacheMode(WebSettings.LOAD_DEFAULT);
    settings.setAppCacheEnabled(true);
    settings.setAppCachePath(mWebView.getContext().getCacheDir().getAbsolutePath());

    //设置可访问文件
    settings.setAllowFileAccess(true);
    //支持自动加载图片
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
        settings.setLoadsImagesAutomatically(true);
    } else {
        settings.setLoadsImagesAutomatically(false);
    }
    //设置编码格式
    settings.setDefaultTextEncodingName("UTF-8");
}

private void initWebViewClient() {
    mWebView.setWebViewClient(new WebViewClient() {

        //页面开始加载时
        @Override
        public void onPageStarted(WebView view, String url, Bitmap favicon) {
            super.onPageStarted(view, url, favicon);
        }


        //页面完成加载时
        @Override
        public void onPageFinished(WebView view, String url) {
            super.onPageFinished(view, url);
        }

        //是否在WebView内加载新页面
        @Override
        public boolean shouldOverrideUrlLoading(WebView view, WebResourceRequest request) {
            return super.shouldOverrideUrlLoading(view, request);
        }

        @Override
        public boolean shouldOverrideUrlLoading(WebView view, String url) {
            if (url.contains(HTTP) || url.contains(HTTPS)) {
                return super.shouldOverrideUrlLoading(view, url);
            } else if (UriHandlerDispatcher.handleUri(getContext(), url)) {
                return true;
            } else {
                return super.shouldOverrideUrlLoading(view, url);
            }
        }

        //网络错误时回调的方法
        @Override
        public void onReceivedError(WebView view, WebResourceRequest request, WebResourceError error) {
            super.onReceivedError(view, request, error);
        }

        @Override
        public void onReceivedHttpError(WebView view, WebResourceRequest request, WebResourceResponse errorResponse) {
            super.onReceivedHttpError(view, request, errorResponse);
        }

        @Override
        public void doUpdateVisitedHistory(WebView view, String url, boolean isReload) {
            super.doUpdateVisitedHistory(view, url, isReload);
            mWebView.clearHistory(); // 不保存历史，不允许goBack
        }
    });
}

private void initWebChromeClient() {

    mWebView.setWebChromeClient(new WebChromeClient() {

        @Override
        public void onReceivedTitle(WebView view, String title) {
            super.onReceivedTitle(view, title);
        }

        @Override
        public void onProgressChanged(WebView view, int newProgress) {
            super.onProgressChanged(view, newProgress);
        }

        @Override
        public Bitmap getDefaultVideoPoster() {
            return super.getDefaultVideoPoster();
        }
    });
}
```
## 总结
之前走了不少坑。这个是PlanB+，之前的心路历程PlanA -> PlanB -> PlanC -> PlanB+。😂 
好在看了不少源码，感觉收获挺大的。后续打算还是继续结合具体需求，整理记录知识点儿。



