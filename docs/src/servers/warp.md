# Warp

[Warp]は、超簡単でコンポーザブルな、ワープスピードのためのウェブサーバフレームワークです。
warpの基本的な構成要素はFilterです：それらは組み合わせたり、構成することでリクエストに対する豊富な要求を表現することができます。Warpは[Hyper]をベースに構築されており、Rustの安定したチャンネルで動作します。

JuniperのWarp統合は、[`juniper_warp`][juniper_warp]クレートに含まれています。

```toml
[dependencies]
juniper = "0.15.7"
juniper_warp = "0.7.0"
```

ソースには、基本的な GraphQL と [GraphiQL] ハンドラをセットアップする [example][example] が含まれています。

[graphiql]: https://github.com/graphql/graphiql
[hyper]: https://hyper.rs/
[warp]: https://crates.io/crates/warp
[juniper_warp]: https://github.com/graphql-rust/juniper/tree/master/juniper_warp
[example]: https://github.com/graphql-rust/juniper/blob/master/juniper_warp/examples/warp_server.rs