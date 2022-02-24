# オブジェクトとジェネリクス

GraphQLとRustが異なるもうひとつのポイントは、ジェネリックの仕組みです。Rustでは、ほとんどすべての型がジェネリックになり得ます。つまり、型パラメータを取ります。GraphQLでは、リストと非Nullableの2つしかジェネリック型がありません。

このため、RustからGraphQLで公開できるものに制限があります。ジェネリックな構造体は公開できず、すべての型パラメータはバインドされなければなりません。例えば、`Result<T, E>` などは GraphQL 型にできませんが、`Result<User, String>` などは GraphQL 型にすることができます。

[前章](non_struct_objects.md)の実装をもう少しコンパクトに、しかし汎用的にしてみましょう。

```rust
# extern crate juniper;
# #[derive(juniper::GraphQLObject)] struct User { name: String }
# #[derive(juniper::GraphQLObject)] struct ForumPost { title: String }

#[derive(juniper::GraphQLObject)]
struct ValidationError {
    field: String,
    message: String,
}

# #[allow(dead_code)]
struct MutationResult<T>(Result<T, Vec<ValidationError>>);

#[juniper::graphql_object(
    name = "UserResult",
)]
impl MutationResult<User> {
    fn user(&self) -> Option<&User> {
        self.0.as_ref().ok()
    }

    fn error(&self) -> Option<&Vec<ValidationError>> {
        self.0.as_ref().err()
    }
}

#[juniper::graphql_object(
    name = "ForumPostResult",
)]
impl MutationResult<ForumPost> {
    fn forum_post(&self) -> Option<&ForumPost> {
        self.0.as_ref().ok()
    }

    fn error(&self) -> Option<&Vec<ValidationError>> {
        self.0.as_ref().err()
    }
}

# fn main() {}
```

ここでは、 `Result` のラッパーを作成し、 `Result<T, E>` の具体的なインスタンスを個別の GraphQL オブジェクトとして公開しています。このラッパーが必要な理由は、Rustのトレイトを派生させる際のルールにあります。この場合、 `Result` と Juniper の内部 GraphQL トレイトは、どちらもサードパーティから取得したものです。

ジェネリックスを使用しているので、インスタンス化された型の名前も指定する必要があります。仮にJuniperがその名前を理解できたとしても、 `MutationResult<User>` は有効なGraphQLの型名ではありません。