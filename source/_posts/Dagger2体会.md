---
title: Dagger2体会
date: 2016-08-11 17:01:56
tags: Android 第三方
comments: true
---


## 前言
最近刚来我司，开始入手公司的项目，MVP，RxJava，Dagger2搭建的框架。对于我这个刚没多长时间的Android菜鸟，着实花了一段时间，说点儿题外话，最近研究了Java8，已经开始由命令式编程过度到函数式编程了，尤其是加了Lambda表达式，配起来RxJava的切换线程，异步操作爽呆了。

至于MVP，开始我是只知道MVC，但在看我司的项目架构的时候发现有差别，后来在简书上看到一个博主，醍醐灌顶，明白了MVP其实跟MVC是不一样的，就单单Model来说，在MVC中只是我们的一些javaBean，而在MVP它涉及到数据的来龙去脉，数据是来自内存，硬盘，还是网络。已经数据将会怎样存储都包含其中，至于MV的不同，请大家看[MVP](https://hexo.io/)。

接下来就是这篇博客的重点了，Dagger2，相信大家都或多或少用过或者听说过。让大家津津乐道的就是它的依赖注入，之前有瞭解过依赖注入，知道他最大的好处是解藕，大学时候没好好研究，今天我想说一下我的心得。
<!--more-->

## 几个疑问

### 什么是解耦？

解耦就是松耦合，在我们开始学Java的时候new过各种事物，(PS世间万物皆对象)，但当工程变大，你的java文件过多，一旦需求妹妹过来跟你娇滴滴的说，构造函数加个参数呗，毕竟是妹子求，怎能不从，然后你开始昏天黑地的找，这种最直观的牵一发而动全身的体验就是耦合，很直观吧。官方的说法就是，在A中new B违反了单一职责原则，B 只能在A中new，违反了开闭原则。我们需要做的就是弱化这种关系。


### 什么是依赖注入？

那么依赖注入这个思想是什么那，假设A需要B，我们要做的不是硬编码new B，而是通过一个工厂去生产它，你看到这一定蒙蔽了，工厂不也得new吗，讲道理，是这样，但你想，当你想改B的有参构造或是其他，你只需要改他的工厂。设计模式就是这样，你可能会感觉，好麻烦啊，我开始也是这样，觉得明明几行，为什么用了设计模式会这么多代码。担当开始接收项目，你会感觉自己的代码越写越难以维护，藕合度越高，最直观的感受，写的自己都心力交瘁，牵一发动全身，好了，回归正题，设想假如你在100个类里面new 了B，当B改变的时候你是不是要去一个个改，但是假如你只是修改他的工厂那就不用了，因为B 的源头是你在用到它的时候由工厂注入，无论是初始化加载，还是lazy加载，都一样。

其实依赖注入我们一定见过，这里聚一下例子：
#### 构造函数

``` Java
public class A {
    B b;
    public A(B b) {
        this.b = b;
    }
}
```

#### set方法注入
``` Java
public class A {
    B b;
    public void setB(B b) {
        this.b = b;
    }
}
```

#### 接口注入
``` Java
interface InjectInterface() {
    public void InjectB(B b);
}

public class A implements InjectInterface {
    B b;
    @override
    public void InjectB(B b) {
        this.b = b;
    }
}
```

#### 注解方式(也是Dagger框架的主要方式)
``` Java
public class A {
    @Inject
    B b;
}
```

### 关于Dagger2的依赖注入？

ok，我觉得他流弊的地方是设计思想。dagger的最终目的是依赖注入，是解耦，但是他的实现方法很流弊。接下来我们来聊一聊他的几个keywords: Inject，Component，Module，Provides，这里允许我盗一张图(ps: [图的来源]( http://www.jianshu.com/p/cd2c1c9f68d4), 这篇文章的作者就是上面的MVP的作者。非常感谢大哥，之前也详细解答了我很多)

![关系图](http://upload-images.jianshu.io/upload_images/1504173-0b81f8a57768a703.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### Inject:
两个作用:

1.标识哪里需要被注入。

2.标识哪里可以提供注入。
``` Java
public class A {
    // 这里需要被注入
    @Inject
    B b;
}

public class B {
    // 提供注入
    @Inject
    B() {
      ...
    }
}
```

#### Commponent

字面理解就是主持人，它更像是我们的分发器。发负责得到B的实例，并去给被标识注入的地方进行注入。定点投放。正所谓来龙去脉知道，我们需要Commponent来实现这个注入过程。
``` Java
public class B {
    // 提供注入
    @Inject
    B() {
      ...
    }
}

public class A {
    // 这里我们需要找到找到B的实例进行注入，也就是需要Commponent来建立联系
    @Inject
    B b;
}


@Commponent
public interface Commponent {
    // 这就是让A，B发生联系的地方
    void Inject(A a);
}

// 这里我们修改一下Class A

public class A {
    // 这里我们需要找到找到B的实例进行注入，也就是需要Commponent来建立联系
    @Inject
    B b;

    void A() {
        // Dagger2 自动生成的组件用来注入，这样也就让A，B建立了联系
        DaggerComponent.builder()
                       .build()
                       .inject(this);
    }

}
```

#### Model

讲到这里，你会想，既然目标，来源都用Inject注解了，注入器Commponent也已经ok了，为何要用到Model，其实也很好理解，假如我们项目用第三方或者公司封装好的类库，你不可能指望你去打开它去修改，在你需要的位置加上Inject，这样的话，我们就可以用Model，将第三方我们需要的类暴露出来。Module其实是一个简单工厂模式，Module里面的方法基本都是创建类实例的方法。
还是上代码清晰一些:
``` Java
@Model
public class A {
    // B 为我们第三方lib中的类
    B ProduceB() {
        return new B();
    }
}
```
然而这样并没有完成，Model仅仅是创建实例的方法，我没还没有让他跟我的Component发生联系，我们需要Model可以跟我之前Inject需要注入的地方发生联系，接下来就引出来我下面的keyword: Provides

#### Provides

接着我们上面的话题，不要停。上面说到我们需要让我的Model跟我们的Commponent建立联系，这样我们可以提供Provides标注我们需要的构造方法，这样就实现了我们的需求。是不是很神奇。然后我们简单改一下上面的代码。

``` Java
@Model
public class A {
    // Provides来标注
    @Provides
    B ProduceB() {
        return new B();
    }
}
```
### 关于Model，Inject优先级？
这里Dagger2处理的优先级是：Model > Inject
也就是在初始化构造的时候，Dagger2会先去查找Model有没有Provide我们需要的构造方法，假如没有，它会去查找Inject。

### 关于有参无参的构造？
无参直接就create这个类，有参就去查看@model的@Provide，然后再查 Inject 来构造我们这个类需要的有参构造的参数，过程中如果又发现还有需要构造参数的就继续查@model的@Provide  然后再查 Inject 以此类推。

## 简单总结
Dagger2的出现大大加快了Android的MVP模式的开发。而今天我说的是基础方面的，至于还有很多例如 Qualifier（限定符）、Singleton（单例）、Scope（作用域）、我会在接下来的Blog中进行分析，由衷地希望有看到这篇博客的童鞋，假如发现我的理解有问题，及时纠正我。

最后郑重声明，感谢[牛大哥](http://www.jianshu.com/users/2ce7b74b592b/latest_articles).
