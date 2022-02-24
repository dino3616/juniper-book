# 非構造体オブジェクト

これまで、構造体とGraphQLオブジェクトのマッピングについてだけ見てきました。しかし、どんなRust型でもGraphQLオブジェクトにマッピングすることが可能です。この章では、列挙型を取り上げますが、traitも同様に使えます。

`Result` ライクな列挙型を使用することで、例えば変異による検証エラーなどを報告するのに便利です。

```rust
# extern crate juniper;
# use juniper::{graphql_object, GraphQLObject};
# #[derive(juniper::GraphQLObject)] struct User { name: String }
#
#[derive(GraphQLObject)]
struct ValidationError {
    field: String,
    message: String,
}

# #[allow(dead_code)]
enum SignUpResult {
    Ok(User),
    Error(Vec<ValidationError>),
}

#[graphql_object]
impl SignUpResult {
    fn user(&self) -> Option<&User> {
        match *self {
            SignUpResult::Ok(ref user) => Some(user),
            SignUpResult::Error(_) => None,
        }
    }

    fn error(&self) -> Option<&Vec<ValidationError>> {
        match *self {
            SignUpResult::Ok(_) => None,
            SignUpResult::Error(ref errors) => Some(errors)
        }
    }
}
#
# fn main() {}
```

ここでは、ユーザーの入力データが有効かどうかを判断するためにenumを使用しており、例えばサインアップミューテーションの結果として使用することができます。

これはGraphQLオブジェクトを表現するのに構造体以外を使用する方法の例ですが、「予想される」エラー（検証エラーのようなエラー）に対するエラー処理を実装する方法の例でもあります。GraphQLでエラーをどのように表現するかについて明確なルールはありませんが、GraphQLの著者の一人が「ハード」フィールドエラーの使用方法と、期待されるエラーのモデル化方法について[いくつか](https://github.com/facebook/graphql/issues/117#issuecomment-170180628) [コメント](https://github.com/graphql/graphql-js/issues/560#issuecomment-259508214)を発表しています。