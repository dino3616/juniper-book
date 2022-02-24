# Rocket

[Rocket][rocket] は、RustのためのWebフレームワークで、柔軟性や型安全性を犠牲にすることなく、高速なWebアプリケーションを簡単に書くことができるようにするものです。すべて最小限のコードで。RocketはRustの安定版チャンネルでは動作せず、代わりにナイトリーチャンネルが必要です。

JuniperのRocket統合は、[`juniper_rocket`][juniper_rocket]クレートに含まれています。

```toml
[dependencies]
juniper = "0.15.7"
juniper_rocket = "0.8.0"
```

ソースには、基本的な GraphQL と [GraphiQL] ハンドラをセットアップする [example][example] が含まれています。

[graphiql]: https://github.com/graphql/graphiql
[rocket]: https://rocket.rs/
[juniper_rocket]: https://github.com/graphql-rust/juniper/tree/master/juniper_rocket
[example]: https://github.com/graphql-rust/juniper/blob/master/juniper_rocket/examples/rocket_server.rs