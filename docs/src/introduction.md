# Juniper

Juniper は、Rust 用の [GraphQL] サーバーライブラリです。型安全で高速なAPIサーバーを、最小限の定型文と設定で構築できます。

[GraphQL][graphql]は、Facebookが開発したデータクエリ言語です。
モバイルおよびWebアプリケーションのフロントエンドにサービスを提供します。

Juniper は、Rust で GraphQL サーバを型安全かつ高速に記述することを可能にします。また、Rust が許す限り、GraphQL のスキーマの宣言と解決をできるだけ便利にすることを心がけています。

Juniper にはウェブサーバーは含まれていません。代わりに、既存のサーバーとの統合を容易にするためのビルディングブロックを提供しています。オプションで、[Hyper][hyper]、[Iron][iron]、[Rocket]、[Warp]フレームワーク用の組み込み済みインテグレーションを提供し、簡単にデバッグできるように[Graphiql][graphiql]も組み込まれています。

* [Cargo crate](https://crates.io/crates/juniper)
* [APIリファレンス][docsrs]

## 特徴

ジュニパーは、インターフェース、ユニオン、スキーマイントロスペクション、検証を含む、[仕様][graphql_spec]に準拠したGraphQLクエリ言語をフルサポートしています。
しかし、スキーマ言語はサポートしていません。

他の言語向けGraphQLライブラリーの例外として、Juniperはデフォルトで非NULL型を構築します。`Vec<Episode>` 型のフィールドは、 `[Episode!]!` に変換されます。例えば `[Episode]` に対応する Rust 型は `Option<Vec<Option<Episode>>` です。

## インテグレーション

### データ型

Juniperは、スキーマの構築を容易にするために、非常に一般的なRustクレートを自動的に統合しています。これらのクレートの型は、スキーマの中で自動的に使用できるようになります。

* [uuid][uuid]
* [url][url]
* [chrono][chrono]
* [bson][bson]

### ウェブフレームワーク

* [hyper][hyper]
* [rocket][rocket]
* [iron][iron]
* [warp][warp]

## API の安定性

Juniperはまだ1.0に到達していないため、APIが不安定になることが予想されます。

[graphql]: http://graphql.org
[graphiql]: https://github.com/graphql/graphiql
[iron]: https://github.com/iron/iron
[graphql_spec]: http://facebook.github.io/graphql
[test_schema_rs]: https://github.com/graphql-rust/juniper/blob/master/juniper/src/tests/schema.rs
[tokio]: https://github.com/tokio-rs/tokio
[hyper_examples]: https://github.com/graphql-rust/juniper/tree/master/juniper_hyper/examples
[rocket_examples]: https://github.com/graphql-rust/juniper/tree/master/juniper_rocket/examples
[iron_examples]: https://github.com/graphql-rust/juniper/tree/master/juniper_iron/examples
[hyper]: https://hyper.rs
[rocket]: https://rocket.rs
[book]: https://graphql-rust.github.io
[book_quickstart]: https://graphql-rust.github.io/quickstart.html
[docsrs]: https://docs.rs/juniper
[warp]: https://github.com/seanmonstar/warp
[warp_examples]: https://github.com/graphql-rust/juniper/tree/master/juniper_warp/examples
[uuid]: https://crates.io/crates/uuid
[url]: https://crates.io/crates/url
[chrono]: https://crates.io/crates/chrono
[bson]: https://crates.io/crates/bson