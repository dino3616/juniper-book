# 入力オブジェクト

入力オブジェクトは、GraphQLフィールドの引数として使用することができる複雑なデータ構造です。Juniperでは、単純なオブジェクトやenumと同様に、カスタムのderive属性を使用して入力オブジェクトを定義できます。

```rust
# #![allow(unused_variables)]
# extern crate juniper;
#[derive(juniper::GraphQLInputObject)]
struct Coordinate {
    latitude: f64,
    longitude: f64
}

struct Root;
# #[derive(juniper::GraphQLObject)] struct User { name: String }

#[juniper::graphql_object]
impl Root {
    fn users_at_location(coordinate: Coordinate, radius: f64) -> Vec<User> {
        // 座標をデータベースに送信する.
        // ...
# unimplemented!()
    }
}

# fn main() {}
```

## ドキュメンテーションとリネーム

他のderivesと同じように、型とフィールドの両方の名前を変更したり、ドキュメントを追加したりすることが可能です。

```rust
# #![allow(unused_variables)]
# extern crate juniper;
#[derive(juniper::GraphQLInputObject)]
#[graphql(name="Coordinate", description="A position on the globe")]
struct WorldCoordinate {
    #[graphql(name="lat", description="The latitude")]
    latitude: f64,

    #[graphql(name="long", description="The longitude")]
    longitude: f64
}

struct Root;
# #[derive(juniper::GraphQLObject)] struct User { name: String }

#[juniper::graphql_object]
impl Root {
    fn users_at_location(coordinate: WorldCoordinate, radius: f64) -> Vec<User> {
        // 座標をデータベースに送信する.
        // ...
# unimplemented!()
    }
}

# fn main() {}
```