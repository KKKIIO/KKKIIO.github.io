---
title: "实用的软件测试"
date: 2024-06-02 00:04:00 +0800
tags: [engineering, backend]
---

> Most of the tests in the 'IntelliJ Platform' codebase are model-level functional tests.
>
> The tests run in a headless environment that uses real production implementations for most components, except for many UI components.
>
> ...
> In a product with 20+ years of a lifetime that has gone through many internal refactorings, we find that this benefit dramatically outweighs the downsides of slower test execution and more difficult debugging of failures compared to more isolated unit tests.
>
> -- JetBrains s.r.o.[^1]

<!--more-->

## 我能注释掉你的测试吗？

在我第一份工作快一年时，我进了个新项目，跟同事两人负责开发一个服务端。

当时的技术决策比较自由，我不想循规蹈矩，希望能参考国外软件工程，写高质量代码。
于是我怂恿同事跟我一起写单元测试，向他吹捧写单元测试的好处。

在实践一段时间后，发现不对劲：

1. 经常要因为一些模块接口的改动去修正单元测试。一些构建系统要求测试也要编译通过。
2. 测试没有测出一些基础 BUG。比如按单元测试理论 Mock 掉数据库等外部依赖，会导致很多 SQL 的问题不会被测出来。

终于在工期压力下，同事忍不住问我，没时间改正测试了，能不能先注释掉，这个尝试便以失败告终。

## 更短，更有效

一年后，我为了少写重复代码，想写一个 GoLand IDE 插件快速生成错误信息。

在快速完成功能开发后，我顺着指引来到了插件测试文档。
出乎意料的是，文档一开始就推荐写功能测试而不是单元测试，理由是功能测试更稳定。

在我参考文档写了几个功能测试后，我发现功能测试的好处远不止于此。
它比单元测试更简洁易懂，也更容易覆盖问题场景。

比如，当我们要测插件的补全功能时，写出来的功能测试大致是：

1. 加载测试输入文件
2. 执行补全命令
3. 选中补全列表中插件的提供补全选项
4. 检查结果是否跟测试输出文件一致

这里每个步骤只需一到两行代码，也能形象地描述出操作过程，还覆盖了整条链路，不用担心 Mock 导致的问题。

花同样的时间，写单元测试估计还在考虑测哪个接口合适，Mock 怎么写，各种类型的参数怎么构造。

我听过一个单元测试的分享，他把十几个访问数据库的函数都 Mock 了，才能跑通测试。
这会让测试变成一个高成本、收益还不明朗的事情。

## 后端服务也是 headless environment

如果把 JetBrains 的测试经验应用到后端服务，会发现它们惊人的相似：

1. 后端服务也是 headless environment，也就是没有 UI 界面的软件。
2. 后端服务也提供稳定的 API 接口。

用 API 做测试，可以解决我之前遇到的代码不稳定和 mock 服务缺乏校验的问题。
它的缺点是依赖更多，比如需要数据库，需要其他微服务。

### 接受数据库依赖

开发后端项目时，一个可用的数据库服务是必备的。
可到了写测试时，人们却因为单元测试的理念对数据库依赖有抵触。

在用过 Python Django 开发项目后，我开始相信，测试依赖数据库带来的问题可以被好的测试框架解决，而好处则是大幅提升的开发效率。
[Django](https://docs.djangoproject.com/en/5.0/topics/testing/overview/#writing-tests) 对依赖数据库的测试支持得很好，它会自动创建一个测试数据库，在所有测试结束后自动删除。
它还用事务隔离了不同测试的数据，在每次测试后会 rollback 过程中的更改，保证测试之间互不影响。

实践中很多项目没有这么便利的测试框架支持，我常用的解决方案是多租户：每个测试用例都新建一个业务上隔离的租户，例如各种 SaaS 软件里的“团队”，在这个租户范围下进行测试。

### 减少微服务依赖

不同于数据库，业务微服务的部署很不稳定。
我工作中遇过的后端项目经常连本地环境都搭不起来（见{% post_link struggling-dev-env %}）。
减少依赖还能提高服务的可用性，降低业务耦合，一举多得。

如果说服不了同事上级，那可以尝试部署依赖服务。
另一个可能的方法是 [Mock 服务](https://github.com/wiremock/wiremock)，但大概率不如放弃写测试 😈。

## 对自己有用=实用

> It’s like being half of a cyborg, or having a jetpack, or something. You write code, and then you ask the computer if the code is correct, and if not then you try again.
>
> -- FoundationDB dev[^2]

测试一大作用是让你更有信心地写代码，不用在繁琐、枯燥的业务规则下推测代码正确性。
在国内这种既要又要的环境下，这更是唯一作用。
不要被上层一些“重视质量”的口号误导，要把关注点放在如何帮助自己提效。

### 本地执行测试

反馈快能大幅提升编程的体验和效率，因此实用的测试需要能本地快速执行。

在 CI 跑测试虽然有部署成本低等优势，但反馈链路太长，需要提交代码并等待（可能是所有）测试结果，通常在十分钟以上。
这会降低你迭代代码的速度，还会成为你编写新测试的阻碍。

### 不只测 API

黑盒测试人员不会关注你的接口，他们只看业务功能是否正确。
那为了阻止 BUG 逃逸到测试人员，功能测试也不应该只关注单一接口，更重要的是通过 API 模拟业务流程，验证结果。

我见过一些“自动化测试”用例，内容基本是：

1. 传一个正常参数，检查错误码是不是 OK。
2. 传一个空参数，检查错误码是不是 Bad Request。
3. 传一个不存在参数，检查错误码是不是 Not Found。

这些测试虽然也有作用，但覆盖的范围小，性价比不高。
比如测试一个删除 API， 即使 API 返回成功，如果不用查询 API 检查目标有没有被移出结果，就有可能有 BUG 逃逸到测试环节。

[^1]: [Testing Overview | IntelliJ Platform Plugin SDK](https://plugins.jetbrains.com/docs/intellij/testing-plugins.html)
[^2]: [Is something bugging you?](https://antithesis.com/blog/is_something_bugging_you/)
