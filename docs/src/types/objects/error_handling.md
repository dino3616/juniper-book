# エラーハンドリング

GraphQLにおけるエラー処理は、複数の方法で行うことができます。以下では、2つの異なるエラー処理モデルについて説明します。フィールドの結果とGraphQLスキーマの裏付けとなるエラーです。それぞれのアプローチには利点があります。正しいエラー処理方法を選択することは、アプリケーションの要件に依存します。両方のアプローチを調査することは有益です。

## フィールドの結果

Rustはエラーを処理する方法を2つ提供しています。回復可能なエラーには `Result<T, E>` を、回復不可能なエラーには `panic!` を使用します。Juniperはパニックに対して何もしません。パニックは周囲のフレームワークにバブルアップされ、うまくいけばそこで処理されます。

回復可能なエラーの場合、Juniperは組み込みの `Result` 型でうまく動作し、 `?` 演算子を使用することで、一般的に期待通りの動作をします。

```rust
# extern crate juniper;
use std::{
    str,
    path::PathBuf,
    fs::{File},
    io::{Read},
};
use juniper::{graphql_object, FieldResult};

struct Example {
    filename: PathBuf,
}

#[graphql_object]
impl Example {
    fn contents(&self) -> FieldResult<String> {
        let mut file = File::open(&self.filename)?;
        let mut contents = String::new();
        file.read_to_string(&mut contents)?;
        Ok(contents)
    }

    fn foo() -> FieldResult<Option<String>> {
        // 一部無効なバイトがあります.
        let invalid = vec![128, 223];

        Ok(Some(str::from_utf8(&invalid)?.to_string()))
    }
}
#
# fn main() {}
```

`FieldResult<T>` は `Result<T, FieldError>` のエイリアスで、すべてのフィールドが返さなければならないエラータイプです。演算子 `?` や `try!` マクロを使用すると、 `Display` トレイトを実装した型 (世の中のほとんどのエラータイプ) は、それらのエラーが自動的に `FieldError` に変換されます。

## エラーペイロード、NULL、および部分エラー

Juniperのエラー動作は、[GraphQL仕様](https://spec.graphql.org/June2018/#sec-Errors-and-Non-Nullability)に準拠しています。

フィールドがエラーを返すと、フィールドの結果は `null` に置き換えられ、レスポンスのトップレベルに `errors` オブジェクトが追加で作成され、実行が再開されます。例えば、先ほどの例と以下のようなクエリの場合、

```graphql
{
  example {
    contents
    foo
  }
}
```

もし `str::from_utf8` が `std::str::Utf8Error` を発生させた場合は、以下のような結果が返ります。

```json
{
  "data": {
    "example": {
      contents: "<Contents of the file>",
      foo: null
    }
  },
  "errors": [
    "message": "invalid utf-8 sequence of 2 bytes from index 0",
    "locations": [{ "line": 2, "column": 4 }])
  ]
}
```

上記の例のように、NULL でないフィールドからエラーが返された場合、 `null` 値は最初に NULL にできる親フィールドまで伝搬され、NULL にできるフィールドがない場合はルートの `data` オブジェクトに伝搬されます。

例えば、以下のようなクエリの場合です。

```graphql
{
  example {
    contents
  }
}
```

上記の `File::open()` が `std::io::ErrorKind::PermissionDenied` を返した場合、以下のような結果が得られます。

```json
{
  "errors": [
    "message": "Permission denied (os error 13)",
    "locations": [{ "line": 2, "column": 4 }])
  ]
}
```

### 構造化されたエラー

時には、追加の構造化されたエラー情報をクライアントに返すことが望ましい場合があります。これは、[`IntoFieldError`](https://docs.rs/juniper/latest/juniper/trait.IntoFieldError.html) を実装することで実現可能です。

```rust
# #[macro_use] extern crate juniper;
# use juniper::{graphql_object, FieldError, IntoFieldError, ScalarValue};
#
enum CustomError {
    WhateverNotSet,
}

impl<S: ScalarValue> IntoFieldError<S> for CustomError {
    fn into_field_error(self) -> FieldError<S> {
        match self {
            CustomError::WhateverNotSet => FieldError::new(
                "Whatever does not exist",
                graphql_value!({
                    "type": "NO_WHATEVER"
                }),
            ),
        }
    }
}

struct Example {
    whatever: Option<bool>,
}

#[graphql_object]
impl Example {
    fn whatever(&self) -> Result<bool, CustomError> {
        if let Some(value) = self.whatever {
            return Ok(value);
        }
        Err(CustomError::WhateverNotSet)
    }
}
#
# fn main() {}
```

指定された構造化エラー情報は、[`extensions`](https://facebook.github.io/graphql/June2018/#sec-Errors)キーに含まれます。

```json
{
  "errors": [{
    "message": "Whatever does not exist",
    "locations": [{"line": 2, "column": 4}],
    "extensions": {
      "type": "NO_WHATEVER"
    }
  }]
}
```

## GraphQLのスキーマに裏付けられたエラー

Rustのエラーのモデルは、GraphQLに適応することができます。Rust のパニックは `FieldError` に似ています。クエリ全体が中断され、(エラー関連情報を除いて)何も取り出せなくなります。

すべてのエラーがこのような厳密な処理を必要とするわけではありません。回復可能なエラーや部分的なエラーは、クライアントがインテリジェントに処理できるように GraphQL スキーマに入れることができます。

このアプローチを実装するには、すべてのエラーを2つのエラークラスに分割する必要があります。

* ユーザーが修正できないクリティカルなエラー(例：データベースエラー)
* ユーザが修正可能な回復可能エラー(例：無効な入力データ)

クリティカルなエラーは、リゾルバから `FieldErrors` (前のセクションで説明) として返される。非クリティカルなエラーは GraphQL スキーマの一部であり、クライアントによって優雅に処理することができます。Rustと同様に、GraphQLでもユニオンを使った同様のエラーモデルが可能です（Unionsを参照）。

### 単純な入力検証の例

この例では、基本的な入力検証を GraphQL タイプで実装しています。問題のあるフィールド名を特定するために文字列が使用されています。特定のフィールドに対するエラーも文字列として返されます。この例では、文字列はサーバーサイドでローカライズされたエラーメッセージを含んでいます。しかし、一意の文字列識別子を返し、クライアントがローカライズされた文字列をユーザーに提示するようにすることも可能です。

```rust
# extern crate juniper;
# use juniper::{graphql_object, GraphQLObject, GraphQLUnion};
#
#[derive(GraphQLObject)]
pub struct Item {
    name: String,
    quantity: i32,
}

#[derive(GraphQLObject)]
pub struct ValidationError {
    field: String,
    message: String,
}

#[derive(GraphQLObject)]
pub struct ValidationErrors {
    errors: Vec<ValidationError>,
}

#[derive(GraphQLUnion)]
pub enum GraphQLResult {
    Ok(Item),
    Err(ValidationErrors),
}

pub struct Mutation;

#[graphql_object]
impl Mutation {
    fn addItem(&self, name: String, quantity: i32) -> GraphQLResult {
        let mut errors = Vec::new();

        if !(10 <= name.len() && name.len() <= 100) {
            errors.push(ValidationError {
                field: "name".to_string(),
                message: "between 10 and 100".to_string()
            });
        }

        if !(1 <= quantity && quantity <= 10) {
            errors.push(ValidationError {
                field: "quantity".to_string(),
                message: "between 1 and 10".to_string()
            });
        }

        if errors.is_empty() {
            GraphQLResult::Ok(Item { name, quantity })
        } else {
            GraphQLResult::Err(ValidationErrors { errors })
        }
    }
}
#
# fn main() {}
```

各関数は異なる戻り値の型を持ち、入力パラメータに応じて新しい結果型を必要とする場合があります。例えば、ユーザーを追加する場合は、`Ok(Item)`の代わりに `Ok(User)` というバリアントを含む新しい結果型を必要とします。

クライアントは、以下の例に示すように、変異のリクエストを送信し、その結果のエラーを処理することができます。

```graphql
{
  mutation {
    addItem(name: "", quantity: 0) {
      ... on Item {
        name
      }
      ... on ValidationErrors {
        errors {
          field
          message
        }
      }
    }
  }
}
```

この方法の有用な副次的効果は、部分的に成功したクエリーまたはミューテーションを持つことです。あるリゾルバーが失敗しても、成功したリゾルバーの結果は破棄されません。

### 複雑な入力検証の例

文字列を使ってエラーを伝える代わりに、GraphQLの型システムを使って、より正確にエラーを記述することが可能です。

誤りやすい入力変数ごとに、GraphQLオブジェクトのフィールドが作成されます。そのフィールドは、その特定のフィールドに対するバリデーションが失敗した場合に設定されます。必要な型の数が以前より大幅に増えたため、繰り返しを減らすために何らかのコード生成が必要になると思われます。各リゾルバ関数はカスタム `ValidationResult` を持ち、その関数が提供するフィールドのみを含みます。

```rust
# extern crate juniper;
# use juniper::{graphql_object, GraphQLObject, GraphQLUnion};
#
#[derive(GraphQLObject)]
pub struct Item {
    name: String,
    quantity: i32,
}

#[derive(GraphQLObject)]
pub struct ValidationError {
    name: Option<String>,
    quantity: Option<String>,
}

#[derive(GraphQLUnion)]
pub enum GraphQLResult {
    Ok(Item),
    Err(ValidationError),
}

pub struct Mutation;

#[graphql_object]
impl Mutation {
    fn addItem(&self, name: String, quantity: i32) -> GraphQLResult {
        let mut error = ValidationError {
            name: None,
            quantity: None,
        };

        if !(10 <= name.len() && name.len() <= 100) {
            error.name = Some("between 10 and 100".to_string());
        }

        if !(1 <= quantity && quantity <= 10) {
            error.quantity = Some("between 1 and 10".to_string());
        }

        if error.name.is_none() && error.quantity.is_none() {
            GraphQLResult::Ok(Item { name, quantity })
        } else {
            GraphQLResult::Err(error)
        }
    }
}
#
# fn main() {}
```

```graphql
{
  mutation {
    addItem {
      ... on Item {
        name
      }
      ... on ValidationErrorsItem {
        name
        quantity
      }
    }
  }
}
```

期待されるエラーはクエリ内部で直接処理されます。さらに、重要でないエラーはすべて、サーバとクライアントの両方が事前に知っています。

### 重大なエラーを含む入力検証の例

これまでの例では、ノンクリティカルエラーしか含まれていませんでした。GraphQLスキーマの内部でエラーを提供することで、予期せぬクリティカルエラーが発生したときに、それを返すことができます。

以下の例では、理論上のデータベースが故障し、エラーを発生させる可能性があります。データベースが失敗することは一般的ではないので、対応するエラーはクリティカルエラーとして返されます。

```rust
# extern crate juniper;
#
use juniper::{graphql_object, graphql_value, FieldError, GraphQLObject, GraphQLUnion, ScalarValue};

#[derive(GraphQLObject)]
pub struct Item {
    name: String,
    quantity: i32,
}

#[derive(GraphQLObject)]
pub struct ValidationErrorItem {
    name: Option<String>,
    quantity: Option<String>,
}

#[derive(GraphQLUnion)]
pub enum GraphQLResult {
    Ok(Item),
    Err(ValidationErrorItem),
}

pub enum ApiError {
    Database,
}

impl<S: ScalarValue> juniper::IntoFieldError<S> for ApiError {
    fn into_field_error(self) -> FieldError<S> {
        match self {
            ApiError::Database => FieldError::new(
                "Internal database error",
                graphql_value!({
                    "type": "DATABASE"
                }),
            ),
        }
    }
}

pub struct Mutation;

#[graphql_object]
impl Mutation {
    fn addItem(&self, name: String, quantity: i32) -> Result<GraphQLResult, ApiError> {
        let mut error = ValidationErrorItem {
            name: None,
            quantity: None,
        };

        if !(10 <= name.len() && name.len() <= 100) {
            error.name = Some("between 10 and 100".to_string());
        }

        if !(1 <= quantity && quantity <= 10) {
            error.quantity = Some("between 1 and 10".to_string());
        }

        if error.name.is_none() && error.quantity.is_none() {
            Ok(GraphQLResult::Ok(Item { name, quantity }))
        } else {
            Ok(GraphQLResult::Err(error))
        }
    }
}
#
# fn main() {}
```

## 追加資料

[Shopify API](https://shopify.dev/docs/admin-api/graphql/reference)も同様のアプローチを実装しています。彼らのAPIは、実世界のアプリケーションでこのアプローチを探求するための良い参考となります。

# 比較

上で説明した最初のアプローチ、つまりすべてのエラーが `FieldResult` で定義されたクリティカルエラーである場合、実装はより簡単です。しかし、クライアントはどのようなエラーが発生するのかを知らないので、代わりにエラー文字列から何が起こったのかを推測しなければなりません。これは脆弱で、クライアントやサーバーが変化することによって、時間の経過とともに変化する可能性があります。したがって、クライアントとサーバー間の暗黙の契約を維持するために、クライアントとサーバー間の広範な統合テストが必要とされます。

GraphQLスキーマにクリティカルでないエラーをエンコードすることで、クライアントとサーバーの間の契約を明示的にします。これにより、クライアントはこれらのエラーを理解して正しく処理することができ、サーバーは変更がクライアントを破壊する可能性があることを知ることができます。しかし、このエラー情報をGraphQLスキーマにエンコードするためには、追加のコードとノンクリティカルエラーの先行定義が必要です。