---
title: Android TV 焦点探索
date: 2017-07-20 1:06:56
tags: Android 源码
comments: true
---

## 前言
毕业从事Android TV 开发一年了，这一年萦绕着我，挥之不去的就是焦点逻辑，TV 区别于手机的UI交互区别，应该就是focus事件与touch事件了，而国内关于Android TV的盘子还不如手机端踩得多，那么今天打算总结一下焦点的几个疑惑。这里我会分焦点搜索，焦点丢失来谈谈。踩踩盘子。
<!--more-->
## 焦点搜索
这块儿Android源码很有意思，谷歌爹们真的牛啊，好了，当我们用遥控器按上下左右键的时候，我们会调用focusSearch，我们结合源码看一下.
``` Java
public View focusSearch(@FocusRealDirection int direction) {
    if (mParent != null) {
        return mParent.focusSearch(this, direction);
    } else {
        return null;
    }
}
```
focus 当前获得焦点的View，direction当前按键事件方向，我们可以看到这里View会将焦点上抛给Parent，接个我们进去看。
``` Java
public View focusSearch(View focused, int direction) {
    if (isRootNamespace()) {
        // root namespace means we should consider ourselves the top of the
        // tree for focus searching; otherwise we could be focus searching
        // into other tabs.  see LocalActivityManager and TabHost for more info
        return FocusFinder.getInstance().findNextFocus(this, focused, direction);
    } else if (mParent != null) {
        return mParent.focusSearch(focused, direction);
    }
    return null;
}
```
这里我们看到有一个是否是根空间的判断，倘若当前
