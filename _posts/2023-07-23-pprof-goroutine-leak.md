---
layout: post
title: "泄露100万个go协程是什么体验？"
date: 2023-07-23 21:06:00 +0800
categories: engineering
---

![Goroutine Leak](/assets/image/goleak.png)

## 背景

我司有一个Go后台服务，在运行几周后内存占用很高，即使在没有用户使用的闲置时段，它也仍然会占用大约3G的内存。另外，我们发现每天闲置时段内存占用都比前一天更高些，这是典型的内存泄漏现象。

问题根源正如标题所说，是goroutine泄漏，我们可以拿这个作为例子，看看怎么分析内存泄露。

## 还原现场

简化后的BUG代码如下[^1]:

```go
func calculate(multiple int) (int, error) {
	var res int
	var obErr error
	ctx, cancel := context.WithCancel(context.TODO())
	<-asyncFetch(ctx).Map(func(_ context.Context, i interface{}) (interface{}, error) {
		return multiple * i.(int), nil
	}).ForEach(func(i interface{}) {
		res += i.(int)
	}, func(err error) {
		obErr = err
		cancel()
	}, func() {
		// BUG: should always call cancel()
	}, rxgo.WithContext(ctx))
	return res, obErr
}

func asyncFetch(ctx context.Context) rxgo.Observable {
	ch := make(chan rxgo.Item)
	go func() {
		defer close(ch)
		values := []int{1, 2, 3, 4, 5}
		for _, v := range values {
			rxgo.Of(v).SendContext(ctx, ch)
		}
	}()
	return rxgo.FromChannel(ch, rxgo.WithContext(ctx))
}
```

## 用 pprof 分析程序

假设我们不知道问题原因，我们应该如何处理这个内存泄露问题呢？

跟排查所有BUG一样，我们需要信息，而不是浪费时间猜测。 Go 官方提供了[pprof](https://pkg.go.dev/net/http/pprof)工具，可以用来了解CPU/内存/Goroutine等使用情况，这里省略工具的具体使用方法。

### 分析内存使用

我们用pprof生成“内存占用”的火焰图[^2]:

![pprof heap火焰图](/assets/image/goleak-memory.png)

火焰图的宽度是使用资源的比例，可以看到，`runtime.malg`这个函数占用了很多内存，它是用来创建goroutine的，正常情况下它不应该占用很多内存，这种表现说明服务有大量的goroutine在运行，而且都没有退出，大概率是goroutine泄漏了。

### 分析Goroutine状态

大多数程序的火焰图会充满许多细窄的调用栈，你只靠内存火焰图很难分析出是哪里泄露了goroutine，所以我们来进一步分析goroutine的火焰图：

![pprof goroutine火焰图](/assets/image/goleak-goroutine.png)

非常明显，`onecontext.(*OneContext).run`创建了大量的goroutine，而且都没有退出，这就是我们要找的泄漏点。

## 总结

泄露了这么多goroutine，听上去有些吓人，其实也只是多占用了一些资源，我们也只需根据现象去分析程序。

[^1]: https://github.com/ReactiveX/RxGo
[^2]: https://www.brendangregg.com/flamegraphs.html