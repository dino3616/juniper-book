# コンテキストを使う

コンテキスト型は、Juniper の機能で、フィールドリゾルバがグローバルデータ (最も一般的なのはデータベース接続や認証情報) にアクセスできるようにするものです。コンテキストは通常、context factory から作成されます。そのため、[Iron](../../servers/iron.md) や [Rocket](../../servers/rocket.md) のドキュメントをチェックしてみてください。

この章では、コンテキストタイプを定義し、それをフィールドリゾルバで使用する方法を説明します。例えば、`HashMap` の中に簡単なユーザーデータベースがあるとします。

```rust
# #![allow(dead_code)]
# use std::collections::HashMap;
#
struct Database {
    users: HashMap<i32, User>,
}

struct User {
    id: i32,
    name: String,
    friend_ids: Vec<i32>,
}
#
# fn main() { }
```

`User` オブジェクトのリストを返す `friends` フィールドを `User` に設けたいと思います。
しかし、そのようなフィールドを作成するためには、データベースに問い合わせる必要があります。

これを解決するために、`Database` を有効なコンテキストタイプとしてマークして、user オブジェクトに割り当てます。

コンテキストにアクセスするには、その型に指定した `Context` と同じ型を持つ引数を指定する必要があります。

```rust
# extern crate juniper;
# use std::collections::HashMap;
# use juniper::graphql_object;
#
// この構造体は、私たちのコンテキストを表します.
struct Database {
    users: HashMap<i32, User>,
}

// データベースをJuniperの有効なコンテキストタイプとしてマークします.
impl juniper::Context for Database {}

struct User {
    id: i32,
    name: String,
    friend_ids: Vec<i32>,
}

// UserのコンテキストタイプにDatabaseを指定する.
#[graphql_object(context = Database)]
impl User {
    // コンテキストの種類を引数に指定して、コンテキストを注入する.
    // Note: 
    //   - 型は参照でなければなりません.
    //   - 引数の名前は `context` であるべきです.
    fn friends<'db>(&self, context: &'db Database) -> Vec<&'db User> {
        // データベースを利用してユーザーを検索する.
        self.friend_ids.iter()
            .map(|id| context.users.get(id).expect("Could not find user with ID"))
            .collect()
    }

    fn name(&self) -> &str { 
        self.name.as_str() 
    }

    fn id(&self) -> i32 { 
        self.id 
    }
}
#
# fn main() { }
```

コンテキストへの不変の参照を得るだけなので、実行内容に変更を加えたい場合は、 `RwLock` や `RefCell` などの [可変なイテレータ](https://doc.rust-lang.org/book/first-edition/mutability.html#interior-vs-exterior-mutability) を使用する必要があります。

## 可変参照の扱い

並行フィールド解決が行われる可能性があるため、コンテキストをミュータブル参照で指定することはできません。もし、コンテキストにミュータブル参照によるアクセスが必要なものがあれば、そのために [可変なイテレータ][1] を活用する必要があります。

例えば、[work stealing][2] (`tokio` など) で非同期ランタイムを使用する場合、明らかにスレッドセーフが必要ですが、それに対応する非同期バージョンの `RwLock` を使用する必要があります。

非同期解決を使用しない場合は、 `tokio::sync::RwLock` を `std::sync::RwLock` (または類似のもの) に置き換えてください。

[1]: https://doc.rust-lang.org/book/ch15-05-interior-mutability.html
[2]: https://en.wikipedia.org/wiki/Work_stealing