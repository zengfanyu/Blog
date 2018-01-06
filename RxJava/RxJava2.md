title: 只因为在众多框架中多看了你一眼 RxJava （二） 从概念上理解 RxJava
date: 2018-1-6 21:27:15
tags:
- RxJava
categories: [Android]
---

![](https://i.imgur.com/cCjinS0.png)
摘要：一开始学习 RxJava ，觉得这东西特抽象，不好把握，但是在阅读了一些文章之后，也逐渐对 RxJaba 在宏观上有一个大概的认识，这一篇就记录一下我对 RxJava 最基础的理解。

<!-- more -->

<font face=黑体>

# RxJava 是什么 #

想要知道 RxJava 是什么，当然还是要去看看 RxJava 在 Github 上面的主页，毕竟官方的介绍是最准确的。

RxJava – Reactive Extensions for the JVM – a library for composing asynchronous and event-based programs using observable sequences for the Java VM.

对，这就是官方对 RxJava 的解释，翻译一下：RxJava---JVM的反应式扩展。 是一个使用可观测的序列来组成异步的，基于事件序列的库。这么一说，我还是不能理解这个库到底是干嘛的，毕竟这是官方站在一定的高度上对 RxJava 的总结，就类似于在 Java 中的一句话：Everything is objects，刚开始接触 Java 时肯定也无法理解这句话的意思 。

既然官方的解释对于初学者的我来说这么难懂，那就先看看 Rx 是什么意思吧。Rx---全称是Reactive Extensions，直译过来就是响应式扩展。Rx**基于观察者模式**，他是一种编程模型，目标是提供一致的编程接口，帮助开发者更方便的处理异步数据流。ReactiveX.io给的定义是，Rx是一个使用可观察数据流进行异步编程的编程接口，ReactiveX结合了观察者模式、迭代器模式和函数式编程的精华。

这么一捋之后，我似乎有一点点理解了，它完成的功能似乎就是和 Google 官方的 AsyncTask Handler 类似，**解决异步问题的**，**但是为什么大家都用它而不去使用 AsyncTask Handler 呢？ 这些也都是解决异步的呀？** 

答案是：**简洁**，逻辑上的简洁。


所以这里就一句话，**RxJava 是用一种扩展的观察者模式解决异步问题的基于事件序列的库**。

所以我就从观察者模式开始学习了。

## 观察者模式 ##

那就要先说说观察者模式了。

打个比方来说，我们公司外面的馄饨店，每天中午好多人都回去哪里吃馄饨，大家过去了之后「点了一碗馄饨并付账」，就坐在那里等着老板叫号了。老板一叫：25号鲜肉小馄饨好了，那么一哥们听到之后，就立马「屁颠颠的跑过去端回来吃」，老板又叫：28号荠菜大馄饨好了，一妹子又「屁颠颠的跑过去端回来吃」。这个过程就是一个观察者模式。

这里涉及到三个对象，观察者：坐着等馄饨的吃货们。被观察者：馄饨店老板。订阅关系：点了一碗馄饨并付账。 还涉及到一个具体的反应过程：「屁颠颠的跑过去端回来吃」。

图解：

![](https://i.imgur.com/AhpeI4B.png)




再说说在 Android 开发中会经常接触到的观察者模式，对一个 Button 的监听。Button 是被观察者，OnClickListener 是观察者，两者通过 setOnClickListener 方法达成订阅关系，那么此时 OnClickListener 就对 Button 的点击事件高度敏感了，只要 Button 以被用户点击，那么 OnClickListener 就会“立马”（这个立马，要打个引号，应为这里还涉及到事件的分发过程，不展开说了）做出反应，也就是调用它的 onClick 方法。


图解：

![](https://i.imgur.com/6z9gunU.png)


其实上面两个过程我们也可以这样理解，**观察者和被观察者之间先有订阅关系，然后被观察者发出了一个事件，然后观察者接收到这个事件之后，做出了响应**。

在从事件出发的角度捋捋上面两个过程：

吃货去馄饨店里「点了一碗馄饨并付账」，那么就**订阅了馄饨店老板发出的 HuntunReady 事件**。当馄饨店老板**发出了一个 HuntunReady 的事件**（图中没有画出）的时候，吃货A**接收到这个事件**并做出反应，也就是**调用了 onHuntunReady 方法去处理**,具体的处理逻辑就是：「屁颠颠的跑过去端回来吃」（图中也没有画出）。

OnClickListener 通过 setOnclickListener 订阅 Button 的 Click 事件，当Button 发出了一个 Click 的事件， OnClickListener 接受到了这个事件，就要做出了反映，也就是调用 onClick 方法，执行当中的逻辑。

对照表：

![](https://i.imgur.com/7HYKYLy.png)


那么我们把这个概念抽象为观察者，被观察者，订阅，那么图应该是这样的：


![](https://i.imgur.com/E31Ycqd.png)


## RxJava 当中的观察者模式 ##

有了上面两个例子的铺垫，这里就推导出，RxJava 就是 Observer 和 Observable 两者先发生订阅关系，然后 Observable 发出事件序列，Observer 接受事件并响应处理。

但之前又说到 RxJava 使用的是一种扩展的观察者模式，那么跟上面两个例子中肯定是有不同的。它扩展之后的形式如下：


![](https://i.imgur.com/HYAnpoS.png)



### 区别（一） ###

看的出来， RxJava 中的观察者模式跟传统观察者模式比起来，事件的回调方法除了普通事件 onNext（相当于上面例子中的 onClick 和 onHuntunReady）之外，还定义了两个特殊的事件， onCompleted()， onError();

这三个事件遵从一定的规则：

1. Observable 可以发送无限个 onNext 事件，Observer 可以接受无限个 onNext 事件。
2. 当 Observable 发送了 onCompleted 或者 onError 序列之后，之后的事件序列仍然会发送，但是 Observer 这边在接收到 onCompleted 或者 onError 事件之后，是不会在继续接收之后的事件序列的。
3. Observable 可以不发送 onCompleted 或者 onError 事件。
4. onCompleted 和 onError 事件必须互斥。也就是说在一个事件序列中，要么是没有 onCompleted 和 onError 事件，要么有且只能有其中一个。


### 区别（二） ###

就是 Observable 在发送事件到 Observer 的过程中，多了一个 Operate 过程，这个操作就是对事件进行一系列的处理，然后再发送至 Observer。要说到这个操作就不得不提 Java8 当中的 Stream 和它对其中每一个元素进行的函数式操作。

我们用“过滤”这个操作来打个比方，


>
>
>就像下面这幅图中画的那样，我有一杯混合着大大小小石子的蓝色的水。
>
>
>![](http://www.uwanttolearn.com/wp-content/uploads/2017/03/war_against_learning_curve_of_rx_java_2_java_8_stream_1.jpg)
>
>现在按照我们关于“流”的定义，我用下图中的方法将水转化成“流”。
>
>![](http://www.uwanttolearn.com/wp-content/uploads/2017/03/war_against_learning_curve_of_rx_java_2_java_8_stream_2.jpg)
>
>
>为了让水变成水流，我把水从一个杯子倒进另一个杯子 里。现在我想去掉水中的大石子，所以我造了一>
>
>个可以帮我滤掉大石子的过滤器。“大石子过滤器”如下图所示。
>
>![](http://www.uwanttolearn.com/wp-content/uploads/2017/03/war_against_learning_curve_of_rx_java_2_java_8_stream_3.jpg)
>
>现在，将这个过滤器作用在水流上，这会得到不包含大石子的水。如下图所示。
>
>![](http://www.uwanttolearn.com/wp-content/uploads/2017/03/war_against_learning_curve_of_rx_java_2_java_8_stream_4.jpeg)

>接下来，我想从水中清除掉所有石子。已经有一个过滤大石子的过滤器了，我们需要造一个新的来过滤>小石子。“小石子过滤器”如下图所示。
>
>![](http://www.uwanttolearn.com/wp-content/uploads/2017/03/war_against_learning_curve_of_rx_java_2_java_8_stream_5.jpg)
>
>像下图这样，将两个过滤器同时作用于水流上。
>
>![](http://www.uwanttolearn.com/wp-content/uploads/2017/03/war_against_learning_curve_of_rx_java_2_java_8_stream_6.png)
>
>接下来，我想把水的颜色从蓝色变成黑色。为了达到这个目的，我需要造一个像下图这样的“水颜色转换器（mapper）”。
>
>![](http://www.uwanttolearn.com/wp-content/uploads/2017/03/war_against_learning_curve_of_rx_java_2_java_8_stream_7.jpg)
>
>像下图这样使用这个转换器。
>
>![](http://www.uwanttolearn.com/wp-content/uploads/2017/03/war_against_learning_curve_of_rx_java_2_java_8_stream_8.jpg)
>
>把水转换成水流后，我们做了很多事情。我先用一个过滤器去掉了大石子，然后用另一个过滤器去掉了>小石子， 最后用一个转换器（map）把水的颜色从蓝色变成黑色。
>
>这个过程就是 Java8 当中对流的操作的一个具体的体现。


好了，现在可以回到 Rxjava 当中了。

RxJava 当中的 operate 过程也类似于上面的过滤过程，在 Observable 发出事件之后，可以利用「操作符」对事件进行一系列的操作，包括但不仅仅局限于 “过滤”、“合并”、“线程切换”等等，得到我们最终想要的“过滤”后的事件。




恩，这就是 RxJava 当中最基础的东西了，先对 RxJava 有一个宏观上的认识，后面才好继续学习。



