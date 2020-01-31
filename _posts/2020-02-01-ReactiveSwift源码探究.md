---
layout: post
title: ReactiveSwift源码探究-事件转换
date: 2020-02-01
categories: blog
tags: [Swift, ReactiveSwift]
description: ReactiveSwift

---

由于网上已经有[一个系列的文章](https://www.cnblogs.com/ludashi/p/6908859.html)很详细地在解析ReactiveSwift源码了，我这里就不再细致解析每句代码了。在这里，我主要是解答自己的一些疑问点和记录自己的理解。

## 事件流

要理解ReactiveSwift，首先需要知道一个概念：事件流。可以理解为一个串行的数据管道，一个个事件在里面流动。这个事件流里面的事件，在ReactiveSwift中就是Event这个枚举了。每个事件流都对应一个数据类型和错误类型，表示这个事件流中会流过的数据类型。

```
enum Event {
    case value(Value)  // 数据
    case failed(Error) // 事件流发送错误，事件流终止
    case completed     // 事件流完成了，不再有value过来了
    case interrupted
}
```

这个Event还实现了map方法，用来对自身进行转换，在函数式编程中，这个Event就是一个函子。另外这个Event中还提供很多静态转换操作方法(`Transformation`)，比如filter、compactMap、take、skip、map等等。但是这些转换操作方法并不是把Event转换成另一个Event，而是把监听者转换为另一个监听者。下面会介绍观察者和监听者，也会讲到这个`Transformation`。
                                             
## Observer是什么

Observer 这个类，既包含了观察者的职能，又包含了监听者的职能。

```
let observer1 = Signal<Int, Never>.Observer(value: { print("\($0)") })
observer1.send(value: 2)

// 输出2
```

上面代码我们初始化一个`Observer`时，我们传入了一个block，当发生value事件时，会自动回调这个block，这里这个block就相当于监听者。接着我们直接用这个创建的`Observer`来发送通知，上面的block（监听者）就会收到这个通知（被调用）

在Observer类中为一个闭包声明了一个别名`Action`

```
public typealias Action = (Event) -> Void
private let _send: Action
```

这个`Action`，我们可以理解为监听者，当有事件来时，会直接调用这个block

```
// 这里发送事件，就会立刻通知监听者，调用Action这个block
public func send(_ event: Event) {
    _send(event)
}
```


## Signal

Signal的init初始化方法只有一个

```
public init(_ generator: (Observer, Lifetime) -> Void) {
    core = Core(generator)
}
```

这个Core类就是用来管理这个事件流的，为什么不直接写到Signal里管理呢？这是为了让后Signal在没有外部引用和观察者时，能够清理自己（self dispose），避免循环引用。

我们知道，信号是可以转换的，Signal里面提供了大量的转换方法，有map、take、skip、throttle等等，这是信号流中非常重要和有用的地方，使得我们可以自定义自己的观察行为。这么多转换中，都会调用到一个方法

```
internal func flatMapEvent<U, E>(_ transform: @escaping Event.Transformation<U, E>) -> Signal<U, E> {
    return Signal<U, E> { output, lifetime in
        let input = transform(output.send, lifetime)
        lifetime += self.observe(input)
    }
}
```

这个方法是传入一个`Transformation`转换操作，从而来生成一个新的Signal。这个`Transformation`的定义在Event中

```
extension Signal.Event {
	internal typealias Transformation<U, E: Swift.Error> = (@escaping Signal<U, E>.Observer.Action, Lifetime) -> Signal<Value, Error>.Observer.Action
}
```

这是一个Block，接收一个Action和Lifetime，返回一个新的Action。大家还记得上面说过的这个Action吗？它是事件发生时会调用的Block。

举个例子，我们看看Event中提供的filter操作

```
internal static func filter(_ isIncluded: @escaping (Value) -> Bool) -> Transformation<Value, Error> {
    return { action, _ in
        return { event in
            switch event {
            case let .value(value):
                if isIncluded(value) {
                    action(.value(value))
                }

            case .completed:
                action(.completed)

            case let .failed(error):
                action(.failed(error))

            case .interrupted:
                action(.interrupted)
            }
        }
    }
}
```

一开始看到这种return里面又有return的代码，可能会有点懵。我们慢慢来看。首先这个函数返回一个Block，这个Block的作用是接收Action和Lifetime，返回一个新的Action（这个Action也是一个Block）。因此里面嵌套的那个return，返回来的是一个新的Action，在这个新的Action里面，我们调用了旧的Action来通知原来的监听者，这样，我们前面的转换操作会影响后面的操作和监听，形成一条链式结构。

然后我们新建一个Signal，传入这个新的监听者Action，并返回这个新的Signal，也就是说，我们在对信号流进行转换处理时，其实是新建了一个信号流Signal，也就有下面的链式调用，其中skipRepeats操作会影响后面的map操作。

```
signal.skip(first: 1)
    .skipRepeats()
    .map(value: String.self)
    .observeValues { (value) in
        print("value come \(value)")
    }
``` 

还有一点需要特别注意的，那就是flatMapEvent 中调用transform来获取新的Action时，传入的这个旧的action，并不是Observer中的内部的_send这个变量，而是Observer中的下面这个函数地址

```
public func send(_ event: Event) {
    _send(event)
}
```
