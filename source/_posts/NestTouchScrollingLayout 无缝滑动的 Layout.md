---
title: NestTouchScrollingLayout 无缝拖拽的 Layout
date: 2018-12-05 23:16:00
tag: Android 开源控件
comments: true
---

## 前言
今年年初接触回答页面改版，由之前的左右滑动回答改为上下滑动回答，由于当时回答页的代码太过于庞大，所以第一次改版复用了之前的 UI 框架，外层 ViewPager + Fragment，内层是 WebView 嵌套 Hybrid 页面。

问题出现了，WebView 可以滚动的时候，会持有整个 Touch 事件流程，导致当 webView 拖拽到底部，手指不脱离屏幕继续拖拽的时候，无法将当前的拖拽操作给翻页器，产生体验上的割裂感。

接下来就是 UI 交互优化的历程
<!--more-->
## 调研
### 1.NestedScrolling:
Support V4 提供了一套 API 来支持嵌入的滑动效果。NestedScrolling 提供了一套父 View 和子 View 滑动交互机制。要完成这样的交互，父 View 需要实现 NestedScrollingParent 接口，而子 View 需要实现 NestedScrollingChild 接口。

作为一个可以嵌入 NestedScrollingChild 的父 View，需要实现 NestedScrollingParent，这个接口方法和 NestedScrollingChild 大致有一一对应的关系。同样，也有一个 NestedScrollingParentHelper 辅助类来默默的帮助你实现和 Child 交互的逻辑。滑动动作是 Child 主动发起，Parent 就收滑动回调并作出响应。

从上面的 Child 分析可知，滑动开始的调用 startNestedScroll()，Parent 收到 onStartNestedScroll() 回调，决定是否需要配合 Child 一起进行处理滑动，如果需要配合，还会回调 onNestedScrollAccepted()。

每次滑动前，Child 先询问 Parent 是否需要滑动，即 dispatchNestedPreScroll()，这就回调到 Parent 的 onNestedPreScroll()，Parent 可以在这个回调中“劫持”掉 Child 的滑动，也就是先于 Child 滑动。

Child 滑动以后，会调用 onNestedScroll()，回调到 Parent 的 onNestedScroll()，这里就是 Child 滑动后，剩下的给 Parent 处理，也就是 后于 Child 滑动。

最后，滑动结束，调用 onStopNestedScroll() 表示本次处理结束。

这个方案其实很不错，但最后被 pass 了，因为由于工程的原因，我们的 webview 是被包裹起来的，不可以任意去继承 NestedScrollingChild 并做定制修改。


### 2.自定义 ViewGroup

其实目前的问题是当子 View scroll 到顶部或者底部的时候，无法将 Touch 事件流交还给父布局。
因此这里我采用的思路是通过我的 ViewGroup 去统一 dispatchTouchEvent 给我的子 View，条件就是，假如子 View 可以滚动，我就会构造一套完整的 touch 时间流分发给他。否则我会自己消化。

## 解决方案
### step 1:
通过第二种方式的思路，我们第一步需要在我的 ViewGroup 拦截所有的 Touch 事件。所以...
``` java
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    if (isParentDispatchTouchEvent) {
        return true;
    } else {
        return super.onInterceptTouchEvent(ev);
    }
}
```
上来我们就拦截出所有的 Touch 事件。

### step2:
开始在 ViewGroup 的 onTouchEvent 处理所有相关的 Event。
``` java
// 1.初始记录 Touch 坐标
int mDownY = event.getY();
int deltaY = 0;

// 2.默认子 View 持有事件流起始点 isHoldTouch = true,通过 isChildCanScroll 来判别当前子 View 是否可以滚动。
if (isHoldTouch && !isChildCanScroll(event, deltaY) && deltaY != 0) {
    // 3.假如子 View 不可以滚动，当前 ViewGroup 需要阻断 Touch 的下发，为了遵循 Touch 事件流的规范，当被外部阻断时，需要对其下发 ACTION_CANEL。同时 isHoldTouch = false。
    isHoldTouch = false;
    MotionEvent cancelEvent = MotionEvent.obtain(event);
    cancelEvent.setAction(MotionEvent.ACTION_CANCEL);
    getChildAt(0).dispatchTouchEvent(cancelEvent);
    cancelEvent.recycle();
}

// 5.假如当我们在 ViewGroup 滚动过程中，滑动到了子 View 可滚动的状态，这时候会将 ViewGroup 调整至滚动初始位置，然后对子 View 做一个 ACTION_DOWN 的操作，从而开始陆续分发子 View Touch 事件。同时 isHoldTouch = true。
if (!isHoldTouch && isChildCanScroll(event, deltaY) && deltaY != 0) {
    setSheetTranslation(maxSheetTranslation);
    isHoldTouch = true;
    if (event.getAction() == MotionEvent.ACTION_MOVE) {
        MotionEvent downEvent = MotionEvent.obtain(event);
        downEvent.setAction(MotionEvent.ACTION_DOWN);
        getChildAt(0).dispatchTouchEvent(downEvent);
        downEvent.recycle();
    }
}

if (isHoldTouch && deltaY != 0) {
    // 6.当前判断子 View 已经处于可分发 Touch 状态，会陆续将 ACTION_MOVE 分发给他。从而实现子 View 的滚动。
    event.offsetLocation(0, mSheetTranslation - mTouchParentViewOriginMeasureHeight);
    getChildAt(0).dispatchTouchEvent(event);
} else {
    // 4.当上面阻断完 Touch 的下发以后，这里我们开始自己消化 Touch 事件，也就是这里会做一个 TranslationY 修改，从而达到 ViewGroup 做 Y轴方向的偏移.
    setSheetTranslation(newSheetTranslation);

    if (event.getAction() == MotionEvent.ACTION_UP || event.getAction() == MotionEvent.ACTION_CANCEL) {
        // 7.为了将这个结束事件后面分发给子 View
        isHoldTouch = true;
    }
}
```
### step3:
判断子 View 是否可以滚动
``` java
 /**
    * child can scroll
    * @param view
    * @param x
    * @param y
    * @param lockRect 是否开启 允许 touch 脱离当前子 View 区域继续生效。
    * @return
    */
protected boolean canScrollUp(View view, float x, float y, boolean lockRect) {

    if (view instanceof WebView) {
        return canWebViewScrollUp();
    }
    if (view instanceof ViewGroup) {
        ViewGroup vg = (ViewGroup) view;
        for (int i = 0; i < vg.getChildCount(); i++) {
            View child = vg.getChildAt(i);
            int childLeft = child.getLeft() - view.getScrollX();
            int childTop = child.getTop() - view.getScrollY();
            int childRight = child.getRight() - view.getScrollX();
            int childBottom = child.getBottom() - view.getScrollY();
            boolean intersects = x > childLeft && x < childRight && y > childTop && y < childBottom;
            if ((!lockRect || intersects)
                    && canScrollUp(child, x - childLeft, y - childTop, lockRect)) {
                return true;
            }
        }
    }

    if (view instanceof CoordinatorLayout &&
            ((CoordinatorLayout) view).getChildCount() > 0 &&
            ((CoordinatorLayout) view).getChildAt(0) instanceof AppBarLayout) {
        AppBarLayout layout = (AppBarLayout) ((CoordinatorLayout) view).getChildAt(0);
        OnNestOffsetChangedListener listener = mOnOffsetChangedListener.get(layout.hashCode());
        if (listener != null) {
            if (listener.getOffsetY() < layout.getMeasuredHeight() && listener.getOffsetY() > 0) {
                return true;
            }
        }
    }

    return view.canScrollVertically(-1);
}

/**
    * child can scroll
    * @param view
    * @param x
    * @param y
    * @param lockRect 是否开启 允许 touch 脱离当前子 View 区域继续生效。
    * @return
    */
protected boolean canScrollDown(View view, float x, float y, boolean lockRect) {
    if (view instanceof WebView) {
        return canWebViewScrollDown();
    }
    if (view instanceof ViewGroup) {
        ViewGroup vg = (ViewGroup) view;
        for (int i = 0; i < vg.getChildCount(); i++) {
            View child = vg.getChildAt(i);
            int childLeft = child.getLeft() - view.getScrollX();
            int childTop = child.getTop() - view.getScrollY();
            int childRight = child.getRight() - view.getScrollX();
            int childBottom = child.getBottom() - view.getScrollY();
            boolean intersects = x > childLeft && x < childRight && y > childTop && y < childBottom;
            if ((!lockRect || intersects)
                    && canScrollDown(child, x - childLeft, y - childTop, lockRect)) {
                return true;
            }
        }
    }

    if (view instanceof CoordinatorLayout &&
            ((CoordinatorLayout) view).getChildCount() > 0 &&
            ((CoordinatorLayout) view).getChildAt(0) instanceof AppBarLayout) {
        AppBarLayout layout = (AppBarLayout) ((CoordinatorLayout) view).getChildAt(0);
        OnNestOffsetChangedListener listener = mOnOffsetChangedListener.get(layout.hashCode());
        if (listener != null) {
            if (listener.getOffsetY() < layout.getMeasuredHeight() && listener.getOffsetY() > 0) {
                return true;
            }
        }
    }

    return view.canScrollVertically(1);
}
```
这部分核心在于递归查询当前 MotionEvent 当前坐标下的所有子 View 有没有可以滚动的 View，从而根据 view.canScrollVertically 来进行判断。

## 优势
1.支持绝大多数的 View 嵌套滚动。
2.成本低，只需要在你想要嵌套滚动的 View 上面包一层这个 Layout。

## 功能
1.支持嵌套滚动，无缝拖拽
2.支持 BottomSheet (使用方法详见下方)
3.支持 Appbarlayout
3.支持拖拽阻尼 (使用方法详见下方)


## 效果

|![demo1](https://github.com/JarvisGG/NestedTouchScrollingLayout/blob/master/captures/demo1.gif?raw=true)|![demo2](https://github.com/JarvisGG/NestedTouchScrollingLayout/blob/master/captures/demo2.gif?raw=true)|
|-----------|:-----------:|
|normal|webview|
|![demo3](https://github.com/JarvisGG/NestedTouchScrollingLayout/blob/master/captures/demo3.gif?raw=true)|![demo4](https://github.com/JarvisGG/NestedTouchScrollingLayout/blob/master/captures/demo4.gif?raw=true)|
|bottomsheet normal|bottomsheet appbarlayout|
|![demo8](https://github.com/JarvisGG/NestedTouchScrollingLayout/blob/master/captures/demo8.gif?raw=true)|![demo9](https://github.com/JarvisGG/NestedTouchScrollingLayout/blob/master/captures/demo9.gif?raw=true)|
|view recyclerview|webview recyclerview|
|![demo5](https://github.com/JarvisGG/NestedTouchScrollingLayout/blob/master/captures/demo5.gif?raw=true)|![demo6](https://github.com/JarvisGG/NestedTouchScrollingLayout/blob/master/captures/demo6.gif?raw=true)|
|vip|question|


### Usage example

#### normal use
``` XML
<jarvis.com.library.NestedTouchScrollingLayout
    android:id="@+id/wrapper"
    android:layout_gravity="center"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <android.support.v7.widget.RecyclerView
        android:id="@+id/container_rv"
        android:layout_width="match_parent"
        android:layout_height="400dp"
        android:background="#fff"
        android:overScrollMode="always">
    </android.support.v7.widget.RecyclerView>

</jarvis.com.library.NestedTouchScrollingLayout>
```

``` Java
// 设置手指下拉阻尼
mNestedTouchScrollingLayout.setDampingDown(2.0f / 5);
// 设置手指上拉阻尼
mNestedTouchScrollingLayout.setDampingUp(3.0f / 5);

mNestedTouchScrollingLayout.registerNestScrollChildCallback(new NestedTouchScrollingLayout.INestChildScrollChange() {
        
        // 当前 Layout 偏移距离
	@Override
	public void onNestChildScrollChange(float deltaY) {

	}
	
	// finger 脱离屏幕 Layout 偏移量，以及当前 Layout 的速度
	@Override
	public void onNestChildScrollRelease(final float deltaY, final int velocityY) {
		mNestedTouchScrollingLayout.recover(0, new Runnable() {
			@Override
			public void run() {
				Log.i("NestedTouchScrollingLayout ---> ", "deltaY : " + deltaY + " velocityY : " + velocityY);
			}
		});
	}
	// 手指抬起时机
	@Override
	public void onFingerUp(float velocityY) {

	}

	// 横向拖拽
	@Override
	public void onNestChildHorizationScroll(MotionEvent event, float deltaX, float deltaY) {
	
	}
});
```
#### bottomsheet use
``` xml
 <jarvis.com.library.NestedTouchScrollingLayout
	android:id="@+id/wrapper"
	android:layout_marginTop="30dp"
	android:layout_width="match_parent"
	android:layout_height="match_parent">

	<android.support.v7.widget.RecyclerView
		android:background="#fff"
		android:id="@+id/container_rv"
		android:layout_width="match_parent"
		android:layout_height="match_parent" />

</jarvis.com.library.NestedTouchScrollingLayout>
```
``` java
// 临界速度，根据业务而定
public static int mVelocityYBound = 1300;

// 规定 sheetView 弹起方向
mNestedTouchScrollingLayout.setSheetDirection(NestedTouchScrollingLayout.SheetDirection.BOTTOM);

mNestedTouchScrollingLayout.registerNestScrollChildCallback(new NestedTouchScrollingLayout.INestChildScrollChange() {
	@Override
	public void onNestChildScrollChange(float deltaY) {

	}

	@Override
	public void onNestChildScrollRelease(final float deltaY, final int velocityY) {
		int totalYRange = mNestedTouchScrollingLayout.getMeasuredHeight();
		int helfLimit = (totalYRange - DisplayUtils.dpToPixel(BottomSheetActivity.this, 400)) / 2;
		int hideLimit = totalYRange - DisplayUtils.dpToPixel(BottomSheetActivity.this, 400) / 2;
		int helfHeight = totalYRange - DisplayUtils.dpToPixel(BottomSheetActivity.this, 400);
		if (velocityY > mVelocityYBound && velocityY > 0) {
			if (Math.abs(deltaY) > helfHeight) {
				mNestedTouchScrollingLayout.hiden();
			} else {
				mNestedTouchScrollingLayout.peek(mNestedTouchScrollingLayout.getMeasuredHeight() - DisplayUtils.dpToPixel(BottomSheetActivity.this,400));
			}
		} else if (velocityY < -mVelocityYBound && velocityY < 0) {
			if (Math.abs(deltaY) < helfHeight) {
				mNestedTouchScrollingLayout.expand();
			} else {
				mNestedTouchScrollingLayout.peek(mNestedTouchScrollingLayout.getMeasuredHeight() - DisplayUtils.dpToPixel(BottomSheetActivity.this,400));
			}
		} else {
			if (Math.abs(deltaY) > hideLimit) {
				mNestedTouchScrollingLayout.hiden();
			} else if (Math.abs(deltaY) > helfLimit) {
				mNestedTouchScrollingLayout.peek(mNestedTouchScrollingLayout.getMeasuredHeight() - DisplayUtils.dpToPixel(BottomSheetActivity.this, 400));
			} else {
				mNestedTouchScrollingLayout.expand();
			}
		}
	}

	@Override
	public void onFingerUp(float velocityY) {

	}

	@Override
	public void onNestChildHorizationScroll(MotionEvent event, float deltaX, float deltaY) {

	}
});
```

### Usage
方式 1:
``` Gradle
repositories {
    // ...
    maven { url "https://jitpack.io" }
}

dependencies {
    implementation 'com.github.JarvisGG:NestedTouchScrollingLayout:v1.2.0'
}
```
方式 2:
``` Gradle
repositories {
    // ...
    jcenter()
}
dependencies {
    implementation 'com.jarvis.library.NestedTouchScrollingLayout:library:1.2.0'
}
```

### 源码
https://github.com/JarvisGG/NestedTouchScrollingLayout
### 欢迎大家食用



