# ヒント機構のリファクタリング（PR #18090）

このドキュメントは [astral-sh/uv#18090](https://github.com/astral-sh/uv/pull/18090) の内容を解説します。
このリポジトリは astral-sh/uv のフォークであり、このドキュメントが解説する変更はすでにこのリポジトリのコードに適用されています。
このPRは、エラーメッセージに付随する「ヒント（hint）」の扱いを抽象化・統一するリファクタリングです。

## 背景：変更前の問題点

変更前は、各エラー型の `Display` 実装の中でヒントを直接書き込んでいました。
例えば `uv-build-frontend/src/error.rs` では以下のようなコードがありました：

```rust
// 変更前 (BuildBackendError の Display)
impl Display for BuildBackendError {
    fn fmt(&self, f: &mut Formatter<'_>) -> std::fmt::Result {
        write!(f, "{} ({})", self.message, self.exit_code)?;
        // ... stdout/stderr の出力 ...

        // ヒントがエラー本文に直接埋め込まれていた
        write!(
            f,
            "\n{}{} This usually indicates a problem with the package or the build environment.",
            "hint".bold().cyan(),
            ":".bold()
        )?;
        Ok(())
    }
}
```

この方法には以下の問題がありました：

- ヒントがエラーチェーンの途中に埋め込まれるため、表示位置が一定でない
- ヒントのフォーマット（`hint:` プレフィックス）が各エラー型に分散している
- ヒントをプログラム的に取得・操作できない

## 変更の概要

PR #18090 では以下の変更が行われました：

1. **新しい `uv-errors` クレートの追加**：ヒント処理の中核となる型とトレイトを定義
2. **各エラー型への `Hint` トレイトの実装**：ヒントをエラー本文から分離
3. **中央集権的なヒント収集**：`diagnostics.rs` でエラーチェーンを走査しヒントを一元管理

## `uv-errors` クレート

新たに追加された `crates/uv-errors/src/lib.rs` には以下の型が定義されています。

### `Hint` トレイト

```rust
pub trait Hint {
    fn hints(&self) -> Hints<'_> {
        Hints::none()
    }
}
```

ヒントを提供したいエラー型はこのトレイトを実装します。
デフォルト実装は空のヒントを返すため、ヒントが不要な型は `Hints::none()` を返すだけです。

### `Hints<'a>` 構造体

```rust
pub struct Hints<'a>(Vec<Cow<'a, str>>);
```

ユーザー向けヒントメッセージのコレクションです。主なメソッド：

| メソッド | 説明 |
| :------- | :--- |
| `Hints::none()` | 空のヒントコレクションを作成 |
| `Hints::from("...")` | 文字列リテラルから単一ヒントを作成 |
| `push(hint: String)` | ヒントを追加 |
| `extend(other: Hints<'_>)` | 別のヒントコレクションを結合（重複排除あり） |
| `is_empty()` | ヒントが空かどうか確認 |
| `into_owned()` | ライフタイムを `'static` に変換 |

`Display` 実装により、各ヒントは `\nhint: <メッセージ>` の形式で出力されます。

### `ErrorWithHints<'a, E>` 構造体

```rust
pub struct ErrorWithHints<'a, E> {
    error: E,
    hints: Hints<'a>,
}
```

エラーとヒントをまとめて表示するためのアダプターです。
テストや特定の表示箇所で、エラー本文の後にヒントを続けて表示するために使います：

```rust
let formatted = ErrorWithHints::new(error_message, err.hints()).to_string();
```

### `HintPrefix` 構造体

```rust
pub struct HintPrefix;

impl fmt::Display for HintPrefix {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}{}", "hint".bold().cyan(), ":".bold())
    }
}
```

ヒントのプレフィックス（`hint:`）をスタイル付きで表示します。

## 各エラー型への `Hint` トレイト実装

### `uv-build-frontend`

`crates/uv-build-frontend/src/error.rs` では、`Error` に `Hint` を実装しています：

```rust
impl Hint for Error {
    fn hints(&self) -> Hints<'_> {
        match self {
            Self::BuildBackend(_) => Hints::from(
                "Build failures usually indicate a problem with the package or the build environment",
            ),
            Self::MissingHeader(err) => Hints::from(err.cause.to_string()),
            Self::Lowering(err) => err.hints(),
            Self::RequirementsResolve(_, err) | Self::RequirementsInstall(_, err) => err.hints(),
            _ => Hints::none(),
        }
    }
}
```

対応して、`BuildBackendError::fmt` と `MissingHeaderError::fmt` からは
ヒントの書き込み処理が削除されました：

```rust
// 変更後 (BuildBackendError の Display)
impl Display for BuildBackendError {
    fn fmt(&self, f: &mut Formatter<'_>) -> std::fmt::Result {
        write!(f, "{} ({})", self.message, self.exit_code)?;
        // ... stdout/stderr の出力 ...
        // ヒントは書かない！ → Hint トレイト実装に移動
        Ok(())
    }
}
```

### `uv-resolver`

```rust
impl uv_errors::Hint for ResolveError {
    fn hints(&self) -> uv_errors::Hints<'_> {
        match self {
            Self::NoSolution(no_solution) => uv_errors::Hint::hints(no_solution.as_ref()),
            _ => uv_errors::Hints::none(),
        }
    }
}
```

解決エラーは `NoSolutionError` のヒントを委譲します。

### `uv-distribution`

```rust
impl uv_errors::Hint for Error {
    fn hints(&self) -> uv_errors::Hints<'_> {
        match self {
            Self::Build(err) => err.hints(),
            Self::MetadataLowering(err) => err.hints(),
            _ => uv_errors::Hints::none(),
        }
    }
}
```

### `uv/src/commands/pip/install.rs`

```rust
impl Hint for ExternallyManagedError {
    fn hints(&self) -> Hints<'_> {
        if self.system {
            Hints::from("Virtual environments were not considered due to the `--system` flag")
        } else {
            Hints::from("Consider creating a virtual environment, e.g., with `uv venv`")
        }
    }
}
```

### `uv/src/commands/tool/common.rs`

```rust
impl Hint for NoExecutablesError {
    fn hints(&self) -> Hints<'_> {
        // 複数のヒントを状況に応じて構築
        let mut hints = Hints::none();
        // ...
        hints
    }
}
```

## 中央集権的なヒント収集：`hints_for_error`

`crates/uv/src/commands/diagnostics.rs` に追加された `hints_for_error` 関数が、
ヒント収集の中心的な役割を担います：

```rust
pub(crate) fn hints_for_error(err: &anyhow::Error) -> Hints<'static> {
    let mut hints = Hints::none();
    for cause in err.chain() {
        collect_hint::<Box<uv_resolver::NoSolutionError>>(cause, &mut hints);
        collect_hint::<uv_resolver::NoSolutionError>(cause, &mut hints);
        collect_hint::<uv_resolver::ResolveError>(cause, &mut hints);
        collect_hint::<uv_resolver::LockError>(cause, &mut hints);
        collect_hint::<pip::operations::Error>(cause, &mut hints);
        // ... 他の多くの型 ...
        collect_hint::<uv_build_frontend::Error>(cause, &mut hints);
        collect_hint::<uv_python::Error>(cause, &mut hints);
        // ...
    }
    hints
}
```

`collect_hint` 関数はダウンキャストを使って、エラーチェーンの各要素が
`Hint` を実装する既知の型かどうかを確認し、ヒントを収集します：

```rust
fn collect_hint<T: Hint + std::error::Error + 'static>(
    cause: &(dyn std::error::Error + 'static),
    hints: &mut Hints<'static>,
) {
    if let Some(inner) = cause.downcast_ref::<T>() {
        hints.extend(inner.hints());
    }
}
```

この設計により：

- エラーがどの深さにあってもヒントを収集できる
- エラー本文の表示が完了した後にヒントを一括表示できる
- 新しいエラー型の追加が容易（`Hint` を実装して `collect_hint` に型を追加するだけ）

## テストでの活用：`format_error_with_hints`

`uv-build-frontend/src/error.rs` のテストでは、`ErrorWithHints` を使ってエラーとヒントをまとめて
文字列化するヘルパー関数が導入されました：

```rust
fn format_error_with_hints(err: &Error) -> String {
    // Unix/Windows のエラーコード表記を統一
    let formatted = std::error::Error::source(err)
        .unwrap()
        .to_string()
        .replace("exit status: ", "exit code: ");
    let formatted = ErrorWithHints::new(formatted, err.hints()).to_string();
    anstream::adapter::strip_str(&formatted).to_string()
}
```

これにより、テストのスナップショットで「エラー本文 → ヒント」の順序を確認できます：

```
Failed building wheel through setup.py (exit code: 0)

[stderr]
...
pygraphviz/graphviz_wrap.c:3020:10: fatal error: graphviz/cgraph.h: No such file or directory
...

hint: This error likely indicates that you need to install a library that provides "graphviz/cgraph.h" for `pygraphviz-1.11`
```

## まとめ：変更の効果

| 観点 | 変更前 | 変更後 |
| :--- | :----- | :----- |
| ヒントの位置 | エラー本文の末尾（エラーチェーン内） | エラーチェーン表示後に一括表示 |
| ヒントのフォーマット | 各エラー型が個別に実装 | `Hints` の `Display` に統一 |
| ヒントの取得 | 不可（表示のみ） | `Hint::hints()` でプログラム的に取得可能 |
| ヒントの重複排除 | なし | `Hints::extend` が自動排除 |
| 新しい型の追加 | 各型が独自にフォーマット | `Hint` トレイトを実装するだけ |

このリファクタリングにより、ヒントは「エラーメッセージの付属物」から
「エラー型が持つ第一級の情報」へと昇格しました。
