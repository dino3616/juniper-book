# オブジェクトの定義

Rust のどの型も GraphQL オブジェクトとして公開することができますが、最も一般的なのは構造体です。

JuniperでGraphQLオブジェクトを作成するには、2つの方法があります。
単純な構造体を公開したい場合、最も簡単な方法は、カスタムの`derive`属性があります。
もうひとつの方法は、[複雑なフィールド](complex_fields.md)の章で説明します。

```rust
# extern crate juniper;
# use juniper::GraphQLObject;
#[derive(GraphQLObject)]
struct Person {
    name: String,
    age: i32,
}
#
# fn main() {}
```

これは `Person` という名前の GraphQL オブジェクトタイプを作成し、2つのフィールドを持つようにします。`name` は `String!` 型で、`age` は `Int!` 型です。Rustの型システムにより、デフォルトではすべて非NULLでエクスポートされます。もし、NULLにできるフィールドが必要な場合は、 `Option<T>` を使用することができます。

GraphQLがセルフドキュメントであることを利用して、型やフィールドに説明を加えるべきでしょう。Juniperは、関連するdocコメントを自動的にGraphQLの記述として使用します。

```rust
# extern crate juniper;
# use juniper::GraphQLObject;
#[derive(GraphQLObject)]
/// 個人に関する情報
struct Person {
    /// 本人のフルネーム
    name: String,
    /// 年齢
    age: i32,
}
#
# fn main() {}
```

docコメントを持たないオブジェクトやフィールドは、代わりに `graphql` 属性で `description` を設定することができます。
以下の例は、上記と同等です。

```rust
# extern crate juniper;
# use juniper::GraphQLObject;
#[derive(GraphQLObject)]
#[graphql(description = "個人に関する情報")]
struct Person {
    #[graphql(description = "本人のフルネーム")]
    name: String,
    #[graphql(description = "年齢")]
    age: i32,
}
#
# fn main() {}
```

`graphql` 属性で設定した記述は、Rust のドキュメントコメントよりも優先されます。これにより、内部の Rust ドキュメントと外部の GraphQL ドキュメントを区別することができます。

```rust
# extern crate juniper;
# use juniper::GraphQLObject;
#[derive(GraphQLObject)]
#[graphql(description = "この記述は、GraphQLに表示される")]
/// この記述は、RustDocに表示される.
struct Person {
    #[graphql(description = "この記述は、GraphQLに表示される")]
    /// この記述は、RustDocに表示される.
    name: String,
    /// この記述は、RustDocとGraphQLの両方で表示される.
    age: i32,
}
#
# fn main() {}
```

## 関係性

カスタム`derive`属性は、以下のような状況下でのみ使用することができます。

- アノテーションされた型は `struct` である。
- すべての構造体フィールドは、以下のいずれかである。
  - プリミティブ型(`i32`, `f64`, `bool`, `String`, `juniper::ID`)
  - 有効なカスタムGraphQLタイプ(例えば、この属性でマークされた別の構造体)
  - 上記のいずれかを含むコンテナまたは参照(例えば、`Vec<T>`、`Box<T>`、`Option<T>` など)

オブジェクト間のリレーションシップ構築の意味を見てみましょう。

```rust
# extern crate juniper;
# use juniper::GraphQLObject;
#[derive(GraphQLObject)]
struct Person {
    name: String,
    age: i32,
}

#[derive(GraphQLObject)]
struct House {
    address: Option<String>, // 文字列に変換される(NULL可).
    inhabitants: Vec<Person>, // [Person!]に変換される.
}
#
# fn main() {}
```

`Person` は有効な GraphQL 型なので、構造体の中に `Vec<Person>` を入れても、自動的に NULL でない `Person` オブジェクトのリストに変換されます。

## フィールド名の変更

デフォルトでは、構造体フィールドはRustの標準的な命名規則である `snake_case` から、GraphQLの `camelCase` に変換されます。

```rust
# extern crate juniper;
# use juniper::GraphQLObject;
#[derive(GraphQLObject)]
struct Person {
    first_name: String, // GraphQLスキーマではfirstNameとして公開される.
    last_name: String, // lastName として公開される.
}
#
# fn main() {}
```

個々の構造体フィールドに `graphql` 属性を指定することで、名前をオーバーライドすることができます。

```rust
# extern crate juniper;
# use juniper::GraphQLObject;
#[derive(GraphQLObject)]
struct Person {
    name: String,
    age: i32,
    #[graphql(name = "websiteURL")]
    website_url: Option<String>, // スキーマで `websiteURL` として公開されるようになった.
}
#
# fn main() {}
```

または、構造体のすべてのフィールドに対して、異なるリネームポリシーを設定します。

```rust
# extern crate juniper;
# use juniper::GraphQLObject;
#[derive(GraphQLObject)]
#[graphql(rename_all = "none")] // リネームを無効にする.
struct Person {
    name: String,
    age: i32,
    website_url: Option<String>, // スキーマで `website_url` として公開されるようになった.
}
#
# fn main() {}
```

## 非推奨のフィールド

フィールドを非推奨にするには、`graphql` 属性で非推奨の理由を指定します。

```rust
# extern crate juniper;
# use juniper::GraphQLObject;
#[derive(GraphQLObject)]
struct Person {
    name: String,
    age: i32,
    #[graphql(deprecated = "代わりに名前フィールドを使用してください")]
    first_name: String,
}
#
# fn main() {}
```

`name`, `description`, `deprecation` の各引数は、もちろん組み合わせることができます。オブジェクトフィールドとenum値のみ非推奨とすることができるなど、GraphQL仕様の制限事項がまだ適用されます。

## フィールドを無視する

デフォルトでは、`GraphQLObject` のすべてのフィールドが生成される GraphQL 型に含まれます。特定のフィールドを含めないようにするには、そのフィールドに `#[graphql(ignore)]` というアノテーションを付けます。

```rust
# extern crate juniper;
# use juniper::GraphQLObject;
#[derive(GraphQLObject)]
struct Person {
    name: String,
    age: i32,
    #[graphql(ignore)]
    # #[allow(dead_code)]
    password_hash: String, // GraphQLからの問い合わせや変更はできない
}
#
# fn main() {}
```