# サブスクリプション

### GraphQLサブスクリプションでリアルタイムデータを実現する方法

GraphQLサブスクリプションは、サーバーからリアルタイムのメッセージを要求するクライアントに対して、サーバーからデータをプッシュする方法です。サブスクリプションは、クライアントに配信するフィールドのセットを指定するという点でクエリーと似ていますが、1つの回答をすぐに返すのではなく、サーバーで特定のイベントが発生するたびに結果が送信されます。

サブスクリプションを実行するには、コーディネーター（接続を生成するもの）と、ストリームに解決できるGraphQLオブジェクトが必要です。[`juniper_subscriptions`][juniper_subscriptions]クレートは、デフォルトの接続実装を提供します。現在、サブスクリプションは `master` ブランチでのみサポートされています。以下を `Cargo.toml` に追加してください。

```toml
[dependencies]
juniper = { git = "https://github.com/graphql-rust/juniper", branch = "master" }
juniper_subscriptions = { git = "https://github.com/graphql-rust/juniper", branch = "master" }
```

### スキーマ定義

`Subscription` は単なる GraphQL オブジェクトで、[Schema][Schema] でオペレーション用に定義したクエリールートやミューテーションオブジェクトと同じものです。サブスクリプションでは、すべてのフィールド/オペレーションは非同期でなければならず、[Stream][Stream]を返す必要があります。

この例では、2つのイベント、文字列 `Hello` と `World!` を順次返すサブスクリプションオペレーションを示しています。

```rust
# use juniper::{graphql_object, graphql_subscription, FieldError};
# use futures::Stream;
# use std::pin::Pin;
#
# #[derive(Clone)]
# pub struct Database;
# impl juniper::Context for Database {}

# pub struct Query;
# #[graphql_object(context = Database)]
# impl Query {
#    fn hello_world() -> &'static str {
#        "Hello World!"
#    }
# }
pub struct Subscription;

type StringStream = Pin<Box<dyn Stream<Item = Result<String, FieldError>> + Send>>;

#[graphql_subscription(context = Database)]
impl Subscription {
    async fn hello_world() -> StringStream {
        let stream = futures::stream::iter(vec![
            Ok(String::from("Hello")),
            Ok(String::from("World!"))
        ]);
        Box::pin(stream)
    }
}
#
# fn main () {}
```

### コーディネーター

サブスクリプションは、通常のクエリよりも少し多くのリソースを必要とし、DOS攻撃の大きなベクトルを提供します。これは、正しく処理されないと、サーバを簡単にダウンさせることができます。[`SubscriptionCoordinator`][SubscriptionCoordinator] トレイトは、DOS 攻撃の緩和やリソース制限などの機能を有効にするための調整ロジックを提供します。

[`SubscriptionCoordinator`][SubscriptionCoordinator] はスキーマを含み、開いた接続を追跡し、サブスクリプションの開始と終了を処理し、それぞれのサブスクリプションに対してグローバルなサブスクリプション ID を維持することができます。接続が確立されるたびに、[`SubscriptionCoordinator`][SubscriptionCoordinator] は [`SubscriptionConnection`][SubscriptionConnection] を生成します。[`SubscriptionConnection`][SubscriptionConnection] は単一の接続を処理し、クライアントストリーム用のリゾルバロジックと再接続およびシャットダウンロジックを提供します。

[`SubscriptionCoordinator`][SubscriptionCoordinator] は自分で実装することもできますが、Juniper には [`Coordinator`][Coordinator] というシンプルで汎用的な実装が用意されています。ここで、[`Connection`][Connection] は操作によって返される値の `Stream` で、[`GraphQLError`][GraphQLError] は購読が失敗した場合のエラーです。

```rust
# #![allow(dead_code)]
# extern crate futures;
# extern crate juniper;
# extern crate juniper_subscriptions;
# extern crate serde_json;
# extern crate tokio;
# use juniper::{
#     http::GraphQLRequest,
#     graphql_object, graphql_subscription, 
#     DefaultScalarValue, EmptyMutation, FieldError, 
#     RootNode, SubscriptionCoordinator,
# };
# use juniper_subscriptions::Coordinator;
# use futures::{Stream, StreamExt};
# use std::pin::Pin;
# 
# #[derive(Clone)]
# pub struct Database;
# 
# impl juniper::Context for Database {}
# 
# impl Database {
#     fn new() -> Self {
#         Self {}
#     }
# }
# 
# pub struct Query;
# 
# #[graphql_object(context = Database)]
# impl Query {
#     fn hello_world() -> &'static str {
#         "Hello World!"
#     }
# }
# 
# pub struct Subscription;
# 
# type StringStream = Pin<Box<dyn Stream<Item = Result<String, FieldError>> + Send>>;
# 
# #[graphql_subscription(context = Database)]
# impl Subscription {
#     async fn hello_world() -> StringStream {
#         let stream =
#             futures::stream::iter(vec![Ok(String::from("Hello")), Ok(String::from("World!"))]);
#         Box::pin(stream)
#     }
# }
type Schema = RootNode<'static, Query, EmptyMutation<Database>, Subscription>;

fn schema() -> Schema {
    Schema::new(Query {}, EmptyMutation::new(), Subscription {})
}

async fn run_subscription() {
    let schema = schema();
    let coordinator = Coordinator::new(schema);
    let req: GraphQLRequest<DefaultScalarValue> = serde_json::from_str(
        r#"{
            "query": "subscription { helloWorld }"
        }"#,
    )
        .unwrap();
    let ctx = Database::new();
    let mut conn = coordinator.subscribe(&req, &ctx).await.unwrap();
    while let Some(result) = conn.next().await {
        println!("{}", serde_json::to_string(&result).unwrap());
    }
}
#
# fn main() { }
```

### Web統合とその事例

現在、[warp][warp]を使ったサブスクリプションの例がありますが、まだαの状態です。
[WS][WS]上のGraphQLはまだ完全にサポートされておらず、非標準的です。

* [Warp Subscription Example](https://github.com/graphql-rust/juniper/tree/master/examples/warp_subscriptions)
* [Example](https://github.com/graphql-rust/juniper/tree/master/examples/basic_subscriptions)

[juniper_subscriptions]: https://github.com/graphql-rust/juniper/tree/master/juniper_subscriptions
[Stream]: https://docs.rs/futures/0.3.4/futures/stream/trait.Stream.html
 <!-- TODO: juniper_subscriptionsのドキュメントが定義されている場合、これらのリンクを修正する. --->
[Coordinator]: https://docs.rs/juniper_subscriptions/0.15.0/struct.Coordinator.html
[SubscriptionCoordinator]: https://docs.rs/juniper_subscriptions/0.15.0/trait.SubscriptionCoordinator.html
[Connection]: https://docs.rs/juniper_subscriptions/0.15.0/struct.Connection.html
[SubscriptionConnection]: https://docs.rs/juniper_subscriptions/0.15.0/trait.SubscriptionConnection.html
<!--- --->
[Future]: https://docs.rs/futures/0.3.4/futures/future/trait.Future.html
[warp]: https://github.com/graphql-rust/juniper/tree/master/juniper_warp
[WS]: https://github.com/apollographql/subscriptions-transport-ws/blob/master/PROTOCOL.md
[GraphQLError]: https://docs.rs/juniper/0.14.2/juniper/enum.GraphQLError.html
[Schema]: ../schema/schemas_and_mutations.md