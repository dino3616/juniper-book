# [Japanese Translation of Juniper Book](https://dino3616.github.io/Japanese-Translation-of-Juniper-Book/introduction.html)

こちらはRust用GraphQL APIである[Juniper](https://github.com/graphql-rust/juniper)が提供する[Book](https://graphql-rust.github.io)の非公式な日本語訳です。

## Note

**これは非公式であり、常に最新版であるとは限らないことに注意してください。**

## Contributing

### Requirements

このサイトは、[mdBook](https://github.com/rust-lang-nursery/mdBook)を使って作られています。

以下のコマンドでインストールできます。

```bash
cargo install mdbook
```

### Starting a local test server

継続的にページを再構築し、自動で再読み込みするローカルのテストサーバーを起動するには、次のように実行します。

```bash
mdbook serve
```

### Building the book

以下のコマンドで、レンダリングされたHTMLにページをビルドすることができます。

```bash
mdbook build
```

出力は ~~`./_rendered`ディレクトリ~~ `./book` ディレクトリになります。

### Running the tests

~~本書に掲載されているすべてのExampleを検証するテストを実行するには、以下を実行します。~~

<!--
```bash
cd ./tests
cargo test
```
-->

## Test setup

~~本サイトに掲載されているRustのExampleはすべてCI上でコンパイルされています。~~

~~これは、[skeptic](https://github.com/budziq/rust-skeptic)ライブラリを使用して実行されます。~~