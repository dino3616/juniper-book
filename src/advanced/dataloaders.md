# データローダーによるN+1問題の回避

Graphqlサーバーに共通する問題は、リゾルバがデータソースに問い合わせる方法です。
この問題により、不必要なデータベースクエリーやhttpリクエストが大量に発生します。
例えば、あるカルト宗教に属する人々をリストアップしたいとします。

```graphql
query {
  persons {
    id
    name
    cult {
      id
      name
    }
  }
}
```

SQLデータベースで実行されるのは、次のようなものでしょう。

```sql
SELECT id, name, cult_id FROM persons;
SELECT id, name FROM cults WHERE id = 1;
SELECT id, name FROM cults WHERE id = 1;
SELECT id, name FROM cults WHERE id = 1;
SELECT id, name FROM cults WHERE id = 1;
SELECT id, name FROM cults WHERE id = 2;
SELECT id, name FROM cults WHERE id = 2;
SELECT id, name FROM cults WHERE id = 2;
# ...
```

ユーザーのリストが返されると、各ユーザーのカルトを検索するために別のクエリーが実行されます。
これはすぐに問題になることがおわかりいただけると思います。

これに対する一般的な解決策は、**dataloader** を導入することです。
これはJuniperで[cksac/dataloader-rs](https://github.com/cksac/dataloader-rs)というクレートを使って行うことができ、キャッシュ型と非キャッシュ型の2種類のdataloaderを持っています。

#### キャッシュローダー

`DataLoader`はメモ化キャッシュを提供します。与えられたキーで`.load()`が一度呼ばれた後、結果の値は冗長なロードを排除するためにキャッシュされます。

`DataLoader`のキャッシュは、RedisやMemcache、その他のアプリケーションレベルの共有キャッシュを置き換えるものではありません。`DataLoader`は、何よりもまずデータロードのメカニズムであり、そのキャッシュは、アプリケーションへの単一のリクエストのコンテキストで同じデータを繰り返しロードしないという目的を果たすだけです。[(続きを読む)](https://github.com/graphql/dataloader#caching)

### どのようなものですか？

```toml
[dependencies]
actix-identity = "0.4.0-beta.4"
actix-rt = "1.0"
actix-web = {version = "2.0", features = []}
juniper = { git = "https://github.com/graphql-rust/juniper" }
futures = "0.3"
postgres = "0.15.2"
dataloader = "0.12.0"
async-trait = "0.1.30"
```

```rust, ignore
// use dataloader::cached::Loader;
use dataloader::non_cached::Loader;
use dataloader::BatchFn;
use std::collections::HashMap;
use postgres::{Connection, TlsMode};
use std::env;

pub fn get_db_conn() -> Connection {
    let pg_connection_string = env::var("DATABASE_URI").expect("need a db uri");
    println!("Connecting to {}", pg_connection_string);
    let conn = Connection::connect(&pg_connection_string[..], TlsMode::None).unwrap();
    println!("Connection is fine");
    conn
}

#[derive(Debug, Clone)]
pub struct Cult {
  pub id: i32,
  pub name: String,
}

pub fn get_cult_by_ids(hashmap: &mut HashMap<i32, Cult>, ids: Vec<i32>) {
  let conn = get_db_conn();
  for row in &conn
    .query("SELECT id, name FROM cults WHERE id = ANY($1)", &[&ids])
    .unwrap()
  {
    let cult = Cult {
      id: row.get(0),
      name: row.get(1),
    };
    hashmap.insert(cult.id, cult);
  }
}

pub struct CultBatcher;

#[async_trait]
impl BatchFn<i32, Cult> for CultBatcher {

    // 各オリジナルキーと カルト を対応させた配列を返す必要があるため、ハッシュマップを使用します.
    async fn load(&self, keys: &[i32]) -> HashMap<i32, Cult> {
        println!("load cult batch {:?}", keys);
        let mut cult_hashmap = HashMap::new();
        get_cult_by_ids(&mut cult_hashmap, keys.to_vec());
        cult_hashmap
    }
}

pub type CultLoader = Loader<i32, Cult, CultBatcher>;

// 新規にローダーを作成する.
pub fn get_loader() -> CultLoader {
    Loader::new(CultBatcher)
      // 通常、DataLoaderは、実行中の1フレーム内で発生する個々のロードをすべてまとめ、要求されたすべてのキーでバッチ関数を呼び出します.
      // しかし、この動作が好ましくない、あるいは最適でない場合があります.
      // おそらく、リクエストはその後の数回のティックに分散されることを想定しているのでしょう.
      // See: https://github.com/cksac/dataloader-rs/issues/12 
      // More info: https://github.com/graphql/dataloader#batch-scheduling 
      // Yield Countが大きいと、より多くのリクエストをバッチに追加できますが、実際にロードされるまでの待ち時間が長くなります.
      .with_yield_count(100)
}

#[juniper::graphql_object(Context = Context)]
impl Cult {
  //  あなたのリゾルバ

  // データローダを呼び出す.
  pub async fn cult_by_id(ctx: &Context, id: i32) -> Cult {
    ctx.cult_loader.load(id).await
  }
}

```

### どうやって呼び出すのですか？

データローダを作成すると、非同期関数 `.load()` と `.load_many()` を持つようになります。
上記の例では、 `cult_loader.load(id: i32).await` が `Cult` を返します。もし `cult_loader.load_many(Vec<i32>).await` を使用していれば、 `Vec<Cult>` を返していたことでしょう。

### データローダーはどこで作成するのですか？

データローダーは、あるユーザーが他のユーザーから認証された範囲外のキャッシュ/バッチデータをロードできるバグのリスクを回避するために、リクエストごとに作成する必要があります。
個々のリゾルバ内にデータローダを作成すると、バッチ処理の発生が妨げられ、 データローダの利点が損なわれることになります。

例えば、_あなたのコンテキストを宣言するとき_

```rust, ignore
use juniper;

#[derive(Clone)]
pub struct Context {
    pub cult_loader: CultLoader,
}

impl juniper::Context for Context {}

impl Context {
    pub fn new(cult_loader: CultLoader) -> Self {
        Self {
            cult_loader
        }
    }
}
```

_GraphQLのハンドラ(注意：ここでコンテキストをインスタンス化すると、リクエストごとに保持されます)_

```rust, ignore
pub async fn graphql(
    st: web::Data<Arc<Schema>>,
    data: web::Json<GraphQLRequest>,
) -> Result<HttpResponse, Error> {

    // コンテキストの設定.
    let cult_loader = get_loader();
    let ctx = Context::new(cult_loader);

    // 実行.
    let res = data.execute(&st, &ctx).await; 
    let json = serde_json::to_string(&res).map_err(error::ErrorInternalServerError)?;

    Ok(HttpResponse::Ok()
        .content_type("application/json")
        .body(json))
}
```

### さらなる例

`Dataloaders` と `Context` を使った完全な例は [jayy-lmao/rust-graphql-docker](https://github.com/jayy-lmao/rust-graphql-docker) を参照してください。