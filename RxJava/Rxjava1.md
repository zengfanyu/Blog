title: 只因为在众多框架中多看了你一眼 RxJava
date: 2018-1-4 21:27:15
tags:
- RxJava
categories: [Android]
---

![](https://i.imgur.com/cCjinS0.png)


<font face=黑体>


# 前言 #

年关将近,部门里任务也多了起来,最近一个月多都在忙着部门项目,因为突然接手了三个项目,虽然已经开发了一个版本,但是我还是花了很多时间去消化现有的版本,然后接着迭代开发,昨天上线了,现在有了空余的时间,准备接着写博客了。

之前在写 [MVP 系列第一篇文章](http://zengfanyu.top/2017/10/20/MVP1/)的时候就立了一个 `Flag`， 要学习 `MVP`、`okHttp`、`Rxjava`、`Retrofit2`、`Dagger2`，然后用这些流行开源框架撸一个 `APP`。现在 `MVP` 系列算是有了基础的了解了，并在学 MVP 的时候，把 `OkHttp` 也封装着在 `Model` 层使用，`Retrofit2` 也零零散散看过一些 `Demo` ，`Dagger2` 之前也总结了一篇文章 [Dagger2基础内容归纳](http://zengfanyu.top/2017/11/04/Dagger/)。剩下比较难啃的就是 `RxJava了`。这个任务已经被我添加到我 `2018` 年的计划当中了。

在两年前刚开始接触到 `RxJava` 的时候，是被它的链式调用所吸引，子线程操作、主线程操作、线程切换在链式调用里一气呵成，再也看不到 `AsyncTask` 中繁琐的各种方法，之前在学校里的时候就看过「扔物线」大神的经典入门文章[给 Android 开发者的 RxJava 详解](https://gank.io/post/560e15be2dca930e00da1083) ，但是当时忙于找工作、实习，之后忙着毕业论文、毕业疯，一直没有好好实践，现在准备好好学习一下时发现，已经到了 `RxJava2` 的时代了，之前「扔物线」大神的那篇是基于 `Rxjava1` 写的，是先学学 `RxJava1` 再去学习 `2` 好呢，还是直接去学习 `2` ，思来想去感觉这个问题就像： 学习 `java` 之前要先学习 `C` 吗？ 抛开别的不说，「扔物线」的那篇入门文章是质量是相当高的，深入浅出，对于我这样的初学者来说，学习的价值是十分大的，然后在看完这篇 [RxJava2 vs RxJava1](https://www.jianshu.com/p/850af4f09b61) 对比的文章之后，**我觉得还是很有必要去了解一下 `Rxjava1` 的，这样才能知道 `2` 改进在哪里，为什么这么改进， 是因为 `1` 中使用有什么问题。**

并且在学习了 `java8` 当中的 `Lambda` 以及 `Streams` 之后发现，`RxJava` 当中很多实现都和其十分相似，特别是 闭包 这一特性，至于究竟有什么关系，希望我学习完 `RxJava` 之后能够搞清楚吧。

# 刚接触RxJava的情景还原 #


说说最开始接触到 `RxJava` 的代码吧，应该是 `2016` 年的某一天。

一开始看到的是下面这两段代码：

```java

	//被观察者
	Observable<String> myObservable = Observable.create(
	    new Observable.OnSubscribe<String>() {
	        @Override
	        public void call(Subscriber<? super String> sub) {
	            sub.onNext("Hello, world!");
	            sub.onCompleted();
	        }
	    }
	);
	//观察者
	Subscriber<String> mySubscriber = new Subscriber<String>() {
	    @Override
	    public void onNext(String s) { System.out.println(s); }
	
	    @Override
	    public void onCompleted() { }
	
	    @Override
	    public void onError(Throwable e) { }
	};

	//订阅
	myObservable.subscribe(mySubsciber);

```

好了，一个简单的使用 `RxJava` 的代码就完成了，最初我看到这段代码，心中就是

![](https://i.imgur.com/sp3uNa4.jpg)

打印一个字符串搞这个复杂？？ 这是靠代码量算工资吗？？？

后来人家说，上面那样写太复杂了，正确姿势：

```java

	Observable.just("Hello, world!")
	    .subscribe(new Action1<String>() {
	        @Override
	        public void call(String s) {
	              System.out.println(s);
	        }
	    });

```

人家说这样跟上面那一梭代码功能是一样的。 后来又有人说，这还是太复杂，要简化一下：

```java

	Observable.just("Hello, world!")
	    .subscribe(s -> System.out.println(s));

```

我的内心：

![](https://i.imgur.com/r1DOkxd.jpg)

之后我知道，原来这段单代码使用了 `Java8 Lambda` 表达式，利用函数式编程的优势，简化了程序中余的代码，所以在学习 `RxJava` 之前我应该先去学习一下 `Java8 lambda` 。总结文章：[新姿势学习之Java8---Lambda Expressions](http://zengfanyu.top/2017/11/13/Java8---Lambda%20Expressions/)。

我以为这就行了，但是接下来看到的代码更是让我更加坚定要学习 `Java8 Lambda`、函数式编程、`RxJava`。

要想在 `Subscriber` 当中打印出 `Observable` 发送出来的每一个字符串后面加上 `"-ZFY"`:

```java

	Observable.just("Hello, world!")
	    .map(new Func1<String, String>() {
	        @Override
	        public String call(String s) {
	            return s + " -ZFY";
	        }
	    })
	    .subscribe(s -> System.out.println(s));

```

同样的用 Lambda 简化；

```java

	Observable.just("Hello, world!")
	    .map(s -> s + " -ZFY")
	    .subscribe(s -> System.out.println(s));

```

然后我突然又想输出 接受的字符串拼接上"-ZFY" 的HashCode 的字符串， 那么可以这样：


```java

	Observable.just("Hello, world!")
	    .map(s -> s.hashCode())
	    .map(i -> Integer.toString(i))
	    .subscribe(s -> System.out.println(s));

```

![](https://i.imgur.com/RxHG48a.jpg)

到这里还没有涉及到线程切换的问题，还没有涉及到和 `Retrofit` 配合使用，但是单单就上面那些特性，就足以对我产生强大的吸引力。

就在这一刻，我决定，`Lambda` 、函数式编程、`RxJava` 我也要学，我也要写出上面那样简洁高效的代码。


这就是我刚看到 `RxJava` 的情景。<p>


从今天开始 `RxJava` 的学习。



