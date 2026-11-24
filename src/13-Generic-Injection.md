# 13 Generic Injection

## 13.1 基础数据结构

在正式介绍`bevy_reflect`和`bevy_ecs`的底层之前，我们还有需要额外的补充一点说明。

在`bevy_ecs`中你将会看到这种，利用纯粹的rust泛型支持来实现依赖注入的技巧，这种实现依赖注入的方式完全依赖rust强大的泛型系统而无需过程宏。

让我们以一个`http`框架的依赖注入系统为例子，来解释一下该如何利用纯粹的泛型实现依赖注入功能。

假设我们的`Request`和`Response`结构体长这个样子，这是一个最简单的情况之一。

```rust
// 这是我们的Request结构体，包含body和Parts，Parts包含了http的请求头
pub struct Parts { pub headers: Vec<String> }
pub struct Body(pub String);
pub struct Request { parts: Parts, body: Body }

impl Request {
    fn new(headers: Vec<String>, body: &str) -> Self {
        Request { parts: Parts { headers }, body: Body(body.to_owned()) }
    }
    fn into_parts(self) -> (Parts, Body) { (self.parts, self.body) }
    fn from_parts(parts: Parts, body: Body) -> Self { Request { parts, body } }
}


// 下面是Response结构体以及IntoResponse trait
// 我们为常见的类型String、&'static str、Result等实现了IntoResponse trait，这样我们就可以在返回这些类型时自动转换为我们的Response结构体
pub struct Response(pub String);

pub trait IntoResponse { 
    fn into_response(self) -> Response; 
}
impl IntoResponse for Response { 
    fn into_response(self) -> Response { self } 
}
impl IntoResponse for String { 
    fn into_response(self) -> Response { Response(self) } 
}
impl IntoResponse for &'static str {
    fn into_response(self) -> Response { Response(self.to_owned()) }
}
impl<T: IntoResponse, E: IntoResponse> IntoResponse for Result<T, E> {
    fn into_response(self) -> Response {
        match self {
            Ok(v) => v.into_response(),
            Err(e) => e.into_response(),
        }
    }
}
```

这些代码简单到不言而喻，除了一点——为什么我们要区分`Body`和`Parts`?

这是因为，`Body`和`Parts`在一个`http`请求的里的地位就是不同的，`Body`是一个流，而`Parts`是一个完整的请求头。一个流是无法被重复读取和解析的，因为流不负责保存之前的数据，这就像“滚滚长江东逝水”，长江只负责让水流过去而不负责记录流过去的水都有哪些。用rust里的话来讲，**`body`被消费一次后就不应该再被使用了，也就是所有权转移了。**

在其他语言的框架里，当你二次消费body时，要么会发生异常，要么会发生请求挂起。而在稍后你就可以看，只需要稍加利用rust的所有权系统，就能让rust编译器为我们保证这件事。

---

现在让我们来继续设计下去，对于类似`Header`或者`Query`或者`Cookies`之类的数据，我们应该从`Parts`中获得，对于`Body`我们则经常以`Json`的形式来在前端发送并在后端反序列化，同时除了来自http的数据，我们还有本地的一些应给被共享的`State`，例如数据库。

于是我们可以顺理成章的写出来`FromRequestParts`和`FromRequest`两个`trait`，其中前者用于从`Parts`里转换数据，**后者则消费掉整个`Request`，从`Body`中构造出我们需要的数据。**这样，rust编译器就会替我们保证`body`只会被消费一次了。

```rust
// 一个泛型trait，从Parts中提取出自己
pub trait FromRequestParts<S>: Sized {
    fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, String>;
}

// 一个泛型trait，消耗整个Request并提取出自己
pub trait FromRequest<S>: Sized {
    fn from_request(req: Request, state: &S) -> Result<Self, String>;
}

// 这是我们的公共数据中心，假设里面有来自sqlx的PgPool，这是一个数据库连接池，是一个Arc的共享指针
#[derive(Clone)]
pub struct AppState { pub db: PgPool }

// 从Parts中提取出Header
pub struct Header(pub String);
impl<S> FromRequestParts<S> for Header {
    fn from_request_parts(parts: &mut Parts, _s: &S) -> Result<Self, String> {
        // 在这里使用parts来构造一个Header结构体，具体的提取代码我们在此处省略
    }
}


// 从Parts中提取出Query
pub struct Query(pub String);
impl<S> FromRequestParts<S> for Query {
    fn from_request_parts(parts: &mut Parts, _s: &S) -> Result<Self, String> {
        // 在这里使用parts来构造一个Query结构体，具体的提取代码我们在此处省略
    }
}

// 从Body中提取出JsonBody
pub struct JsonBody(pub String);
impl<S> FromRequest<S> for JsonBody {
    fn from_request(req: Request, _s: &S) -> Result<Self, String> {
       // 注意这里我们的直接消费了整个Request
       Ok(JsonBody(req.body.0))
    }
}
    
pub struct Db(pub PgPool);
// 从我们的AppState中提取出pgpool
impl FromRequestParts<AppState> for Db {
    fn from_request_parts(_p: &mut Parts, s: &AppState) -> Result<Self, String> {
        Ok(Db(s.db.clone()))
    }
}
```

## 13.2 单参数注入

看起来好像一切都很好，假如我们能够用某种方法实现依赖注入的话，我们好像就可以让下面的这样的函数能被自动调用和转换响应，那么我们该怎么办呢？

```rust
async fn two_args(h: Header, db: Db)->impl IntoResponse;
```

---

要实现这个魔法的核心关键是，**我们需要找到某种办法，让rust的泛型系统能够根据参数所实现的trait是`FromRequestParts`还是`FromRequest`来自动生成正确的调用的代码。**

为了实现这个目的，我们尝试着来写一下。

首先可以想到的是，我们应该为某些`FnOnce`类型实现一个自定义的`trait`，然后通过这个`trait`取代`FnOnce`的`call`方法，这样我们可以解决第一个问题：怎么将`FnOnce`五花八门的参数来统一成`Request`和我们的`AppState`。

```rust
pub type BoxFuture = Pin<Box<dyn Future<Output = Response> + Send>>;

// 之后我们就通过我们的Handler的call方法来调用FnOnce
pub trait Handler<S>: Clone + Send + 'static {
    fn call(self, req: Request, state: S) -> BoxFuture;
}
```

我们需要玩一点有意思的泛型魔法，这也是我们的依赖注入中最核心的一步。

让我们来降低一下难度，我们**先考虑对于只接受一个参数**并且这个参数消费了`Request`的`http handler`函数，我们的这个函数应该长什么什么样子？

显然这个`FnOnce`的特点是：

- 接受的参数只有一个
- 这唯一的一个参数实现了`FromRequest`
- 该`FnOnce`返回一个`Future`
- 这个`Future`的`Output`实现了`IntoResponse`。

对于这四个约束，再加上我们的我们`AppState`的约束，于是我们可以写出来一个具有五个泛型参数的签名。

```rust
// 这些可真长，前四个泛型参数对应了我们的四个要求，最后一个S是对于我们的AppState的基本要求。
impl<F, T1, Fut, Res, S> Handler<S> for F
where
    F: FnOnce(T1,) -> Fut + Clone + Send + 'static,
    Fut: Future<Output = Res> + Send,
    Res: IntoResponse,
    T1: FromRequest<S> + Send,
    S: Send + Sync + 'static,
{
	fn call(self, req: Request, state: S) -> BoxFuture {
        Box::pin(async move {
            // 我们直接调用from_request并传递给这个函数自己
            let t1 = match T1::from_request(req, &state) {
                Ok(v) => v,
                Err(e) => return Response(format!("400: {e}")),
            };
            self(t1,).await.into_response()
        })
    }
}
```

这看着挺合理的，但是编译器上来就给了我们一巴掌：**the type parameter `T1` is not constrained by the impl trait,self type, or predicates**。

啊，rust编译器多么令人讨厌啊！

这是编译器在告诉我们，T1 决定了这份 impl 在描述哪一种函数，允许它只藏在 where 里，同一个 F 就可能对应好几个 T1，编译器没法选，要求我们强制把这个参数写到Handler中。

于是我们可以这样稍微修改一下我们的代码，在Handler中添加一个参数，虽然我们根本没有用到这个参数，但是为了让编译器满意我们不得不这样做。

```rust
pub trait Handler<T1, S>: Clone + Send + 'static {
    fn call(self, req: Request, state: S) -> BoxFuture;
}

impl<F, T1, Fut, Res, S> Handler<T1, S> for F // 后面的完全不变
```

---

让人满意！让我们来更进一步，`Header`、`Query`、`Db`实现的`FromRequestParts`，我们现在只有对于`FromRequest`的实现，那我们很自然会想——再写一份 impl 不就完了？于是我们写出来了这样的东西。

```rust
impl<F, T1, Fut, Res, S> Handler<T1, S> for F where T1: FromRequest<S> { ... }
impl<F, T1, Fut, Res, S> Handler<T1, S> for F where T1: FromRequestParts<S> { ... }
```

直觉上看是可以的，但是rust编译器会当场拒绝你，并说 **conflicting implementations of trait `Handler<_, _>`！**

这是为什么？因为我们违背了coherence 即一致性规则：对任意一个类型，一个trait最多只能有一个实现。 编译器在任何时候看到 x.foo()，都必须能唯一确定该调哪份代码，不能有二义。

rust通过两个规则来维持这个确定性，其一是**孤儿规则**，这个是大家耳熟能详的，即对于`impl Trait for Type` 这样的代码，必须至少有一方是在当前 crate 定义的；另一个则是**重叠检查**，**即两份 impl 会不会命中同一个类型。**

编译器看到这些签名的时候，根本没看我们的 where 子句，它看的是`<F, T1, Fut, Res, S>`和`Handler<T1, S>`，这两份身份证完全一样，因此rust编译器认为这就是同一个 impl 的两个副本，这违背了**重叠检查**规则！

---

这可麻烦了，**既然这条路走不通，那我们只能想办法用一份实现来兼容两种trait了**，这该怎么办呢？

聪明的你肯定可以想到，只要我们要求让实现了 `FromRequestParts` 的东西同样也实现了 `FromRequest` 不就行了。只要我们手动为`FromRequest`实现一个泛型方法即可，这样，对于接受一个参数的`handler`我们应该就都能够兼容了。

```rust
impl<S, T> FromRequest<S> for T
where
    T: FromRequestParts<S>,
    S: Send + Sync,
{
    fn from_request(req: Request, s: &S) -> Result<Self, String> {
        let (mut parts, _body) = req.into_parts();
        T::from_request_parts(&mut parts, s)
    }
}
```

一切看起来都让人满意了，但是当我们使用的时候，rust编译器又再次站出来给了我们一巴掌，他说：

```tex
error[E0119]: conflicting implementations of trait `FromRequest<AppState>`
  |
  | impl<S, T> FromRequest<S> for T where T: FromRequestParts<S>
  | ------------------------------------------ first implementation here
  | impl FromRequest<AppState> for JsonBody
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ conflicting implementation
```

这不讲道理啊，`JsonBody` 明明没实现 `FromRequestParts` 啊？为什么rust编译器会觉得这里有两份`FromRequest`的实现呢？

**这是因为编译器的推理认为，`JsonBody` 虽然现在没实现`FromRequestParts`，但将来可能实现。我证明不了它不重叠，所以我拒绝编译。**

这下又麻烦了，我们该怎么做才能对编译器保证`JsonBody`绝对不会实现`FromRequestParts`呢？

要实现这个目的我们需要一点小技巧，我们需要引入一个用于标记的泛型参数。这个泛型参数对于具体的实现而言没有意义，它存在的目的只是为了告诉rust编译器，这两个实现不是同一个实现。

```rust
mod private {
    pub struct ViaParts;    // 第一组：只读 Parts
    pub struct ViaRequest;  // 第二组：要吃整个 Request
}

// 我们给FromRequest增加一个标记M参数，这个参数只是在签名中出现
pub trait FromRequest<S, M = private::ViaRequest>: Sized {
    fn from_request(req: Request, state: &S) -> Result<Self, String>;
}


// 之前的用于给所有FromRequestParts实现FromRequest的代码，我们用private::ViaParts来标记
impl<S, T> FromRequest<S, private::ViaParts> for T
where T: FromRequestParts<S> { ... }              // 第一组

// 对于JsonBody我们使用private::ViaRequest来标记
impl FromRequest<AppState> for JsonBody { ... } 
```

---

一旦我们引入了`M`，我们的`handler`的签名同样也要进行修改了。同样这个`M`我们也要放进`Handler`的参数里，否则`M`又成了那个只活在 `where` 子句里的孤儿。

第二个看起来没用的参数`Handler<(M, T1,), S>` 里的 `T`（那个元组）从哪来的呢？它就是一块给M和T1准备的合法席位，运行时没人读它。

为什么我们要写成元组`(M, T1)`而不是 `Handler<M, T1, S>` 三个平铺参数呢？明明在单参数时看起来两者是等价的。

因为下一节要一路支持到最多十三个参数，到那时`M`和 `T1..Tn`打包成元组最省事，`M` 就顺手蹲在第 0 位。

## 13.3 多参数注入

处理好了单个参数，我们就可以顺理成章的拓展到最多能达到16个参数注入的`handler`了。

这里我们以2个参数的handler为例。顺着单参数的写法，我们可以先写出来第一版的两个参数的handler版本。

```rust
impl<F, T1, T2, Fut, Res, S, M> Handler<(M, T1, T2,), S> for F
where
    F: FnOnce(T1, T2,) -> Fut + Clone + Send + 'static,
    Fut: Future<Output = Res> + Send,
    Res: IntoResponse,
    T1: FromRequest<S, M> + Send,   // ← 照抄
    T2: FromRequest<S, M> + Send,   // ← 照抄
    S: Send + Sync + 'static,
{
    fn call(self, req: Request, state: S) -> BoxFuture {
        Box::pin(async move {
            let t1 = match T1::from_request(req, &state) {
                Ok(v) => v,
                Err(e) => return Response(format!("400: {e}")),
            };
            let t2 = match T2::from_request(req, &state) {   // ← 
                Ok(v) => v,
                Err(e) => return Response(format!("400: {e}")),
            };
            self(t1, t2,).await.into_response()
        })
    }
}
```

但是一到`async fn two_args(h: Header, db: Db)->impl IntoResponse;`的时候，就会遇到最让人讨厌的借用检查问题。

```tex
error[E0382]: use of moved value: `req`
   |
   |     let t1 = T1::from_request(req, &state)?;
   |                              --- value moved here
   |     let t2 = T2::from_request(req, &state)?;
   |                              ^^^ value used here after move
```

要解决这个小问题，只需要我们稍微更改一下签名。

```rust
impl<F, T1, T2, Fut, Res, S, M> Handler<(M, T1, T2,), S> for F
where
    F: FnOnce(T1, T2,) -> Fut + Clone + Send + 'static,
    Fut: Future<Output = Res> + Send,
    Res: IntoResponse,
    T1: FromRequestParts<S> + Send,       // ← 只读 Parts
    T2: FromRequest<S, M> + Send,         // ← 唯一可以吃 body 的
    S: Send + Sync + 'static,
{
    fn call(self, req: Request, state: S) -> BoxFuture {
        Box::pin(async move {
            // 把 request 拆成"能借的"和"能移动的"
            let (mut parts, body) = req.into_parts();

            // 前面的参数：只借 &mut parts，body 原地寄存
            let t1 = match T1::from_request_parts(&mut parts, &state) {
                Ok(v) => v,
                Err(e) => return Response(format!("400: {e}")),
            };

            // 把 body 原封不动还回去，重新组成一个 Request
            let req = Request::from_parts(parts, body);

            // 最后一个参数：唯一一次 consume
            let t2 = match T2::from_request(req, &state) {
                Ok(v) => v,
                Err(e) => return Response(format!("400: {e}")),
            };

            self(t1, t2,).await.into_response()
        })
    }
}
```

这里必须解释清楚 `from_parts` 那一步，`into_parts` 把 `request` 拆成两半之后，`body` 就寄存在局部变量里了。它既不能借给 t1（`T1` 的签名里没有 body），也不能凭空消失（后面 t2 还要）。`from_parts`是把 body 从寄存处取回来、交还给唯一那个有权消费它的人的唯一方式。

对于3-16个参数的依赖注入，和这些几乎别无二致，无非就是继续增加T的个数罢了，这里就不再赘述了。

## 13.4 总结

这一节实际上颇有难度，读者可能需要花点时间才能消化这些内容，因此觉得这些东西比较困难是很正常的，不必灰心。上面我们给出的这些，正是`axum`框架的核心机制的最精简的版本，从这些代码里我们也同样可以理解，为什么`axum`框架的使用文档里要强调body的提取器必须放最后这件事了。

回过头来，我们花了整整一节，用 `http` 框架做例子，从 `Handler<S>` 一路推到 `Handler<(M, T1, ..., Tn), S>`。你可能觉得这是在讲别人的故事，毕竟这本书要讲的是 `Bevy`。但其实我们刚刚推导出来的，不是axum 的一个技巧，而是 rust 里做**变长参数 + 泛型注入**的标准解法。

`Bevy` 的 `bevy_ecs` 用的是同一个骨架，在这里我们提前将这种另一种形式的反射补充在这里。





