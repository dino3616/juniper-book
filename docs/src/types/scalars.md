# スカラー

スカラーは、GraphQLクエリの先頭にあるプリミティブな型であり、数値、文字列、ブーリアンです。他のプリミティブ値に対するカスタムスカラーを作成することもできますが、この場合、構築するAPIを利用するためのクライアントライブラリとの調整が必要になることがよくあります。

ワイヤーを介した値は最終的にJSONに変換されるため、使用できるデータ型も限定されます。

カスタムスカラーを定義するには、2つの方法があります。

* プリミティブ型をラップするだけの単純なスカラーであれば、newtypeパターンとカスタムderiveを使用することができる。
* より高度な検証を行う場合は、 `graphql_scalar` 手続き型マクロを使用することができます。

## 組み込みスカラー

ジュニパーは、以下の組み込みをサポートしています。

* `i32` を `Int` とします。
* `f64` を `Float` とします。
* 文字列 `String` と `&str` は `String` とします。
* ブール演算子として `bool` を使用します。
* `juniper::ID` を `ID` とします。この型は [in the spec](http://facebook.github.io/graphql/#sec-ID) は、文字列としてシリアライズされますが、文字列と整数の両方からパースできるタイプです。

GraphQL 仕様では、[`i64`/`u64` に対する組み込みのスカラーはデフォルトで定義されていない](https://spec.graphql.org/June2018/#sec-Int) ので、`i64`/`u64` に対する組み込みのサポートはないことに注意してください。スキーマで [カスタム GraphQL スカラー](#custom-scalars) を利用して、これらをサポートすることができます。

## サードパーティーの型

Juniperは、一般的なサードパーティークレートから、さらにいくつかのタイプを組み込みでサポートしています。これらは、デフォルトでオンになっている機能によって有効化されます。

* uuid::Uuid
* chrono::DateTime
* time::{Date, OffsetDateTime, PrimitiveDateTime, Time, UtcOffset}
* url::Url
* bson::oid::ObjectId

## newtype パターン

既存の型をラップしただけのカスタムスカラーが必要な場合がよくあります。

これは、serde が `#[serde(transparent)]` でこのパターンをサポートしているのと同様に、 newtype パターンとカスタム derive で実現することができます。

```rust
# extern crate juniper;
#[derive(juniper::GraphQLScalarValue)]
pub struct UserId(i32);

#[derive(juniper::GraphQLObject)]
struct User {
    id: UserId,
}

# fn main() {}
```

以上で、スキーマに `UserId` を使用できるようになりました。

また、このマクロを使うことで、よりカスタマイズが可能になります。

```rust
# extern crate juniper;
/// doc commentを使用して説明を指定することができます.
#[derive(juniper::GraphQLScalarValue)]
#[graphql(
    transparent,
    // GraphQLの型名に上書きします.
    name = "MyUserId",
    // カスタムの説明を指定します.
    // 属性に記述すると、docコメントが上書きされます.
    description = "My user id description",
)]
pub struct UserId(i32);

# fn main() {}
```

## カスタムスカラー

カスタムパースやバリデーションが必要な、より複雑な状況では、 `graphql_scalar` 手続き型マクロを使用することができます。

通常、カスタムスカラーは文字列として表現されます。

以下の例では、カスタムの `Date` 型のスカラーを実装しています。

注意: juniper は `chrono::DateTime` 型を `chrono` 機能で既に組み込んでおり、これはデフォルトで有効になっているので、この目的に使用することができます。

以下の例は、説明のために使用しています。

**注意**: この例では、 `Date` 型が `std::fmt::Display` と `std::str::FromStr` を実装していると仮定しています。

```rust
# extern crate juniper;
# mod date {
#    pub struct Date;
#    impl std::str::FromStr for Date {
#        type Err = String; fn from_str(_value: &str) -> Result<Self, Self::Err> { unimplemented!() }
#    }
#    // そして、日付を文字列として表現する方法を定義しています.
#    impl std::fmt::Display for Date {
#        fn fmt(&self, _f: &mut std::fmt::Formatter) -> std::fmt::Result {
#            unimplemented!()
#        }
#    }
# }
#
use juniper::{Value, ParseScalarResult, ParseScalarValue};
use date::Date;

#[juniper::graphql_scalar(description = "Date")]
impl<S> GraphQLScalar for Date
where
    S: ScalarValue
{
    // カスタムスカラーをプリミティブ型に変換する方法を定義します.
    fn resolve(&self) -> Value {
        Value::scalar(self.to_string())
    }

    // プリミティブ型のパース方法を定義し、独自のスカラーを作成します.
    // 注意: エラーの種類は IntoFieldError<S> を実装する必要があります。
    fn from_input_value(v: &InputValue) -> Result<Date, String> {
        v.as_string_value()
            .ok_or_else(|| format!("Expected `String`, found: {}", v))
            .and_then(|s| s.parse().map_err(|e| format!("Failed to parse `Date`: {}", e)))
    }

    // 文字列値のパース方法を定義します.
    fn from_str<'a>(value: ScalarToken<'a>) -> ParseScalarResult<'a, S> {
        <String as ParseScalarValue<S>>::from_str(value)
    }
}
#
# fn main() {}
```