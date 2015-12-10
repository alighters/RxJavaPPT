title: RxJava浅谈
speaker: David 
url: https://github.com/david-wei/RxJavaPPT
transition: move 
files: /js/demo.js,/css/demo.css,/js/zoom.js 
theme: moon
usemathjax: yes

[slide]

# RxJava浅谈
<small>演讲者：卫顺民</small>

[slide]
## ReactiveX
[subslide]
## ReactiveX的历史
---
* ReactiveX是Reactive Extensions的缩写，一般简写为Rx，最初是LINQ的一个扩展，由微软的架构师Erik Meijer领导的团队开发，在2012年11月开源，Rx是一个编程模型，目标是提供一致的编程接口，帮助开发者更方便的处理异步数据流，Rx库支持.NET、JavaScript和C++，Rx近几年越来越流行了，现在已经支持几乎全部的流行编程语言了，Rx的大部分语言库由ReactiveX这个组织负责维护，比较流行的有RxJava/RxJS/Rx.NET，社区网站是 http://reactivex.io/。 {:&.moveIn}


======
## 什么是ReactiveX
---
* 微软给的定义是，Rx可以这样定义：Rx = Observables + LINQ + Schedulers。 {:&.moveIn}
* ReactiveX.io给的定义是，Rx是一个使用可观察数据流进行异步编程的编程接口，ReactiveX结合了观察者模式、迭代器模式和函数式编程的精华。
[/subslide]

[slide]
## RxJava
* Talk is cheap, show me the code.  {:&.fadeIn}
* Read the fucking source code.

[slide]

## 函数接口
+ Func1 {:&.fadeIn}
```
public interface Func1<T, R> extends Function {
    R call(T t);
}
```
+ Action1
```
public interface Action1<T> extends Action {
    void call(T t);
}
```

[slide]
## 简单实现
---
```
Observable observable = Observable.create(new Observable.OnSubscribe<Integer>() {

   @Override
   public void call(Subscriber<? super Integer> subscriber) {
       subscriber.onNext(2);
       subscriber.onCompleted();
   }
});
Subscriber<Integer> subscriber = new Subscriber<Integer>() {
   @Override
   public void onCompleted() {
       System.out.println("订阅完成");
   }

   @Override
   public void onError(Throwable e) {
   }

   @Override
   public void onNext(Integer integer) {
       System.out.println("num:" + integer);
   }
};
observable.subscribe(subscriber);
```

[slide]
## 细节一：OnSubscribe
---  
- 含义：行为 {:&.fadeIn}
- 代码实现
```
public interface OnSubscribe<T> extends Action1<Subscriber<? super T>> {
  // cover for generics insanity
}
```

[slide]
## 细节二: subscribe
---
```
private static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
   ***
  // new Subscriber so onStart it
  subscriber.onStart();
  
  // The code below is exactly the same an unsafeSubscribe but not used because it would 
  // add a significant depth to already huge call stacks.
  try {
      // allow the hook to intercept and/or decorate
      hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
      return hook.onSubscribeReturn(subscriber);
  } catch (Throwable e) {
      
      ***

      // if an unhandled error occurs executing the onSubscribe we will propagate it
      try {
          subscriber.onError(hook.onSubscribeError(e));
      } catch (Throwable e2) {
         ***
      }
      return Subscriptions.unsubscribed();
  }
}
```

[slide]
## Map的用法
---
```
Subscription subscription = Observable.create(new Observable.OnSubscribe<Integer>() {

   @Override
   public void call(Subscriber<? super Integer> subscriber) {
       subscriber.onNext(2);
       subscriber.onCompleted();
   }
}).map(new Func1<Integer, String>() {

   @Override
   public String call(Integer integer) {
       return integer.toString();
   }
}).subscribe(new Action1<String>() {
   @Override
   public void call(String s) {
       System.out.println("result:" + s);
   }
});
```

[slide]
## 源码一：map方法
```
public final <R> Observable<R> map(Func1<? super T, ? extends R> func) {
  return lift(new OperatorMap<T, R>(func));
}
```

[slide]
## 源码二：OperatorMap实现
```
public final class OperatorMap<T, R> implements Operator<R, T> {

    private final Func1<? super T, ? extends R> transformer;

    public OperatorMap(Func1<? super T, ? extends R> transformer) {
        this.transformer = transformer;
    }

    @Override
    public Subscriber<? super T> call(final Subscriber<? super R> o) {
        return new Subscriber<T>(o) {
            ***
            @Override
            public void onNext(T t) {
                try {
                    o.onNext(transformer.call(t));
                } catch (Throwable e) {
                    Exceptions.throwOrReport(e, this, t);
                }
            }

        };
    }

}
```

[slide]
## 源码三：lift操作
```
public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
  return new Observable<R>(new OnSubscribe<R>() {
      @Override
      public void call(Subscriber<? super R> o) {
          try {
              Subscriber<? super T> st = hook.onLift(operator).call(o);
              try {
                  st.onStart();
                  onSubscribe.call(st);
              } catch (Throwable e) {
                  Exceptions.throwIfFatal(e);
                  st.onError(e);
              }
          } catch (Throwable e) {
              Exceptions.throwIfFatal(e);
              o.onError(e);
          }
      }
  });
}
```

[slide]
## schedulers
* Schedulers.immediate()
* Schedulers.trampoline()
* Schedulers.newThread()
* Schedulers.computation()
* Schedulers.io()
* AndroidSchedulers.mainThread()






[slide]
## 参考资料
+ [ReactiveX中文翻译](https://mcxiaoke.gitbooks.io/rxdocs/content/Intro.html)
