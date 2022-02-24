# イントロスペクション

GraphQLは`__schema`という特別な組み込みトップレベルフィールドを定義しています。このフィールドを問い合わせることで、実行時に [スキーマをイントロスペクトする](https://graphql.org/learn/introspection/) ことができ、GraphQL サーバーがどのようなクエリや変異をサポートしているかを確認することができます。

イントロスペクションクエリは通常のGraphQLクエリーに過ぎないため、Juniperはネイティブでサポートしています。たとえば、サポートされているすべての型の名前を取得するには、Juniperに対して次のクエリを実行します。

```graphql
{
  __schema {
    types {
      name
    }
  }
}
```

## スキーマイントロスペクションのJSONでの出力

GraphQLエコシステムの多くのクライアントライブラリやツールは、サーバースキーマの完全な表現を必要とします。多くの場合、この表現は JSON であり、`schema.json` と呼ばれます。スキーマの完全な表現は、特別に細工された introspection クエリを発行することで作成できます。

Juniperはスキーマ全体をイントロスペクトする便利な関数を提供しています。結果はJSONに変換され、[graphql-client](https://github.com/graphql-rust/graphql-client)などのツールやライブラリで使用することができます。

```rust
# #![allow(unused_variables)]
# extern crate juniper;
# extern crate serde_json;
use juniper::{
    graphql_object, EmptyMutation, EmptySubscription, FieldResult, 
    GraphQLObject, IntrospectionFormat,
};

// スキーマを定義する.

#[derive(GraphQLObject)]
struct Example {
  id: String,
}

struct Context;
impl juniper::Context for Context {}

struct Query;

#[graphql_object(context = Context)]
impl Query {
   fn example(id: String) -> FieldResult<Example> {
       unimplemented!()
   }
}

type Schema = juniper::RootNode<
    'static, 
    Query, 
    EmptyMutation<Context>, 
    EmptySubscription<Context>
>;

fn main() {
    // コンテキストオブジェクトを作成する.
    let ctx = Context{};

    // 組み込みのイントロスペクション・クエリを実行する.
    let (res, _errors) = juniper::introspect(
        &Schema::new(Query, EmptyMutation::new(), EmptySubscription::new()),
        &ctx,
        IntrospectionFormat::default(),
    ).unwrap();

    // イントロスペクションの結果をjsonに変換する.
    let json_result = serde_json::to_string_pretty(&res);
    assert!(json_result.is_ok());
}
```