---
layout: post
title: "静态的写 React 网页"
date: 2023-11-05 20:45:00 +0800
categories: engineering
---

![next php](/assets/image/next-php.png)

## 'React' Programming

在看完 React 教程开始写 React 应用后，我的感受是：你需要在一个函数执行流程里，对当时所有可能的状态分支做出应对。
新的状态意味着新的逻辑分支。
而 Web 应用常见的会引入状态的场景，就包括调用后端 API。

### data, error, loading

假设我们在写一个记忆卡片(Flash Card)应用。
卡片内容主要是一段话，里面有些被标注的关键词。
我们可以很轻松地写出一个卡片详情页：

```tsx
export function Card({ id }: { id: string }) {
  const {
    data: card,
    error,
    isLoading,
  } = useSWR(`/api/cards/${id}`, cardFetcher);
  const errorMsg = error ? getErrorMsg(error) : undefined;
  return (
    <>
      {isLoading ? (
        <Skeleton variant="rectangular" />
      ) : (
        <CardText text={card.text} keys={card.keys} />
      )}
      {errorMsg && <Alert severity="error">{errorMsg}</Alert>}
    </>
  );
}
```

这里用了[SWR](https://swr.vercel.app/)来使用后端 API。
它提供 React Hooks 的方式来访问数据。
每个查询被划分为三个状态：`data`, `error`, `loading`。
我们把流程判断简化成是否有加载数据，以及是否有错误这两个独立的分支。

目前流程看起来挺清晰，我们再加点编辑功能：用户可以在旁边修改关键词的释义。
先加一个表单：

```tsx
export function KeyForm({
  keyword,
}: {
  keyword: { id: string; meaning: string };
}) {
  const { trigger, isMutating } = useSWRMutation(
    `/api/cards/${cardId}/keys`,
    updateKeyFetcher
  );
  // ...
}
```

这里用了`useSWRMutation` Hook，调用`trigger`来更新关键词。
`trigger`调用成功后，会自动失效参数 key 指定的 `/api/cards/${cardId}/keys` 的 SWR 缓存。

稍等，查询卡片时用的 key 是`/api/cards/${id}`，跟更新关键词时用的 key 不一样，这样没法刷新关键词列表`card.keys`呀。

有两个解决方案：

1. 更新时用的 key 也改成`/api/cards/${id}`。这样更新关键词后，卡片也会重新加载。
2. 把卡片查询拆分成两个查询：一个查询卡片，一个查询关键词列表。这样更新关键词后只会重新加载关键词列表。

方案 2 听起来更优化，我们加一个`KeysPage`组件管理关键词列表，试试这个方案：

```tsx
export function KeysPage({ cardId }: { cardId: string }) {
  const {
    data: keys,
    error,
    isLoading,
  } = useSWR(`/api/cards/${cardId}/keys`, keysFetcher);
  const [selectedKeyId, setSelectKeyId] = useState<string | null>(null);
  const keyword = !isLoading
    ? keys.find((key) => key.id === selectedKeyId)
    : undefined;
  const errorMsg = error ? getErrorMsg(error) : undefined;
  return (
    <>
      <Card id={cardId} keys={keys} setSelectKeyId={setSelectKeyId} />
      {keyword && <KeyForm keyword={keyword} />}
      {errorMsg && <Alert severity="error">{errorMsg}</Alert>}
    </>
  );
}
```

按之前类似的思路，分别判断是否有加载数据，以及是否有错误。
乍看之下还行，但是这里处理 error 有点问题。
如果关键词和卡片查询都失败了，这里会显示两个错误提示。
而我们只想显示一个卡片加载错误，像方案 1 那样，因为这是一个场景。

为了统一、简单的错误处理，我不得不回退到方案 1，绕开状态分支的问题。

## 'Static' Server Component

2020 年末 React 发布了 Server Component 的 [Demo](https://react.dev/blog/2020/12/21/data-fetching-with-react-server-components)，展示了一种新颖的 SSR 方式。
Server Component 是一种可以在服务端渲染的 React 组件。
你可以在服务端先查出不会修改的数据，渲染出部分页面，然后在客户端继续渲染剩下的部分。
这刚好能解决上面 API 查询的问题：我们可以在服务端查出卡片内容，然后在客户端查出关键词列表并渲染剩下的内容。

### 用服务端查询减少状态分支

我们用 [Next.js](https://nextjs.org/) 来实现这个 Server Component ：

```tsx
export async function Page({ params }: { params: { id: string } }) {
  const card = await prisma.card.findUnique({
    where: { id: params.id },
  });
  if (!card) {
    return notFound();
  }
  return <KeysPage card={card} />;
}
```

首先用 ORM 库 [Prisma](https://www.prisma.io/) 在服务端查询卡片数据。

1. 如果卡片不存在，我们返回 notFound 错误提示页面。
2. 如果卡片存在，我们返回`KeysPage`组件，这个组件会在客户端继续渲染。

这样卡片查询的错误处理提前到了服务端，我们不再需要处理两个可能重复的错误。
`error` 和 `isLoading` 状态由对应的[页面组件](https://nextjs.org/docs/app/building-your-application/routing/error-handling)来处理，减少了功能页面的状态分支。

### 在服务端查询所有数据

如果服务端一开始返回的这份数据，需要被用户修改，那这个数据和相关组件要怎么刷新呢？
[Next.js Server Action](https://nextjs.org/docs/app/building-your-application/data-fetching/forms-and-mutations) 可以解决这个问题。
它是一种能在客户端触发服务端执行的函数(@PHP)。
![use php](/assets/image/use-php.png)
我们可以用它实现更新关键词的功能：

```tsx
export function KeyForm({
  keyword,
}: {
  keyword: { id: string; meaning: string };
}) {
  async function serverAction(_prev: Response, formData: FormData) {
    "use server";
    const parsed = schema.safeParse(formData);
    if (!parsed.success) {
      return { error: MakeValidateError(parsed.error) };
    }
    const { id, meaning } = parsed.data;
    await prisma.keyword.update({
      where: { id },
      data: { meaning },
    });
    revalidatePath(`/cards/${cardId}`);
    return DefaultResponse();
  }
  const [state, action] = useFormState(serverAction, DefaultResponse());
  const { pending } = useFormStatus();
  const errorMsg = GetApiError(state)?.message;
  return (
    <form action={action}>
      <input type="hidden" name="id" value={keyword.id} />
      <textarea name="meaning" rows="4" defaultValue={keyword.meaning} />
      {errorMsg ? <Alert severity="error">{errorMsg}</Alert> : null}
    </form>
  );
}
```

这里的关键是用 `revalidatePath` 触发页面重新渲染，Next.js 称之为[Invalidate Router Cache](https://nextjs.org/docs/app/building-your-application/caching#invalidation-1)。

从这个代码片段来看，`Server Action`并没有比`useSWRMutation`少引入状态。
但是它的优势在于，我们现在可以去掉`KeysPage`组件里的`useSWR`，直接在服务端查询所有数据：

```tsx
export async function Page({ params }: { params: { id: string } }) {
  const card = await prisma.card.findUnique({
    where: { id: params.id },
    include: { keywords: true },
  });
  if (!card) {
    return notFound();
  }
  return <KeysPage card={card} />;
}
```

## 结语

Server Component 这种更线性化的编程方式，会更受 Web 后端开发者的青睐。
虽然它为此增加的自定义传输格式引入了新的复杂性，但带来的状态逻辑简化和性能提升，让人期待它的后续发展。
