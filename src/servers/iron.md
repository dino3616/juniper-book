# Iron

[Iron]は、Rust界隈でしばらく前からあるライブラリですが、最近はあまり開発されていません。とはいえ、Rust の安定版チャンネルで動作する、おなじみのリクエスト/レスポンス/ミドルウェアのアーキテクチャを持つ堅実なライブラリであることに変わりはありません。

JuniperのIronの統合は `juniper_iron` クレートに含まれています。

```toml
[dependencies]
juniper = "0.15.7"
juniper_iron = "0.7.4"
```

ソースには、基本的な GraphQL と [GraphiQL] ハンドラをセットアップする [example](https://github.com/graphql-rust/juniper_iron/blob/master/examples/iron_server.rs) が含まれています。

## 基本的な統合

まずは最小限のスキーマで、GraphQLエンドポイントだけを立ち上げてみましょう。mount]を使用して、`/graphql`にGraphQLハンドラをアタッチします。

context_factory` 関数はリクエストごとに実行され、データベース接続のセットアップ、クッキーからのセッショントークン情報の読み取り、スキーマが必要とするその他のグローバルデータのセットアップに使用されます。

この例では、グローバルデータを使用しないので、単に空の値を返します。

```rust,ignore
extern crate juniper;
extern crate juniper_iron;
extern crate iron;
extern crate mount;

use mount::Mount;
use iron::prelude::*;
use juniper::EmptyMutation;
use juniper_iron::GraphQLHandler;

fn context_factory(_: &mut Request) -> IronResult<()> {
    Ok(())
}

struct Root;

#[juniper::graphql_object]
impl Root {
    fn foo() -> String {
        "Bar".to_owned()
    }
}

# #[allow(unreachable_code, unused_variables)]
fn main() {
    let mut mount = Mount::new();

    let graphql_endpoint = GraphQLHandler::new(
        context_factory,
        Root,
        EmptyMutation::<()>::new(),
    );

    mount.mount("/graphql", graphql_endpoint);

    let chain = Chain::new(mount);

#   return;
    Iron::new(chain).http("0.0.0.0:8080").unwrap();
}
```

## リクエストからデータにアクセスする

フィールドリゾルバからリクエストのソースIPアドレスなどにアクセスしたい場合は、Juniperの[コンテキスト機能](../types/objects/using_contexts.md)を使用してこのデータを渡す必要があります。

```rust,ignore
# extern crate juniper;
# extern crate juniper_iron;
# extern crate iron;
# use iron::prelude::*;
use std::net::SocketAddr;

struct Context {
    remote_addr: SocketAddr,
}

impl juniper::Context for Context {}

fn context_factory(req: &mut Request) -> IronResult<Context> {
    Ok(Context {
        remote_addr: req.remote_addr
    })
}

struct Root;

#[juniper::graphql_object(
    Context = Context,
)]
impl Root {
    field my_addr(context: &Context) -> String {
        format!("Hello, you're coming from {}", context.remote_addr)
    }
}

# fn main() {
#     let _graphql_endpoint = juniper_iron::GraphQLHandler::new(
#         context_factory,
#         Root,
#         juniper::EmptyMutation::<Context>::new(),
#     );
# }
```

[iron]: https://github.com/iron/iron
[graphiql]: https://github.com/graphql/graphiql
[mount]: https://github.com/iron/mount