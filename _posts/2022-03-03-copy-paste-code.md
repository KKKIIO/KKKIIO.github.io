---
layout: post
title: "心安理得地复制代码"
date: 2022-03-03 11:35:00 +0800
categories: development redundancy
---

> 强韧和反脆弱性的系统不必像脆弱的系统一样，后者必须精确地理解这个世界，因而它们不需要预测，这让生活变得简单许多。
> 要看看冗余是一种多么缺乏预测性，或者更确切地说，预测性更低的行为模式，让我们借用一下第 2 章的说法：如果你把多余的现金存入银行（再加上储藏在地下室的贸易品，如猪肉和豆泥罐头，以及金条），你并不需要精确地知道哪些事件可能会陷你于困境。这些事件可能是一场战争、一场革命、一场地震、一次经济衰退、一场疫情、一次恐怖袭击，或者新泽西州的分裂等任何事情，但你并不需要作太多的预测。
>
> 《反脆弱》 — [美] 纳西姆·尼古拉斯·塔勒布

## 重复

几年前我刚到一个 Golang 团队里工作，主要业务是对接一些供应商，完成虚拟产品的线上发货。

当时节奏很紧凑，很多工作是在对接五花八门的供应商 HTTP API，代码大概是这样的：

```go
func Payment(request *Request) (result *Result) {
	result = &Result{Status:Pending}
	url := buildUrl(request)
	body := buildBody(request)
	requestLog.URL = url // 1. noisy log
	requestLog.RequestBody = body // log
	response, err:= http.PostJson(url, body) // 2. global func!
	requestLog.Err = err // log
	if err != nil {
		result.Err = err
		return
	}
	requestLog.ResponseBody = response.body
	handleResponse(response, &requestLog, &result)		// 3. useless indirection
	return
}

func handleResponse(response *Response, requestLog *RequestLog, result *Result) {
	var body FooProviderResponse
	if err := json.Unmarshal(response.body, &body); err != nil {
		result.Err = err
		return
	}
	requestLog.ProviderErrorCode = body.code // log
	requestLog.ProviderErrorMessage = body.message // log
	// ...
}
```

## 优化

关于代码的扩展性和可读性，可能不同人观点不同，但像重复地记录日志这种工作，只会增加我的工作量和出错率，这我可忍不了，立马写了个工具类：

```go
type HttpCall struct {
	Log *RequestLog
}

func (c *HttpCall) PostJson(url string, body []byte) (*Response, error) {
	c.Log.StartTime = time.Now()
	c.Log.Url = url
	c.Log.RequestBody = body
	response, err := http.PostJson(url, body)
	c.Log.EndTime = time.Now()
	c.Log.Err = err
	if err != nil {
		return nil, err
	}
	c.Log.ResponseBody = response.Body
	return response, nil
}

func Payment(request *Request) *Result {
	// ...
	url := fmt.Sprintf(urlTmpl, someValue)
	body := PaymentRequest {}
	call := HttpCall{&requestLog}
	response, err:= call.PostJson(url, body)
	// ...
}
```

简洁、对称的代码。

可惜，QA 给我提了一个 Bug，说我没有记 `ProviderErrorCode` 和 `ProviderErrorMessage` 。这些是供应商的错误码和错误信息，需要先解析各不相同的供应商回复内容，才能拿到数据，这也导致记录的位置比较隐晦，新人经常漏记。

解决这一个 Bug 我只需加两句代码，但我可不想以后还要检查有没有漏记日志，我想做有趣的事，想解决困难的问题，想顺便展示一下我高超的编程水平。我需要写几个抽象：

```go
type PaymentResponse interface {
	GetErrorCode() string
	GetErrorMessage() string
	GetPayStatus() int32
}
type ProviderApiClient interface {
	DoPayment(request *Request, log *RequestLog)(PaymentResponse, error)
}
```

我还需要写一个流程来使用这个抽象：

```go
type PaymentFlow struct {
	Cli ProviderApiClient
}
func (f *PaymentFlow) Payment(request *Request) (result *Result) {
	// ...
}
```

我还在接下来的几个新供应商里都用了这些代码，抽象开始增加，流程代码更加通用和复杂，也就是——跟框架一样。一次合作中，同事认为我增加了不必要的复杂度，被激怒的我批评他“甘愿写屎一样的代码”。

下个供应商由[VISA](https://developer.visa.com/)提供 API，这次的 API 有些复杂：

1. 有 [Message Level Encryption](https://developer.visa.com/pages/encryption_guide)（PM：加解密失败都要有日志）
2. 提供 Check Transaction Status API（PM：要记录 transaction id，下次重试用）
3. 错误码有的放在 [Http Status Code](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)，有的放在 Body 里（PM：判断分支按照这个表格走）

那一周我都在加班，带着我修修补补的框架。

到了测试阶段，QA 过来找我：“为什么会改动其他供应商的文件？因为 go 接口变了？这不在本次测试范围内，有风险”。

我麻了。

### 冗余

没多久又要接新的供应商。

这次我先复制了以前的代码，然后根据需求修改、简化。整个过程迅速、简单，写出来的代码 Bug 少、隔离性强，也没有人说我写代码很绕了。

我悟了，我理解了[KISS](https://en.wikipedia.org/wiki/KISS_principle)，理解了[重复好过依赖](https://yosefk.com/blog/redundancy-vs-dependencies-which-is-worse.html)，理解了[反脆弱](https://book.douban.com/subject/25782902/)。

## 那么，代价呢？

Wiki 上说，复制代码的代价是它增加了项目的[长期维护成本](https://en.wikipedia.org/wiki/Duplicate_code#Costs_and_benefits)，当时在低风险和低维护成本之间，我毫不犹豫地选择了前者，但我没意识到，冗余是有长期风险的。

### TransactionV2

团队一直用一个自研、简单的 ORM 库，数据库事务用一个简单函数实现：

```go
func Transaction(db string, fn func(o *Orm) error) (err error) {
	o: NewOrm(db)
	if err = o.Begin(); err != nil {
		return err
	}
	defer func() {
		if p := recover(); p != nil {
			err = fmt.Errorf("%v", p)
			return
		}
		if err != nil {
			o.Rollback()
			return
		}
		err = o.Commit()
	}()
	return fn(o)
}
```

当时我参与开发了一个重要项目，中途一个主力同事休假了，收尾工作由我负责。

项目上线，一个离线服务没有响应任何请求，也没有崩溃。

气氛有些紧张，我赶紧查 goroutine 都在做什么：

1. 导出函数调用栈，发现很多 goroutine 都卡在获取数据库连接了，很可能是某个地方没有释放连接
2. 尝试用常见 panic 错误搜索日志，定位到一个空指针 Bug 导致 panic
3. 查看调用链，发现实现事务的 `Transaction` 函数处理 panic 时不 `Rollback` 事务

这是一次顺利的问题排查，我有些自豪，开完会的 Leader 刚坐下来，我就过去跟他说了这些问题，并就`Transaction` 的问题提了一个自然的 Hotfix 修复方案：“只需在 `Transaction` 里只需增加一行 `Rollback` 代码”。

为了严谨，我补充了一句：“看上去不会增加额外问题，但理论上影响范围包含**所有**服务”。

Leader 有些迟疑。

我紧张地又补了句：“也可以增加一个带 `Rollback` 的 `TransactionV2` ，只替换本次导致 `panic` 的几处 `Transaction` 调用，影响范围**小**，也明确”。

Leader 选择了第二种方案，我增加了 `TransactionV2` 函数。

### 救火

一天晚上我洗完澡，看到有 Leader 的未接来电，我赶紧回拨，Leader 说我负责的模块有告警，让我帮忙看一下怎么回事。

打开电脑，发现工作群里同事们都在排查，资损有些严重。跟同事们讨论，他们已经确认跟我负责的模块没关系，但我不好意思直接收工，就“主动”地参与问题排查。

检查了一会日志，我发现有些 goroutine 像是消失了，没有继续写日志。刚好群里发过调用栈 Dump，打开一看，又是数据库连接没有释放的问题。于是我熟练地找到 panic 日志，发到群里解释，panic 问题很快被修复了。

### 1%的责任

第二天早上，一些同事夸我能力强，很快就定位到这个隐秘的问题。我谦虚地告诉他们，之所以定位快是因为我遇过这个 Bug，当时还写了`TransactionV2` 来解决问题。

“你既然知道这个 Bug，为什么不直接修正原来的 `Transaction` 函数呢？”

我紧张地解释这个修复方案是 Leader 确定下来的。

Leader 也知道了，问了我一样的问题。

不久，一位关系不错的同事在他给我的 Peer Feedback 上写道：

> 他重构了一些有隐藏 bug 的方法，并给出了 v2 版本，但是只在群里同步了这个问题，但有时候很容易被群消息淹没，我觉得可以把这些上升给上级，让上级在组内向下推动，避免一些问题的发生。

## 回顾

冗余有低风险、简单、灵活等好处，但是会提高长期维护成本和风险，项目的健康发展需要持续维护代码质量。
