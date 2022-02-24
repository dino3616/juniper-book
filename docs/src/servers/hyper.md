# Hyper

[Hyper]は、他の多くのRustウェブフレームワークが利用している高速なHTTP実装です。tokioランタイムによる非同期I/Oを提供し、Rustの安定したチャンネルで動作します。

Hyperは上位のWebフレームワークではないので、シンプルなエンドポイントルーティング、HTTPレスポンスの組み込み、再利用可能なミドルウェアなどの人間工学的な機能は含まれていません。GraphQLの場合、すべてのPOSTとGETは通常、いくつかの明確に定義されたレスポンスペイロードを持つ単一のエンドポイントを通過するため、これらは大きな欠点ではありません。

JuniperのHyper統合は、[`juniper_hyper`][juniper_hyper]クレートに含まれています。

```toml
[dependencies]
juniper = "0.15.7"
juniper_hyper = "0.8.0"
```

ソースには、基本的な GraphQL と [GraphiQL] ハンドラをセットアップする [example][example] が含まれています。

[graphiql]: https://github.com/graphql/graphiql
[hyper]: https://hyper.rs/
[juniper_hyper]: https://github.com/graphql-rust/juniper/tree/master/juniper_hyper
[example]: https://github.com/graphql-rust/juniper/blob/master/juniper_hyper/examples/hyper_server.rs