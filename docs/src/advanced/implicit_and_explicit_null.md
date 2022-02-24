# 暗黙的および明示的なNULL

クライアントがクエリでNullの引数やフィールドを送信するには、2つの方法があります。

私たちは NULL リテラルを使用することができます。

```graphql
{
    field(arg: null)
}
```

あるいは、単に論点を省略することもできます。

```graphql
{
    field
}
```

前者は明示的なNULL、後者は暗黙的なNULLです。

ユーザーがどちらを提供したかを知っておくと便利な場面もあります。

例えば、あなたのビジネスロジックに、ユーザーが自分自身に対してパッチ操作を行うための関数があるとします。ユーザーは、オプションで好きな番号と嫌いな番号を持つことができ、そのための入力は次のようになるとします。

```rust
/// ユーザー属性を更新します. None になっているフィールドはそのままです.
pub struct UserPatch {
    /// もし Some ならば、ユーザのお気に入り番号を更新する.
    pub favorite_number: Option<Option<i32>>,

    /// もし Some ならば、ユーザの一番好きな番号を更新する.
    pub least_favorite_number: Option<Option<i32>>,
}

# fn main() {}
```

ユーザーの好きな数字を 7 に設定するには、`favorite_number` を `Some(Some(7))` に設定します。GraphQLでは、このようになります。

```graphql
mutation { patchUser(patch: { favoriteNumber: 7 }) }
```

ユーザーのお気に入り番号を解除するには、 `favorite_number` を `Some(None)` に設定します。GraphQL では、このようになります。

```graphql
mutation { patchUser(patch: { favoriteNumber: null }) }
```

もし、ユーザーのお気に入り番号をそのままにしておきたい場合は、`None`に設定することになります。GraphQLでは、次のようになります。

```graphql
mutation { patchUser(patch: {}) }
```

最後の2つのケースは、明示的なNULLと暗黙的なNULLを区別できるかどうかにかかっています。

Juniperでは、これは `Nullable` 型を使用して行うことができます。

```rust
# extern crate juniper;
use juniper::{FieldResult, Nullable};

#[derive(juniper::GraphQLInputObject)]
struct UserPatchInput {
    pub favorite_number: Nullable<i32>,
    pub least_favorite_number: Nullable<i32>,
}

impl Into<UserPatch> for UserPatchInput {
    fn into(self) -> UserPatch {
        UserPatch {
            // 明示的な関数は、ビジネスロジック層が期待するように Nullable を Option<Option<T>> に変換します。
            favorite_number: self.favorite_number.explicit(),
            least_favorite_number: self.least_favorite_number.explicit(),
        }
    }
}

# pub struct UserPatch {
#     pub favorite_number: Option<Option<i32>>,
#     pub least_favorite_number: Option<Option<i32>>,
# }

# struct Session;
# impl Session {
#     fn patch_user(&self, _patch: UserPatch) -> FieldResult<()> { Ok(()) }
# }

struct Context {
    session: Session,
}
impl juniper::Context for Context {}

struct Mutation;

#[juniper::graphql_object(context = Context)]
impl Mutation {
    fn patch_user(ctx: &Context, patch: UserPatchInput) -> FieldResult<bool> {
        ctx.session.patch_user(patch.into())?;
        Ok(true)
    }
}
# fn main() {}
```

この型は `Option` とよく似た機能を持ちますが、2つの空のバリアントを持っているので、暗黙的なヌルと明示的なヌルを区別することができます。