# スキーマとミューテーション

Juniperは、GraphQLのスキーマを定義する際、[コードファースト方式][schema_approach] に従っています。[スキーマファースト方式][schema_approach]を採用したい場合は、スキーマファイルからコードを生成する[juniper-from-schema][]を検討してみてください。

スキーマは、クエリオブジェクト、ミューテーションオブジェクト、サブスクリプションオブジェクトの3つのタイプから構成されています。
この3つはそれぞれ、スキーマのルート・クエリー・フィールド、ミューテーション、サブスクリプションを定義します。

サブスクリプションの使い方は、ミューテーションやクエリオブジェクトと少し異なるので、それらについて説明する特定の[セクション][セクション]が用意されています。

クエリーとミューテーションの両オブジェクトは、通常のGraphQLオブジェクトで、Juniperの他のオブジェクトと同様に定義されます。ただし、スキーマは読み取り専用にすることができ、サブスクリプションを必要としないため、MutationオブジェクトとSubscriptionオブジェクトはオプションとなります。ミューテーション／サブスクリプション機能が不要な場合は、[EmptyMutation][EmptyMutation]／[EmptySubscription][EmptySubscription]の利用を検討してください。

Juniper では、`RootNode` 型はスキーマを表します。スキーマが最初に作成されると、Juniperはオブジェクトグラフ全体をトラバースして、見つけられたすべてのタイプを登録します。つまり、どこかでGraphQLオブジェクトを定義しても、それを参照することがなければ、スキーマで公開されることはないのです。

## クエリールート

クエリルートは単なるGraphQLオブジェクトです。Juniper の他の GraphQL オブジェクトと同じように、 `graphql_object` 手続き型マクロを使用して定義します。

```rust
# #![allow(unused_variables)]
# extern crate juniper;
# use juniper::{graphql_object, FieldResult, GraphQLObject};
# #[derive(GraphQLObject)] struct User { name: String }
struct Root;

#[graphql_object]
impl Root {
    fn userWithUsername(username: String) -> FieldResult<Option<User>> {
        // データベースでユーザーを検索する...
#       unimplemented!()
    }
}
#
# fn main() { }
```

## ミューテーション

ミューテーションは単なるGraphQLオブジェクトです。各変異は、データベースを更新するような変異の副作用を実行する単一のフィールドです。

```rust
# #![allow(unused_variables)]
# extern crate juniper;
# use juniper::{graphql_object, FieldResult, GraphQLObject};
# #[derive(GraphQLObject)] struct User { name: String }
struct Mutations;

#[graphql_object]
impl Mutations {
    fn signUpUser(name: String, email: String) -> FieldResult<User> {
        // 入力を検証し、ユーザーをデータベースに保存する...
#       unimplemented!()
    }
}
#
# fn main() { }
```

# Rust スキーマを [GraphQL Schema Language][schema_language] に変換する

GraphQLエコシステムの多くのツールは、スキーマを[GraphQL Schema Language][schema_language]で定義することを要求しています。Rust で定義したスキーマを [GraphQL Schema Language][schema_language] で表現したものを `schema-language` 機能で生成できます（デフォルトでオンになっています）。

```rust
# extern crate juniper;
use juniper::{
    graphql_object, EmptyMutation, EmptySubscription, FieldResult, RootNode,
};

struct Query;

#[graphql_object]
impl Query {
    fn hello(&self) -> FieldResult<&str> {
        Ok("hello world")
    }
}

fn main() {
    // Rustでスキーマを定義する.
    let schema = RootNode::new(
        Query,
        EmptyMutation::<()>::new(),
        EmptySubscription::<()>::new(),
    );

    // Rust スキーマを GraphQL Schema Language に変換します.
    let result = schema.as_schema_language();

    let expected = "\
type Query {
  hello: String!
}

schema {
  query: Query
}
";
    assert_eq!(result, expected);
}
```

依存関係を減らしコンパイル時間を短縮するために、この機能が必要ない場合は`schema-language`機能をオフにすることができることに注意してください。

[schema_language]: https://graphql.org/learn/schema/#type-language
[juniper-from-schema]: https://github.com/davidpdrsn/juniper-from-schema
[schema_approach]: https://blog.logrocket.com/code-first-vs-schema-first-development-graphql/
[section]: ../advanced/subscriptions.md
[EmptyMutation]: https://docs.rs/juniper/0.14.2/juniper/struct.EmptyMutation.html
<!-- TODO: EmptySubscriptionがドキュメントで利用可能になったときにこのURLを修正する. -->
[EmptySubscription]: https://docs.rs/juniper/0.14.2/juniper/struct.EmptySubscription.html