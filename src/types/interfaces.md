# インターフェイス

[GraphQLインターフェース][1]は、Java や C# などの一般的なオブジェクト指向言語で知られているインターフェースによく対応していますが、残念ながら Rust には完全に対応する概念がありません。[GraphQLインターフェース][1]に最も近いアナログはRust traitで、主な違いは、GraphQLでは[interface type][1]は抽象化と boxed 値(具象実装者にダウンキャスト可能)として機能しますが、Rustではtraitは抽象化のみでその boxed 値を表現するにはenumやtrait objectなどの別のタイプが必要_です、Rust traitでは型自体を表現しないので値を持てないのです。この違いは、Rustで[GraphQLインターフェース][1]を表現しようとしたときに、直感的でない、明らかでないコーナーケースをもたらしますが、一方で、どの型がインターフェースを支えるのか、そしてどのように解決するのかを完全に制御することができます。

[GraphQLインターフェース][1]を実装するために、Juniperは `#[graphql_interface]` マクロを提供します。

## トレイト

[GraphQLインターフェース][1]を定義するためには trait の定義が必須です。これは Rust における abstraction を記述する 自明な方法だからです。すべての[interface][1]フィールドは、traitメソッドによって計算されたものとして定義されます。

```rust
# extern crate juniper;
use juniper::graphql_interface;

#[graphql_interface]
trait Character {
    fn id(&self) -> &str;
}
#
# fn main() {}
```

ただし、このような[interface][1]の値を返すためには、その実装者と、この特質の boxed 値を表すRust型を提供する必要があります。この型は，enum型と[trait object][2]の2種類があります．

### 既定の列挙値

デフォルトでは、Juniper は定義された [GraphQL interface][1] の値を表す enum を生成し、素直に `{Interface}Value` という名前を付けます。

```rust
# extern crate juniper;
use juniper::{graphql_interface, GraphQLObject};

#[graphql_interface(for = [Human, Droid])] // すべてのインプリメンターの列挙が必須.
trait Character {
    fn id(&self) -> &str;
}

#[derive(GraphQLObject)]
#[graphql(impl = CharacterValue)] // 列挙型の名前であり、traitの名前ではないことに注意してください.
struct Human {
    id: String,
}

#[derive(GraphQLObject)]
#[graphql(impl = CharacterValue)]
struct Droid {
    id: String,
}
#
# fn main() {}
```

また、必要に応じて、列挙型の名前を明示的に指定することができます。

```rust
# extern crate juniper;
use juniper::{graphql_interface, GraphQLObject};

#[graphql_interface(enum = CharacterInterface, for = Human)] 
trait Character {
    fn id(&self) -> &str;
}

#[derive(GraphQLObject)]
#[graphql(impl = CharacterInterface)]
struct Human {
    id: String,
    home_planet: String,
}
#
# fn main() {}
```

### トレイトメソッドを無視する

[GraphQL interface][1]のフィールドとして想定するため、一部のtraitメソッドを省略し、無視したい場合があります。

```rust
# extern crate juniper;
use juniper::{graphql_interface, GraphQLObject};

#[graphql_interface(for = Human)]  
trait Character {
    fn id(&self) -> &str;

    #[graphql(ignore)] // または #[graphql(skip)], お好みで。
    fn ignored(&self) -> u32 { 0 }
}

#[derive(GraphQLObject)]
#[graphql(impl = CharacterValue)]
struct Human {
    id: String,
}
#
# fn main() {}
```

### フィールド、引数、インターフェースのカスタマイズ

[GraphQLオブジェクト][5]と同様に、Juniperでは[interface][1]のフィールドとその引数を完全にカスタマイズすることができます。

```rust
# #![allow(deprecated)]
# extern crate juniper;
use juniper::graphql_interface;

// GraphQLスキーマのインターフェイスの名前を変更します.
#[graphql_interface(name = "MyCharacter")] 
// GraphQLスキーマでインターフェイスを記述します.
#[graphql_interface(description = "My own character.")]
// 通常の Rust docs も GraphQL インターフェイスの記述としてサポートされていますが、 description 属性が指定されている場合は、そちらが優先されます.
/// このdocはGraphQLスキーマには存在しません. 
trait Character {
    // GraphQLスキーマのフィールド名を変更します.
    #[graphql(name = "myId")]
    // GraphQL スキーマのフィールドを非推奨とします.
    // 通常の Rust #[deprecated] 属性も非推奨フィールドとしてサポートされているが、 deprecated 属性引数が指定されていれば、そちらが優先されます.
    #[graphql(deprecated = "Do not use it.")]
    // GraphQL スキーマでフィールドを記述します.
    #[graphql(description = "ID of my own character.")]
    // Rust の通常のドキュメントもフィールドの説明としてサポートされているが、 description`属性が指定されていれば、そちらが優先される。
    /// GraphQLスキーマにはこの記述はありません.
    fn id(
        &self,
        // GraphQLスキーマでの引数の名前を変更します.
        #[graphql(name = "myNum")]
        // GraphQLスキーマで引数を記述します.
        #[graphql(description = "ID number of my own character.")]
        // 引数のデフォルト値を指定します.
        // 具体的な値は省略可能で、その場合は Default::default のものが使用されます.
        #[graphql(default = 5)]
        num: i32,
    ) -> &str;
}
#
# fn main() {}
```

すべての[GraphQL interface][1]フィールドと引数のリネームポリシーもサポートされています。

```rust
# #![allow(deprecated)]
# extern crate juniper;
use juniper::graphql_interface;

#[graphql_interface(rename_all = "none")] // リネームを無効にします.
trait Character {
    // スキーマで my_id と my_num として公開されるようになりました.
    fn my_id(&self, my_num: i32) -> &str;
}
#
# fn main() {}
```

### カスタムコンテキスト

[GraphQL interface][1] フィールドを解決するために、trait メソッドで [`Context`][6] が必要な場合は、引数として指定します。

```rust
# extern crate juniper;
# use std::collections::HashMap;
use juniper::{graphql_interface, GraphQLObject};

struct Database {
    humans: HashMap<String, Human>,
}
impl juniper::Context for Database {}

#[graphql_interface(for = Human)] // 見て、ママ、コンテキストの型が推論されてる! ＼(^o^)／
trait Character {
    // フィールドの引数が context または ctx という名前であれば、自動的にコンテキスト引数として扱われます.
    fn id(&self, context: &Database) -> Option<&str>;

    // そうでない場合は、明示的にコンテキスト引数としてマークすることができます.
    fn name(&self, #[graphql(context)] db: &Database) -> Option<&str>;
}

#[derive(GraphQLObject)]
#[graphql(impl = CharacterValue, Context = Database)]
struct Human {
    id: String,
    name: String,
}
#
# fn main() {}
```

### エクゼキュータと明示的なジェネリックスカラを使う

[GraphQL interface][1] フィールドを解決するために trait メソッド内で [`Executor`][4] が必要な場合、それを引数として指定します。

この場合、[`Executor`][4] がそうであるように、明示的に [`ScalarValue`][3] をパラメトリックにする必要があります。

```rust
# extern crate juniper;
use juniper::{graphql_interface, graphql_object, Executor, LookAheadMethods as _, ScalarValue};

#[graphql_interface(for = Human, Scalar = S)] // 既存の型パラメータとして ScalarValue を指定している.
trait Character<S: ScalarValue> {             
    // フィールドの引数が executor という名前であれば、自動的にエグゼキュータ引数として扱われます.
    fn id<'a>(&self, executor: &'a Executor<'_, '_, (), S>) -> &'a str;

    // そうでない場合は、エグゼキュータ引数として明示的にマークすることができます.
    fn name<'b>(
        &'b self,
        #[graphql(executor)] another: &Executor<'_, '_, (), S>,
    ) -> &'b str;
    
    fn home_planet(&self) -> &str;
}

struct Human {
    id: String,
    name: String,
    home_planet: String,
}
#[graphql_object(scalar = S: ScalarValue, impl = CharacterValue<S>)]
impl Human {
    async fn id<'a, S>(&self, executor: &'a Executor<'_, '_, (), S>) -> &'a str 
    where
        S: ScalarValue,
    {
        executor.look_ahead().field_name()
    }

    async fn name<'b, S>(&'b self, #[graphql(executor)] _: &Executor<'_, '_, (), S>) -> &'b str {
        &self.name
    }
    
    fn home_planet<'c, S>(&'c self, #[graphql(executor)] _: &Executor<'_, '_, (), S>) -> &'c str {
        // トレイトメソッドにはエクゼキュータが存在しない場合があります.
        &self.home_planet
    }
}
#
# fn main() {}
```

## ScalarValueの考察

デフォルトでは `#[graphql_interface]` マクロは [`ScalarValue`][3] 型に対するジェネリックコードを生成します。これは、[GraphQL interface][1] の実装者のうち少なくとも1人が、その実装において具象的な [`ScalarValue`][3] 型に制限されている場合に問題を引き起こす可能性があります。このような問題を解決するためには、具象 [`ScalarValue`][3] 型を指定する必要があります。

```rust
# extern crate juniper;
use juniper::{graphql_interface, DefaultScalarValue, GraphQLObject};

#[graphql_interface(for = [Human, Droid])]
#[graphql_interface(scalar = DefaultScalarValue)] // この行を削除すると、コンパイルに失敗します.
trait Character {
    fn id(&self) -> &str;
}

#[derive(GraphQLObject)]
#[graphql(impl = CharacterValue, Scalar = DefaultScalarValue)]
struct Human {
    id: String,
    home_planet: String,
}

#[derive(GraphQLObject)]
#[graphql(impl = CharacterValue, Scalar = DefaultScalarValue)]
struct Droid {
    id: String,
    primary_function: String,
}
#
# fn main() {}
```

[1]: https://spec.graphql.org/June2018/#sec-Interfaces
[2]: https://doc.rust-lang.org/reference/types/trait-object.html
[3]: https://docs.rs/juniper/latest/juniper/trait.ScalarValue.html
[4]: https://docs.rs/juniper/latest/juniper/struct.Executor.html
[5]: https://spec.graphql.org/June2018/#sec-Objects
[6]: https://docs.rs/juniper/0.14.2/juniper/trait.Context.html