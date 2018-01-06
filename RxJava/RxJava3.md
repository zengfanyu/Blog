title: 只因为在众多框架中多看了你一眼 RxJava （三） RxJava 最基础的使用
date: 2018-1-5 21:27:15
tags:
- RxJava
categories: [Android]
---

![](https://i.imgur.com/cCjinS0.png)


摘要：上一篇理解了概念之后，这里就要用代码来实现以下了，这一篇只涉及到 RxJava 的最基础的用法，不涉及高级特性。

<font face=黑体>

# RxJava的基本用法

RxJava 的基本实现,总结起来,主要有三点:

1. 创建 Observer
2. 创建 Observable
3. 订阅

## 创建 Observer 观察者

**Observer 即观察者,它决定当事件发生的时候,会有怎么样的行为**.RxJava 中 Observer 接口的实现方式:

```java

	Observer<String> observer = new Observer<String>() {
	    @Override
	    public void onNext(String s) {
	        Log.d(tag, "Item: " + s);
	    }
	
	    @Override
	    public void onCompleted() {
	        Log.d(tag, "Completed!");
	    }
	
	    @Override
	    public void onError(Throwable e) {
	        Log.d(tag, "Error!");
	    }
	};

```

除了 Observer 接口之外, Rxjava 当中还内置了一个实现了 Observer 接口的抽象类, Subscriber , Subscriber 对 Observer 做了扩展,但是两者的使用方式是完全一样的.

```java

	Subscriber<String> subscriber = new Subscriber<String>() {
	    @Override
	    public void onNext(String s) {
	        Log.d(tag, "Item: " + s);
	    }
	
	    @Override
	    public void onCompleted() {
	        Log.d(tag, "Completed!");
	    }
	
	    @Override
	    public void onError(Throwable e) {
	        Log.d(tag, "Error!");
	    }
	};

```

既然 观察者的作用是 决定当事件触发时,会有什么样的行为,那么我们能不能直接有方法定义出行为就可以了呢?

```java

	Action1<String> onNextAction = new Action1<String>() {
	    // onNext()
	    @Override
	    public void call(String s) {
	        Log.d(tag, s);
	    }
	};
	//Action1接口是RxJava当中定义的,其中只有一个call(T param)方法,这个call方法有1个参数,无返回值
	Action1<Throwable> onErrorAction = new Action1<Throwable>() {
	    // onError()
	    @Override
	    public void call(Throwable throwable) {
	        // Error handling
	    }
	};
	//Action0接口是RxJava当中定义的,其中只有一个call()方法,这个call方法无参数无返回值
	Action0 onCompletedAction = new Action0() {
	    // onCompleted()
	    @Override
	    public void call() {
	        Log.d(tag, "completed");
	    }
	};

```
至于 onNextAction onErrorAction onCompletedAction 如何使用,为什么几个 Action 就能代表 Subscriber 的 onNext onError onCompleted 方法呢 ? 下面 Subscribe 订阅的时候说.


## 创建 Observable 被观察者

**Observable 指的是被观察者,它决定事件什么时候触发,以及触发何种事件**. RxJava 用 **create()** 方法来创建一个 Observable ,并为它定义触发规则.

```java

	Observable observable = Observable.create(new Observable.OnSubscribe<String>() {
	    @Override
	    public void call(Subscriber<? super String> subscriber) {
	        subscriber.onNext("Java");
	        subscriber.onNext("C++");
	        subscriber.onNext("Python");
	        subscriber.onCompleted();
	    }
	});

```

- 这里参数传入的是 OnSubscribe 对象,这个对象会被存储在 Observable 中,充当一个**计划表**的角色,具体的计划内容就是 call 方法中的实现.
- call() 方法的参数 Subscriber 就是观察者,两者关系的建立看下面.
- 当 Observable 被订阅的时候, OnSubscribe 的 call 方法会被自动调用,然后去执行计划,上面的代码的计划就是执行三次 onNext() ,然后执行一次 onCompleted().
- 到这里就实现了事件由被观察者向观察者的传递.

看上面的代码,被观察者的作用就是触发事件,那么可不可以有一种类似于 Java8 当中的便捷构造流的方法一样(Stream.of(T...)),直接将事件按照顺序创建出来就行了?当然有
- just(T...)

```java

	Observable observable = Observable.just("Java", "C++", "Python");

	//上述代码等价于创建被观察者,然后依次调用了观察者的 onNext("Java"), onNext("C++"),onNext("Python"),onCompleted()

```

- from(T[]) / from(Iterable<? extends T>) : 将传入的数组或 Iterable 拆分成具体对象后，依次发送出来。

```java

	String[] words = {"Java", "C++", "Python"};
	Observable observable = Observable.from(words);
	// 效果和上面一样

```

## Subscribe 订阅

之前创建了 Observer 和 Observable 之后,就要将两者关联起来了,这就相当于创建了 Button 和 OnClickListener 之后,需要有一个 setOnClickListener 方法,将两者关联起来,这样当 Button 上有点击事件发生的时候,它就回去调用 OnClickListener 的 onClick 方法.这里的 Subscribe 订阅起的作用和 setOnClickListener 是一样的.

```java

	observable.subscribe(observer);
	//或者
	observable.subscribe(subscriber);
	//这个过程就类似于
	//button.setOnClickListener(mOnClickListener);
```

Observable.subscribe(Subscriber) 的内部实现是这样的（仅核心代码）:

```java

	public Subscription subscribe(Subscriber subscriber) {
	    subscriber.onStart();
	    onSubscribe.call(subscriber);
	    return subscriber;
	}

```

订阅过程做三件事:

1. 调用 subscriber.onStart() 做预处理工作.
2. 调用 onSubscribe.call(subscriber) 方法.(这也就是为什么在上面创建被观察者时,我们说 「*当 Observable 被订阅的时候, OnSubscribe 的 call 方法会被自动调用,然后去执行计划*.」)
3. 将传入的 subscriber 返回,便于 unSubscribe().

现在可以说说之前在创建 Observer  时,创建的三个 Action 了.

除了 subscribe(Observer) 和 subscribe(Subscriber) ，subscribe() 还支持不完整定义的回调，RxJava 会自动根据定义创建出 Subscriber 。

```java

	// 自动创建 Subscriber ，并使用 onNextAction 来定义 onNext()
	observable.subscribe(onNextAction);
	// 自动创建 Subscriber ，并使用 onNextAction 和 onErrorAction 来定义 onNext() 和 onError()
	observable.subscribe(onNextAction, onErrorAction);
	// 自动创建 Subscriber ，并使用 onNextAction、 onErrorAction 和 onCompletedAction 来定义 onNext()、 onError() 和 onCompleted()
	observable.subscribe(onNextAction, onErrorAction, onCompletedAction);

```

我们观察可以发现,

- Subscriber 的 onNext , onError 方法的返回类型是 void , 并带有一个参数,分别是 String , Throwable 而 Action1 接口里面的 call 方法也是有一个参数 T ,返回类型是 void.
- Subscriber 的 onCompleted 方法的返回类型是 void, 没有参数,而 Action0 接口里的 call 方法也是没有参数的, 并且返回值为 void.

这么一总结,上面这个操作是不是就像 java8 当中的 lamuda 表达式了 ?   [我在学习lambda表达式时总结的一篇文章](http://zengfanyu.top/2017/11/13/Java8---Lambda%20Expressions/)

```java

	//Before Java8
	mButton.setOnClickListener(new View.OnClickListener() {
	        @Override
	        public void onClick(View view) {
	            System.out.println("Button clicked!");
	        }
	    });
	//In java8 way    
	   mButton.setOnClickListener((v)-> {
	       System.out.println("Button clicked!");
	   });

```
注意看下面的 lambda 表达式中的 (V) ,这里的 v 形参, 就是代表的是上面回调方法中的 view ,后面的 ->System.out.println("Button clicked!") ,就是用于定义 onClick() 的方法体的.

确实类似,这里相当于其他语言中的 「闭包」.

到这里,就已经说完了 RxJava 当中最基础的三个角色了. 那么也可以简单的使用一下啦.


### 简单使用

1)打印一串字符串

```java

	Observable.just("Java","C++","Python")
			  .subscribe(new Action1<String>(){
				@override
				public void call(String languageName){
				    log.d("RxJava","languageName");
				}
		})


```


2)根据 drawable 资源名, 取出 drawable,然后显示到 imageView 上.

```java

	int drawableRes = ...;
	ImageView imageView = ...;
	Observable.create(new OnSubscribe<Drawable>() {
	    @Override
	    public void call(Subscriber<? super Drawable> subscriber) {
	        Drawable drawable = getTheme().getDrawable(drawableRes));
	        subscriber.onNext(drawable);
	        subscriber.onCompleted();
	    }
	}).subscribe(new Subscriber<Drawable>() {
	    @Override
	    public void onNext(Drawable drawable) {
	        imageView.setImageDrawable(drawable);
	    }
	
	    @Override
	    public void onCompleted() {
	    }
	
	    @Override
	    public void onError(Throwable e) {
	        Toast.makeText(activity, "Error!", Toast.LENGTH_SHORT).show();
	    }
	});

```



