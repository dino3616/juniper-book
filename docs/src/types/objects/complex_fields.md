# 複雑なフィールド

もし、計算フィールドや循環構造を含む構造体を直接GraphQLにマッピングできない場合は、より強力なツールである `#[graphql_object]` 手続き型マクロを使用する必要があります。このマクロは、Rust の `impl` ブロックで GraphQL のオブジェクトフィールドを定義することができます。
GraphQL フィールドはデフォルトでこの `impl` ブロックに定義されることに注意してください。もし、構造体に通常のメソッドを定義したい場合は、別の `impl` ブロックで定義するか、 `#[graphql(ignore)]` 属性でマークして、マクロで省略できるようにしなければなりません。
前章の例の続きで、マクロを使用して `Person` を定義する方法を説明します。

```rust
# #![allow(dead_code)]
# extern crate juniper;
# use juniper::graphql_object;
#
struct Person {
    name: String,
    age: i32,
}

#[graphql_object]
impl Person {
    fn name(&self) -> &str {
        self.name.as_str()
    }

    fn age(&self) -> i32 {
        self.age
    }

    #[graphql(ignore)]
    pub fn hidden_from_graphql(&self) {
        // [...]
    }
}

impl Person {
    pub fn hidden_from_graphql2(&self) {
        // [...]
    }
}
#
# fn main() { }
```

これは少し冗長ではありますが、フィールドリゾルバにあらゆる種類の関数を書くことができます。この構文では、フィールドは引数を取ることもできます。

```rust
# extern crate juniper;
# use juniper::{graphql_object, GraphQLObject};
#
#[derive(GraphQLObject)]
struct Person {
    name: String,
    age: i32,
}

struct House {
    inhabitants: Vec<Person>,
}

#[graphql_object]
impl House {
    // フィールド inhabitantWithName(name) を作成し、nullable の Person を返します.
    fn inhabitant_with_name(&self, name: String) -> Option<&Person> {
        self.inhabitants.iter().find(|p| p.name == name)
    }
}
#
# fn main() {}
```

データベース接続や認証情報などのグローバルなデータにアクセスするには、`context` を使用します。これについては、次の章を参照してください。[コンテキストを使う](using_contexts.md) をご覧ください。

## 説明、名前の変更、および非推奨

`derive` 属性と同様に、フィールド名は `snake_case` から `camelCase` に変換されます。変換をオーバーライドしたい場合は、フィールド名を変更するだけです。また、型名はエイリアスで変更することができます。

```rust
# extern crate juniper;
# use juniper::graphql_object;
#
struct Person;

/// Docコメントは、GraphQLの記述として使用されます.
#[graphql_object(
    // この属性で、型の公開GraphQL名を変更することができます.
    name = "PersonObject",

    // ここで説明を指定することもでき、その場合は上書きされます.
    description = "...",
)]
impl Person {
    /// フィールドのdocコメントは、GraphQLにも使用されます.
    #[graphql(
        // もしくは、ここで説明を記述することもできます.
        description = "...",
    )]
    fn doc_comment(&self) -> &str {
        ""
    }

    // 必要に応じて、フィールドの名前を変更することもできます.
    #[graphql(name = "myCustomFieldName")]
    fn renamed_field() -> bool {
        true
    }

    // 非推奨事項期待通りに動作します.
    // Rustの標準的な構文とカスタムderiveの両方を受け入れることができます.
    #[deprecated(note = "...")]
    fn deprecated_standard() -> bool {
        false
    }

    #[graphql(deprecated = "...")]
    fn deprecated_graphql() -> bool {
        true
    }
}
#
# fn main() { }
```

あるいは、`impl`ブロックのすべてのフィールドに対して、異なるリネームポリシーを提供します。

```rust
# extern crate juniper;
# use juniper::graphql_object;
struct Person;

#[graphql_object(rename_all = "none")] // リネームを無効にする.
impl Person {
    // スキーマで `renamed_field` として公開されるようになりました.
    fn renamed_field() -> bool {
        true
    }
}
#
# fn main() {}
```

## 引数のカスタマイズ

メソッドフィールドの引数もカスタマイズすることができます。

それらはカスタム記述とデフォルト値を持つことができます。

```rust
# extern crate juniper;
# use juniper::graphql_object;
#
struct Person {}

#[graphql_object]
impl Person {
    fn field1(
        &self,
        #[graphql(
            // また、必要に応じて引数の名前を変更することができます.
            name = "arg",
            // 存在しない場合に注入されるデフォルト値デフォルト値を設定します.
            // デフォルトは、関数呼び出しなどを含む有効なRust式であれば、どのようなものでもよい.
            default = true,
            // 説明文を設定します.
            description = "The first argument..."
        )]
        arg1: bool,
        // デフォルト値が指定されていない場合は Default::default() の値が使用される。
        #[graphql(default)]
        arg2: i32,
    ) -> String {
        format!("{} {}", arg1, arg2)
    }
}
#
# fn main() { }
```