---
title: "[Rust] 互換性チェックはcargo-semver-checksが便利"
emoji: "🚧"
type: "tech"
topics: ["rust", "cargo"]
published: true
---

cargoは[SemVer](https://semver.org/)を使用してバーション管理をしています．さらに[SemVer Compatibility](https://doc.rust-lang.org/cargo/reference/semver.html)にバーションの互換性を保つためのルールが文章化されています．

[SemVer Compatibility](https://doc.rust-lang.org/cargo/reference/semver.html)では変更を以下のように分類しています．

- Major change
  SemVerでMAJORを増やすような大規模，壊滅的な変更．関数，構造体の削除など．
- Minor change
  SemVerでMINOR，PATCHを増やすような互換性を保つ変更．関数，構造体の追加など．
- Possibly-breaking change
  プロジェクトによってMajor changeかMinor changeかどうか分かれる変更．Rustのバーション変更など．

例えば既存の関数の削除などの壊滅的な変更をしておいて，バーションを1.0.1から1.0.2に上げました！ みたいなことは許されないわけです．

これらはライブラリをアップデートするときに参考になるものですが1つ1つチェックしていくのは面倒です．そこで互換性ルールを自動でチェックする[cargo-semver-checks](https://github.com/obi1kenobi/cargo-semver-checks)が役に立ちます．

以下のようなケースを考えてみましょう．

あるクレートのバーション1.0.0では`Error`という名前の構造体がありました．

```rust:[1.0.0] lib.rs
pub struct Error {}
```

バーション1.1.0では`Error`を`ConfigError`という名前にして変更したいとします．しかし互換性を崩さないためにタイプエイリアスを追加しようとしましたが，うっかり名前を`Error`ではなく`Err`にしてしまいました．これでは互換性が壊れてしまいます．

```rust:[1.1.0 mistaken] lib.rs
pub struct ConfigError {}

#[deprecated(note = "Use `ConfigError` instead.")]
pub type Err = ConfigError;
```

これは人がチェックしていたらうっかり見落としていまう可能があります．そんなときでもcargo-semver-checksを使えばバッチシ弾いてくれるわけです．

```:error
--- failure enum_missing: pub enum removed or renamed ---

Description:
A publicly-visible enum cannot be imported by its prior path. A `pub use` may have been removed, or the enum itself may have been renamed or removed entirely.
        ref: https://doc.rust-lang.org/cargo/reference/semver.html#item-remove
       impl: https://github.com/obi1kenobi/cargo-semver-checks/tree/v0.23.0/src/lints/enum_missing.ron
```

修正したものは以下になります．

```rust:[1.1.0 correct] lib.rs
pub struct ConfigError {}

#[deprecated(note = "Use `ConfigError` instead.")]
pub type Error = ConfigError;
```

[GitHub Action](https://github.com/obi1kenobi/cargo-semver-checks-action)もあるので，とりあえず導入しておくと良いと思います．
