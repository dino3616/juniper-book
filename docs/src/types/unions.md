# ユニオン

サーバーから見ると、[ユニオン][1] は [インターフェース][5] にやや似ています。主な違いは、それ自体にフィールドが含まれないことです。

Rustで[ユニオン][1]を表現する最も明白でわかりやすい方法はenumです。しかし、trait や通常の struct でも表現することができます。そのため、Juniperでは[ユニオン][1]を実装するために、以下のようなものを用意しています。

* enumと構造体に対して `#[derive(GraphQLUnion)]` というマクロを用意しています。
* trait には `#[graphql_union]` マクロを提供します。

## 列挙体

ほとんどの場合、[ユニオン][1]を表現するために、些細でわかりやすいRust enumが必要なだけです。

```rust
# extern crate juniper;
# extern crate derive_more;
use derive_more::From;
use juniper::{GraphQLObject, GraphQLUnion};

#[derive(GraphQLObject)]
struct Human {
    id: String,
    home_planet: String,
}

#[derive(GraphQLObject)]
struct Droid {
    id: String,
    primary_function: String,
}

#[derive(From, GraphQLUnion)]
enum Character {
    Human(Human),
    Droid(Droid),
}
#
# fn main() {}
```

### 列挙体のバリアントを無視する

稀な状況として、GraphQLスキーマでenumバリアントを公開することを省略したい場合があります。

例として、リゾルバで型レベルの面白いことをするために、ある型パラメータ `T` をバインドする必要がある状況を考えてみましょう。これを実現するためには、 `PhantomData<T>` が必要だが、GraphQL スキーマで公開する必要はありません。

> __WARNING__:
> さもなければ、GraphQL クエリの解決は実行時にパニックになります。

```rust
# extern crate juniper;
# extern crate derive_more;
# use std::marker::PhantomData;
use derive_more::From;
use juniper::{GraphQLObject, GraphQLUnion};

#[derive(GraphQLObject)]
struct Human {
    id: String,
    home_planet: String,
}

#[derive(GraphQLObject)]
struct Droid {
    id: String,
    primary_function: String,
}

#[derive(From, GraphQLUnion)]
enum Character<S> {
    Human(Human),
    Droid(Droid),
    #[from(ignore)]
    #[graphql(ignore)]  // または #[graphql(skip)]、お好みで.
    _State(PhantomData<S>),
}
#
# fn main() {}
```

### 外部リゾルバ機能

[ユニオン][1] のバリアントを解決するために何らかのカスタムロジックが必要な場合、それを行うための外部関数を指定することができます。

```rust
# #![allow(dead_code)]
# extern crate juniper;
use juniper::{GraphQLObject, GraphQLUnion};

#[derive(GraphQLObject)]
#[graphql(Context = CustomContext)]
struct Human {
    id: String,
    home_planet: String,
}

#[derive(GraphQLObject)]
#[graphql(Context = CustomContext)]
struct Droid {
    id: String,
    primary_function: String,
}

pub struct CustomContext {
    droid: Droid,
}
impl juniper::Context for CustomContext {}

#[derive(GraphQLUnion)]
#[graphql(Context = CustomContext)]
enum Character {
    Human(Human),
    #[graphql(with = Character::droid_from_context)]
    Droid(Droid),
}

impl Character {
    // 注意: 関数のシグネチャは &self と &Context を含み、 Option<&VariantType> を返さなければなりません。
    fn droid_from_context<'c>(&self, ctx: &'c CustomContext) -> Option<&'c Droid> {
        Some(&ctx.droid)
    }
}
#
# fn main() {}
```

外部リゾルバ関数を使用すると、最初の列挙型の定義にRust型がない場合、新しい[ユニオン][1]バリアントを宣言することもできます。derive構文 `#[graphql(on VariantType = resolver_fn)]` は [ユニオンバリアントをディスパッチするためのGraphQL構文](https://spec.graphql.org/June2018/#example-f8163) に従っています。

```rust
# #![allow(dead_code)]
# extern crate juniper;
use juniper::{GraphQLObject, GraphQLUnion};

#[derive(GraphQLObject)]
#[graphql(Context = CustomContext)]
struct Human {
    id: String,
    home_planet: String,
}

#[derive(GraphQLObject)]
#[graphql(Context = CustomContext)]
struct Droid {
    id: String,
    primary_function: String,
}

#[derive(GraphQLObject)]
#[graphql(Context = CustomContext)]
struct Ewok {
    id: String,
    is_funny: bool,
}

pub struct CustomContext {
    ewok: Ewok,
}
impl juniper::Context for CustomContext {}

#[derive(GraphQLUnion)]
#[graphql(Context = CustomContext)]
#[graphql(on Ewok = Character::ewok_from_context)]
enum Character {
    Human(Human),
    Droid(Droid),
    #[graphql(ignore)]  // または #[graphql(skip)]、お好みで.
    Ewok,
}

impl Character {
    fn ewok_from_context<'c>(&self, ctx: &'c CustomContext) -> Option<&'c Ewok> {
        if let Self::Ewok = self {
            Some(&ctx.ewok)
        } else {
            None
        }       
    }
}
#
# fn main() {}
```

## 構造体

Rust構造体を[ユニオン][1]として使用することは、enumの使用と非常に似ていますが、外部のリゾルバ関数を指定することが[ユニオン][1]のバリアントを宣言する唯一の方法というニュアンスがあります。

```rust
# extern crate juniper;
# use std::collections::HashMap;
use juniper::{GraphQLObject, GraphQLUnion};

#[derive(GraphQLObject)]
#[graphql(Context = Database)]
struct Human {
    id: String,
    home_planet: String,
}

#[derive(GraphQLObject)]
#[graphql(Context = Database)]
struct Droid {
    id: String,
    primary_function: String,
}

struct Database {
    humans: HashMap<String, Human>,
    droids: HashMap<String, Droid>,
}
impl juniper::Context for Database {}

#[derive(GraphQLUnion)]
#[graphql(
    Context = Database,
    on Human = Character::get_human,
    on Droid = Character::get_droid,
)]
struct Character {
    id: String,
}

impl Character {
    fn get_human<'db>(&self, ctx: &'db Database) -> Option<&'db Human>{
        ctx.humans.get(&self.id)
    }

    fn get_droid<'db>(&self, ctx: &'db Database) -> Option<&'db Droid>{
        ctx.droids.get(&self.id)
    }
}
#
# fn main() {}
```

Rust の trait 定義を [ユニオン][1] として使用するには、 `#[graphql_union]` マクロを使用する必要があります。[Rustはtraitに対するderiveマクロを許可していません](https://doc.rust-lang.org/stable/reference/procedural-macros.html#derive-macros)ので、traitに対して `#[derive(GraphQLUnion)]` を使用してもうまくいきません。

> __NOTICE__:  
> なぜならスキーマリゾルバはその背後にある[ユニオン][1]を指定するために[トレイトオブジェクト](https://doc.rust-lang.org/stable/reference/types/trait-object.html) を返さなければならないからです。

```rust
# extern crate juniper;
use juniper::{graphql_union, GraphQLObject};

#[derive(GraphQLObject)]
struct Human {
    id: String,
    home_planet: String,
}

#[derive(GraphQLObject)]
struct Droid {
    id: String,
    primary_function: String,
}

#[graphql_union]
trait Character {
    // 注意: メソッドのシグネチャは &self を含み、 Option<&VariantType> を返さなければなりません.
    fn as_human(&self) -> Option<&Human> { None }
    fn as_droid(&self) -> Option<&Droid> { None }
}

impl Character for Human {
    fn as_human(&self) -> Option<&Human> { Some(&self) }
}

impl Character for Droid {
    fn as_droid(&self) -> Option<&Droid> { Some(&self) }
}
#
# fn main() {}
```

### カスタムコンテキスト

[ユニオン][1] のバリアントを解決するために trait メソッドで [`Context`][6] が必要な場合、それを引数として指定します。

```rust
# #![allow(unused_variables)]
# extern crate juniper;
# use std::collections::HashMap;
use juniper::{graphql_union, GraphQLObject};

#[derive(GraphQLObject)]
#[graphql(Context = Database)]
struct Human {
    id: String,
    home_planet: String,
}

#[derive(GraphQLObject)]
#[graphql(Context = Database)]
struct Droid {
    id: String,
    primary_function: String,
}

struct Database {
    humans: HashMap<String, Human>,
    droids: HashMap<String, Droid>,
}
impl juniper::Context for Database {}

#[graphql_union(context = Database)]
trait Character {
    // 注意: メソッドのシグネチャは、オプションで &Context を含むことができます.
    fn as_human<'db>(&self, ctx: &'db Database) -> Option<&'db Human> { None }
    fn as_droid<'db>(&self, ctx: &'db Database) -> Option<&'db Droid> { None }
}

impl Character for Human {
    fn as_human<'db>(&self, ctx: &'db Database) -> Option<&'db Human> {
        ctx.humans.get(&self.id)
    }
}

impl Character for Droid {
    fn as_droid<'db>(&self, ctx: &'db Database) -> Option<&'db Droid> {
        ctx.droids.get(&self.id)
    }
}
#
# fn main() {}
```

### トレイトメソッドを無視する

enum と同様に、いくつかの trait メソッドを省略して [ユニオン][1] のバリアントとして想定し、それらを無視したい場合があります。

```rust
# extern crate juniper;
use juniper::{graphql_union, GraphQLObject};

#[derive(GraphQLObject)]
struct Human {
    id: String,
    home_planet: String,
}

#[derive(GraphQLObject)]
struct Droid {
    id: String,
    primary_function: String,
}

#[graphql_union]
trait Character {
    fn as_human(&self) -> Option<&Human> { None }
    fn as_droid(&self) -> Option<&Droid> { None }
    #[graphql(ignore)]  // または #[graphql(skip)]、お好みで.
    fn id(&self) -> &str;
}

impl Character for Human {
    fn as_human(&self) -> Option<&Human> { Some(&self) }
    fn id(&self) -> &str { self.id.as_str() }
}

impl Character for Droid {
    fn as_droid(&self) -> Option<&Droid> { Some(&self) }
    fn id(&self) -> &str { self.id.as_str() }
}
#
# fn main() {}
```

### 外部リゾルバ機能

enumやstructと同様に、[ユニオン][1] variant resolverとしてtraitメソッドを使用することは必須ではありません。その代わり、カスタム関数を指定することができます。

```rust
# extern crate juniper;
# use std::collections::HashMap;
use juniper::{graphql_union, GraphQLObject};

#[derive(GraphQLObject)]
#[graphql(Context = Database)]
struct Human {
    id: String,
    home_planet: String,
}

#[derive(GraphQLObject)]
#[graphql(Context = Database)]
struct Droid {
    id: String,
    primary_function: String,
}

struct Database {
    humans: HashMap<String, Human>,
    droids: HashMap<String, Droid>,
}
impl juniper::Context for Database {}

#[graphql_union(context = Database)]
#[graphql_union(
    on Human = DynCharacter::get_human,
    on Droid = get_droid,
)]
trait Character {
    #[graphql(ignore)]  // または #[graphql(skip)]、お好みで.
    fn id(&self) -> &str;
}

impl Character for Human {
    fn id(&self) -> &str { self.id.as_str() }
}

impl Character for Droid {
    fn id(&self) -> &str { self.id.as_str() }
}

// trait オブジェクトは常に `Send` と `Sync` です。
type DynCharacter<'a> = dyn Character + Send + Sync + 'a;

impl<'a> DynCharacter<'a> {
    fn get_human<'db>(&self, ctx: &'db Database) -> Option<&'db Human> {
        ctx.humans.get(self.id())
    }
}

// 外部リゾルバ関数は、ある型のメソッドである必要はありません.
// あくまでも要件に合わせたファンクションシグネチャーの問題です.
fn get_droid<'db>(ch: &DynCharacter<'_>, ctx: &'db Database) -> Option<&'db Droid> {
    ctx.droids.get(ch.id())
}
#
# fn main() {}
```

## スカラ値の考察

デフォルトでは `#[derive(GraphQLUnion)]` と `#[graphql_union]` マクロは [`ScalarValue`][2] 型に対してジェネリックなコードを生成します。これは、[ユニオン][1] の少なくとも1つのバリアントが、その実装において具象的な [`ScalarValue`][2] 型に制限されている場合に問題を引き起こす可能性があります。このような問題を解決するには、具体的な [`ScalarValue`][2] 型を指定する必要があります。

```rust
# #![allow(dead_code)]
# extern crate juniper;
use juniper::{DefaultScalarValue, GraphQLObject, GraphQLUnion};

#[derive(GraphQLObject)]
#[graphql(Scalar = DefaultScalarValue)]
struct Human {
    id: String,
    home_planet: String,
}

#[derive(GraphQLObject)]
struct Droid {
    id: String,
    primary_function: String,
}

#[derive(GraphQLUnion)]
#[graphql(Scalar = DefaultScalarValue)]  // この行を削除すると、コンパイルに失敗します.
enum Character {
    Human(Human),
    Droid(Droid),
}
#
# fn main() {}
```

[1]: https://spec.graphql.org/June2018/#sec-Unions
[2]: https://docs.rs/juniper/latest/juniper/trait.ScalarValue.html
[5]: https://spec.graphql.org/June2018/#sec-Interfaces
[6]: https://docs.rs/juniper/0.14.2/juniper/trait.Context.html