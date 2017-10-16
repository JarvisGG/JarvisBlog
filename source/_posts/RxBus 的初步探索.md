---
title: RxBus 的初步探索
date: 2017-06-20 19:06:56
tags: RxBus
comments: true
---

## 前言
5月份项目上线了，之后就在优化项目结构，减少依赖。之前项目一直用的EventBus来作为项目事件流的框架，这两天偶然看到RxBus这个东西，基于RxJava和RxAndroid，考虑到自身的业务需求，由于本身用EventBus的功能比较单一，而发现RxBus足以实现我现有的业务，所以决定踩踩坑。
<!--more-->
## 具体实现
``` Java
public class RxBus {
    private static volatile RxBus mInstance;
    private final Subject mBus;

    public RxBus() {
        mBus = new SerializedSubject<>(PublishSubject.create());
    }

    public static RxBus getInstance() {
        if (mInstance == null) {
            synchronized (RxBus.class) {
                if (mInstance == null) {
                    mInstance = new RxBus();
                }
            }
        }
        return mInstance;
    }

    public void post(Object object) {
        mBus.onNext(object);
    }

    public <T> Observable<T> toObserverable(Class<T> eventType) {
        return mBus.ofType(eventType);
//        return mBus.filter(eventType::isInstance)
//                .cast(eventType);
    }
}
```
目前只是消息的注册，发送。
#### SerializedSubject
SerializedSubject 特征是线程安全
``` Java
public SerializedSubject(final Subject<T, R> actual) {
    super(new OnSubscribe<R>() {

        @Override
        public void call(Subscriber<? super R> child) {
            actual.unsafeSubscribe(child);
        }

    });
    this.actual = actual;
    this.observer = new SerializedObserver<T>(actual);
}
```
这里有个小细节，actual 是当前的数据链，这里通过SerializedObserver将数据链做一个转换，类似于map。
下面我们看SerializedObserver
``` Java
public void onNext(T t) {
    if (terminated) {
        return;
    }
    synchronized (this) {
        if (terminated) {
            return;
        }
        if (emitting) {
            FastList list = queue;
            if (list == null) {
                list = new FastList();
                queue = list;
            }
            list.add(NotificationLite.next(t));
            return;
        }
        emitting = true;
    }
    try {
        actual.onNext(t);
    } catch (Throwable e) {
        terminated = true;
        Exceptions.throwOrReport(e, actual, t);
        return;
    }
    for (;;) {
        FastList list;
        synchronized (this) {
            list = queue;
            if (list == null) {
                emitting = false;
                return;
            }
            queue = null;
        }
        for (Object o : list.array) {
            if (o == null) {
                break;
            }
            try {
                if (NotificationLite.accept(actual, o)) {
                    terminated = true;
                    return;
                }
            } catch (Throwable e) {
                terminated = true;
                Exceptions.throwIfFatal(e);
                actual.onError(OnErrorThrowable.addValueAsLastCause(e, t));
                return;
            }
        }
    }
}
```
这里丑抽出onNext，我们发现synchronized线程锁，证明当前是线程安全的，当多个线程再要执行onNext，这里线程安全，排队线程会加入queue，然后依次执行。onError，onComplete同理。
#### PublishSubject
与普通的Subject不同，在订阅时并不立即触发订阅事件，而是允许我们在任意时刻手动调用onNext(),onError(),onCompleted来触发事件。
可以看到PublishSubject与普通的Subject最大的不同就是其可以先订阅事件，然后在某一时刻手动调用方法来触发事件。
demo：
``` Java
PublishSubject<String> publishSubject = PublishSubject.create();
publishSubject.subscribe(new Action1<String>() {
        @Override
        public void call(String s) {
            // TODO
        }
});
publishSubject.onNext(result);
```
我们可以根据我们的业务需求先对Subject进行订阅，然后再默一时刻触发我们的onNext。

## 原理总结
这里的publishSubject就是在我们发出通知的时候才会去onNext，而我们的onNext是线程安全的，当并发访问的时候，可以依次执行onNext，这里我们要用到ofType这个操作符，用来过滤TargetEvent.class的Observable来实现“发送端”与“接收端”的约束。

## 使用方法
简单的使用方法
### 消息发送
``` Java
RxBus.getInstance().post(event);
```
### 消息注册，取消注册
这里就不以Activity，Fragment做对照了，基本用法都一样，风向一个View AttachToWindow,DetachFromWindow 的方式 
``` Java
@Override
protected void onAttachedToWindow() {
    super.onAttachedToWindow();
    mSubscription = RxBus.getInstance().toObserverable(IndexLeftBtnGetFocusEvent.class)
            .compose(RxSchedulers.threadSwitchSchedulers())
            .subscribe(event -> {
                // TODO 业务逻辑
            });
//        EventBus.getDefault().register(this);
}

@Override
protected void onDetachedFromWindow() {
    super.onDetachedFromWindow();
    if (mSubscription.isUnsubscribed()) {
        mSubscription.unsubscribe();
    }
//        EventBus.getDefault().unregister(this);
}
```
## 后记
这里我只是先用一个小demo来学习一下这里的代码设计，后期会对我们的RxBus优化，比如添加bind，unbind生命周期的相关逻辑。
