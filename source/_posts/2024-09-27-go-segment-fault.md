# 不用 unsafe ，怎么让 Go 程序 Segment Fault

公司一个 Go 服务 panic 了，与往常不一样的是，错误不是解空指针，而是 `unexpected fault address` —— 访问非法地址。

```
unexpected fault address 0x1a6d
fatal error: fault
[signal SIGSEGV: segmentation violation code=0x1 addr=0x1a6d pc=0x1a6d]

goroutine 69514 gp=0xc000336fc0 m=4 mp=0xc000080008 [running]:
runtime.throw({0x53decb?, 0x10?})
	/usr/local/go/src/runtime/panic.go:1023 +0x5c fp=0xc00031e600 sp=0xc00031e5d0 pc=0x438c1c
runtime.sigpanic()
	/usr/local/go/src/runtime/signal_unix.go:895 +0x285 fp=0xc00031e660 sp=0xc00031e600 pc=0x44fce5
kkkiio.io/algo/fib.unwrap({0x5157a0?, 0xc0005a8990?})
	/home/kkkiio/projects/algo/fib/fib.go:62 +0x1e fp=0xc00031e670 sp=0xc00031e660 pc=0x505e3e
kkkiio.io/algo/fib.Await.func2()
	/home/kkkiio/projects/algo/fib/fib.go:47 +0x3a fp=0xc00031e6a8 sp=0xc00031e670 pc=0x505d5a
```

日志指出，panic 是因为调用了一个函数值。

```go
func unwrap(susp any) any {
	for {
		switch value := susp.(type) {
		case func() any:
			susp = value() // <- runtime.sigpanic
		default:
			return value
		}
	}
}
```

调用函数导致 panic，这可能吗？

<!--more-->

## "safe" Go

调用函数导致 panic，说明函数地址是错的。
也就是说，函数值里的函数指针值是错的。

奇怪的是，代码中并没有使用 `unsafe` 包操作指针。
只是会并发调用一个函数：

```go
func Fib(loader *Loader, key string) any {
	n, _ := strconv.Atoi(key)
	if n == 0 {
		return 0
	}
	if n == 1 {
		return 1
	}
	children := []any{strconv.Itoa(n - 1), strconv.Itoa(n - 2)}
	value_susps := make([]any, len(children))
	for i, child := range children {
		value_susps[i] = loader.Load(Fib, child.(string))
	}
	return func() any {
		values := Await(value_susps).([]any)
		var res int
		for _, v := range values {
			res += v.(int)
		}
		return res
	}
}

func Await(root any) any {
	var result any
	q := []func() any{func() any {
		v := unwrap(root)
		result = v
		return v
	}}
	for len(q) > 0 {
		f := q[0]
		q = q[1:]
		switch value := unwrap(f).(type) {
		case []any:
			for i, elem := range value {
				sub_i := i
				sub_susp := elem
				q = append(q, func() any {
					v := unwrap(sub_susp)
					value[sub_i] = v
					return v
				})
			}
		default:
		}
	}
	return result
}
```

Go 是一个有 GC 的语言，这意味着非空指针指向的内存都是能用的。
再加上不用`unsafe`包， Go 是不允许你做指针运算的，所以不应该出现访问非法地址的错误。

理论上来说。

## safe to `go` ?

代码里还有个 `Loader` ，它用来并发加载数据，并缓存结果：

```go
type Loader struct {
	mu    sync.Mutex
	cache map[string]func() any
}

func NewLoader() *Loader {
	return &Loader{cache: make(map[string]func() any)}
}

func (l *Loader) Load(f func(loader *Loader, key string) any, key string) func() any {
	l.mu.Lock()
	defer l.mu.Unlock()
	if cachef, ok := l.cache[key]; ok {
		return cachef
	}
	done := make(chan struct{})
	var res any
	go func() {
		res = f(l, key)
		close(done)
	}()
	await := func() any {
		<-done
		return res
	}
	l.cache[key] = await
	return await
}
```

这里用了锁和 `channel` 来同步，看起来是安全的。
但它引入了跨 goroutine 的共享数据 `res`。

获取 `res` 时有用 `done` 这个 `channel` 同步，如果 `res` 不会被修改，那便无事发生。
倘若 `res` 会被修改，就会导致[数据竞争(data race)](https://en.wikipedia.org/wiki/Race_condition#Data_race)。

## Data Race

Data Race 可以简单理解为线程访问到的内存数据可能不对。

可能有人以为只是会拿到旧数据，但实际上 Data Race 可能拿到不应该存在的脏数据。

比如读一个 64 位整数`int64`，可能会拿到新旧数据各一半的脏数据。[^1]

更常见的是，读一个结构体时，可能拿到的是结构体不同字段拼在一起的脏数据。

而 Go 原生的 `interface` 值，恰好是一个包含类型`type`和值`value`的结构体。

![gorace1](http://research.swtch.com/gorace1.png)

按这个思路，`unwrap` 里读到的 `susp` ，可能是 `type` 为 `func()` ，`value` 却是其他值的脏数据。

## 按图索骥

回过头来再看调用栈，导致 panic 的 `unwrap` 调用来自 `Await` 这部分代码：

```go
for i, elem := range value { // <- read fib.go:43
	sub_i := i
	sub_susp := elem
	q = append(q, func() any {
		v := unwrap(sub_susp) // <-
		value[sub_i] = v // <- write fib.go:48
		return v
	})
}
```

如果说是 Data Race 造成了 `sub_susp` 变脏数据，那也吻合这里对 `[]any` 数组元素的修改。

数组只能是 `Fib` 里的 `value_susps`。
`value_susps` 会被 `Fib` 返回的闭包引用，而后者作为 `Load` 里的 `res` 被共享。

当 `Fib` 并发调用时，不同 `goroutine` 会从 `cache` 里拿到同一个 `Fib` 闭包，对同一个 `value_susps` 使用 `Await`。
当一个 `goroutine` 想把 `unwrap` 结果，`interface{}`类型的整数，写入数组时，另一个 `goroutine` 可能刚好在读，就读到了 `type` 为 `func()` ，`value` 是 0x1a6d [^2] 的脏数据。

## 防止 Data Race

出于性能考虑[^3]，Go 只提供同步机制，开发者如果在使用共享数据时没有正确使用同步，就会导致 Data Race。

这符合 Go 语言的设计哲学：大道至简。
难做的事情就不做，让开发者自己处理。

Go 还提供了 [Data Race Detector](https://go.dev/doc/articles/race_detector) 工具，可以检测出 Data Race ：

```
$ go test -race ./fib
==================
WARNING: DATA RACE
Read at 0x00c000076690 by goroutine 33:
  kkkiio.io/algo/fib.Await()
      /home/kkkiio/projects/algo/fib/fib.go:43 +0x3f1
...
...
Previous write at 0x00c000076690 by goroutine 45:
  kkkiio.io/algo/fib.Await.func2()
      /home/kkkiio/projects/algo/fib/fib.go:48 +0xb
```

[^1]: [Golang: what is atomic read used for?](https://stackoverflow.com/a/55840680)
[^2]: 0x1a6d 正好是 `fib(20)`
[^3]: [Off to the Races - by Russ Cox](https://research.swtch.com/gorace)
