# 列挙型

GraphQLにおける列挙型は、可能な値の集合を表すためにグループ化された文字列定数です。単純なRustの列挙型は、カスタムのderive属性を使用することでGraphQLの列挙型に変換することができます。

```rust
# extern crate juniper;
#[derive(juniper::GraphQLEnum)]
enum Episode {
    NewHope,
    Empire,
    Jedi,
}

# fn main() {}
```

Juniper はすべての enum 変数を大文字に変換するので、対応する文字列の値はそれぞれ `NEWHOPE`、`EMPIRE`、`JEDI` となります。もし、これを上書きしたい場合は、`graphql`属性を使用します。これは、[オブジェクトの定義](objects/defining_objects.md)と同じように動作します。

```rust
# extern crate juniper;
#[derive(juniper::GraphQLEnum)]
enum Episode {
    #[graphql(name="NEW_HOPE")]
    NewHope,
    Empire,
    Jedi,
}

# fn main() {}
```

## ドキュメンテーションと非推奨

オブジェクトを定義するときと同じように、型そのものは名前を変えて文書化することができ、個々の列挙型の変種は名前を変えて文書化し、非推奨にすることができます。

```rust
# extern crate juniper;
#[derive(juniper::GraphQLEnum)]
#[graphql(name="Episode", description="An episode of Star Wars")]
enum StarWarsEpisode {
    #[graphql(deprecated="We don't really talk about this one")]
    ThePhantomMenace,

    #[graphql(name="NEW_HOPE")]
    NewHope,

    #[graphql(description="Arguably the best one in the trilogy")]
    Empire,
    Jedi,
}

# fn main() {}
```

| Name of Attribute | Container Support |  Field Support   |
| ----------------- | :---------------: | :--------------: |
| context           |         ✔         |        ?         |
| deprecated        |         ✔         |        ✔         |
| description       |         ✔         |        ✔         |
| interfaces        |         ?         |        ✘         |
| name              |         ✔         |        ✔         |
| no-async          |         ✔         |        ?         |
| scalar            |         ✘         |        ?         |
| skip              |         ?         |        ✘         |
| ✔: supported      | ✘: not supported  | ?: not available |