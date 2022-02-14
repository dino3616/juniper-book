# クイックスタート

このページでは、Juniperのコンセプトを簡単に紹介します。

Juniperは、GraphQLのスキーマを定義する際に[コードファースト方式][schema_approach]に従っています。[スキーマファースト方式][schema_approach]を採用したい場合は、スキーマファイルからコードを生成する[juniper-from-schema][]を検討してください。

## インストール

```toml
[dependencies]
juniper = "0.15"
```

## スキーマの例

単純な列挙型や構造体をGraphQLとして公開するには、カスタムの`derive`属性を追加するだけでよいのです。Juniperでは、`Option<T>`、`Vec<T>`、`Box<T>`、`String`、`f64`、`i32`、参照、スライスなど、GraphQL機能に自然に対応するRustの基本型をサポートしています。

より高度なマッピングのために、Juniperは、Rust型をGraphQLスキーマにマッピングするための複数のマクロを提供しています。最も重要なのは [graphql_object][graphql_object] 手続き型マクロで、 `Query` と `Mutation` のルートで使用するリゾルバを持つオブジェクトを宣言するために使用されます。

```rust
# #![allow(unused_variables)]
# extern crate juniper;
# use std::fmt::Display;
use juniper::{
    graphql_object, EmptySubscription, FieldResult, GraphQLEnum, 
    GraphQLInputObject, GraphQLObject, ScalarValue,
};
#
# struct DatabasePool;
# impl DatabasePool {
#     fn get_connection(&self) -> FieldResult<DatabasePool> { Ok(DatabasePool) }
#     fn find_human(&self, _id: &str) -> FieldResult<Human> { Err("")? }
#     fn insert_human(&self, _human: &NewHuman) -> FieldResult<Human> { Err("")? }
# }

#[derive(GraphQLEnum)]
enum Episode {
    NewHope,
    Empire,
    Jedi,
}

#[derive(GraphQLObject)]
#[graphql(description = "スター・ウォーズの世界に登場する人物")]
struct Human {
    id: String,
    name: String,
    appears_in: Vec<Episode>,
    home_planet: String,
}

// GraphQLの入力オブジェクトをマッピングするためのカスタムderiveも用意されています.

#[derive(GraphQLInputObject)]
#[graphql(description = "スター・ウォーズの世界に登場する人物")]
struct NewHuman {
    name: String,
    appears_in: Vec<Episode>,
    home_planet: String,
}

// ここで、オブジェクトマクロを使用して、リゾルバを持つルートQueryとMutationの型を作成します.
// オブジェクトは、データベースプールのような共有状態にアクセスするためのコンテキストを持つことができます.

struct Context {
    // ここでは、実際のデータベースプールを使用してください.
    pool: DatabasePool,
}

// Juniperで使えるコンテキストにするために、マーカートレイトを実装する必要があります。
impl juniper::Context for Context {}

struct Query;

#[graphql_object(
    // ここでは、オブジェクトのコンテキストタイプを指定します.
    // コンテキストにアクセスする必要があるすべての型でこれを行う必要があります.
    context = Context,
)]
impl Query {
    fn apiVersion() -> &'static str {
        "1.0"
    }

    // リゾルバへの引数は、単純な型か入力オブジェクトのいずれかです.
    // コンテキストにアクセスするために、Context型への参照である引数を指定します.
    // Juniperは、ここで自動的に正しいコンテキストを注入します.
    fn human(context: &Context, id: String) -> FieldResult<Human> {
        // DB接続を取得します.
        let connection = context.pool.get_connection()?;
        // DBクエリを実行します.
        // エラーを伝播させるために `?` が使われていることに注意してください.
        let human = connection.find_human(&id)?;
        // 結果を返します.
        Ok(human)
    }
}

// 次に、Mutationタイプについても同じことを行います.

struct Mutation;

#[graphql_object(
    context = Context,
    // もし、オブジェクト定義のどこかで明示的に ScalarValue パラメトリを使用する必要がある場合 (ここでの FieldResult のように)、
    // そのための明示的な型パラメータを宣言して、それを指定すればよいでしょう.
    scalar = S: ScalarValue + Display,
)]
impl Mutation {
    fn createHuman<S: ScalarValue + Display>(context: &Context, new_human: NewHuman) -> FieldResult<Human, S> {
        let db = context.pool.get_connection().map_err(|e| e.map_scalar_value())?;
        let human: Human = db.insert_human(&new_human).map_err(|e| e.map_scalar_value())?;
        Ok(human)
    }
}

// ルートスキーマは、Query、Mutation、Subscriptionから構成されます.
// RootNodeに対して、リクエストクエリを実行することができます.
type Schema = juniper::RootNode<'static, Query, Mutation, EmptySubscription<Context>>;
#
# fn main() {
#   let _ = Schema::new(Query, Mutation{}, EmptySubscription::new());
# }
```

これで、GraphQLサーバーのための非常にシンプルで機能的なスキーマが完成しました。

実際にスキーマを提供するためには、私たちの様々な[サーバー統合](./servers/index.md)のガイドを参照してください。

Juniperは、サーバーを必要とせず、特定のトランスポートやシリアライズ形式にも依存しない、様々な文脈で利用できるライブラリです。クエリの結果を得るために、executorを直接呼び出すことができます。

## Executor

GraphQLクエリを実行するには、`juniper::execute`を直接呼び出すことができます。

```rust
# // マクロにアクセスできないため、2018年版のため必要なだけです.
# #[macro_use] extern crate juniper;
use juniper::{
    graphql_object, EmptyMutation, EmptySubscription, FieldResult, 
    GraphQLEnum, Variables, graphql_value,
};

#[derive(GraphQLEnum, Clone, Copy)]
enum Episode {
    NewHope,
    Empire,
    Jedi,
}

// 任意のコンテキストデータ.
struct Ctx(Episode);

impl juniper::Context for Ctx {}

struct Query;

#[graphql_object(context = Ctx)]
impl Query {
    fn favoriteEpisode(context: &Ctx) -> FieldResult<Episode> {
        Ok(context.0)
    }
}

// ルートスキーマは、Query、Mutation、Subscriptionから構成されます.
// RootNodeに対して、リクエストクエリを実行することができます.
type Schema = juniper::RootNode<'static, Query, EmptyMutation<Ctx>, EmptySubscription<Ctx>>;

fn main() {
    // コンテキストオブジェクトを作成します.
    let ctx = Ctx(Episode::NewHope);

    // executorを実行します.
    let (res, _errors) = juniper::execute_sync(
        "query { favoriteEpisode }",
        None,
        &Schema::new(Query, EmptyMutation::new(), EmptySubscription::new()),
        &Variables::new(),
        &ctx,
    ).unwrap();

    // 値が一致していることを確認します.
    assert_eq!(
        res,
        graphql_value!({
            "favoriteEpisode": "NEW_HOPE",
        })
    );
}
```

[juniper-from-schema]: https://github.com/davidpdrsn/juniper-from-schema
[schema_approach]: https://blog.logrocket.com/code-first-vs-schema-first-development-graphql/
[hyper]: servers/hyper.md
[warp]: servers/warp.md
[rocket]: servers/rocket.md
[iron]: servers/iron.md
[tutorial]: ./tutorial.html
[graphql_object]: https://docs.rs/juniper/latest/juniper/macro.graphql_object.html