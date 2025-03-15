# PyO3 と maturin 入門ガイド

このプロジェクトは、PyO3 と maturin を使用して Rust のコードを Python から呼び出せるようにする方法を示す実践的なガイドです。サンプルとして、The Rust Book で紹介されている数当てゲームを実装しています。

## PyO3 とは

PyO3 は Rust から Python へのバインディングを提供するフレームワークです。これを使用すると、Rust で書いたコードを Python から呼び出したり、Python のオブジェクトを Rust で操作したりすることができます。

### 主な機能

- Rust 関数を Python 関数としてエクスポート (`#[pyfunction]`)
- Rust 構造体を Python クラスとしてエクスポート (`#[pyclass]`)
- Python の型と Rust の型の自動変換
- Python の例外処理と Rust のエラー処理の連携
- GIL (Global Interpreter Lock) の安全な管理

## プロジェクト構造

最小限の PyO3 プロジェクトは以下のファイルで構成されます：

```
project-name/
├── Cargo.toml        # Rust の依存関係と設定
├── pyproject.toml    # Python のビルドシステム設定
└── src/
    └── lib.rs        # Rust のソースコード
```

## 基本的な使い方

### 1. Rust 関数を Python で使用可能にする

`lib.rs` での基本的な実装：

```rust
use pyo3::prelude::*;

#[pyfunction]
fn sum_as_string(a: usize, b: usize) -> PyResult<String> {
    Ok((a + b).to_string())
}

/// Python モジュールの初期化
#[pymodule]
fn module_name(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(sum_as_string, m)?)?;
    Ok(())
}
```

### 2. Rust の構造体を Python のクラスとして公開

```rust
#[pyclass]
struct MyClass {
    #[pyo3(get, set)]
    value: i32,
}

#[pymethods]
impl MyClass {
    #[new]
    fn new(value: i32) -> Self {
        MyClass { value }
    }

    fn double(&self) -> i32 {
        self.value * 2
    }

    fn __repr__(&self) -> PyResult<String> {
        Ok(format!("MyClass(value={})", self.value))
    }
}
```

### 3. Python モジュールへの登録

```rust
#[pymodule]
fn module_name(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(sum_as_string, m)?)?;
    m.add_class::<MyClass>()?;
    Ok(())
}
```

## プロジェクト設定

### Cargo.toml

```toml
[package]
name = "module-name"
version = "0.1.0"
edition = "2021"

[lib]
name = "module_name"  # アンダースコアに注意
crate-type = ["cdylib"]

[dependencies.pyo3]
version = "0.24.0"
features = ["extension-module", "abi3-py38"]
```

### pyproject.toml

```toml
[build-system]
requires = ["maturin>=1.0,<2.0"]
build-backend = "maturin"

[tool.maturin]
features = ["pyo3/extension-module"]
```

## 型変換

PyO3 は以下の型の自動変換をサポートしています：

| Python 型 | Rust 型                           |
| --------- | --------------------------------- |
| `int`     | `i32`, `i64`, `u32`, `u64`, ...   |
| `float`   | `f32`, `f64`                      |
| `bool`    | `bool`                            |
| `str`     | `&str`, `String`                  |
| `bytes`   | `&[u8]`, `Vec<u8>`                |
| `list`    | `Vec<T>`                          |
| `dict`    | `HashMap<K, V>`, `BTreeMap<K, V>` |
| `tuple`   | `(T, U, ...)`                     |
| `None`    | `()`, `Option<T>`                 |

## ビルドとインストール

### maturin によるビルド

```bash
# 開発モードでインストール（コードの変更が即反映される）
maturin develop

# リリースモードでビルド
maturin build --release

# macOS の場合、ユニバーサルバイナリを作成
maturin build --release --universal2

# ARM Mac の場合
maturin build --release --target aarch64-apple-darwin

# Intel Mac の場合
maturin build --release --target x86_64-apple-darwin

# Windows の場合
maturin build --release --target x86_64-pc-windows-msvc
```

### インストール

```bash
pip install target/wheels/*.whl
```

## エラー処理

PyO3 では、Rust の `Result` 型を使って Python の例外を扱えます：

```rust
#[pyfunction]
fn divide(a: f64, b: f64) -> PyResult<f64> {
    if b == 0.0 {
        return Err(PyErr::new::<pyo3::exceptions::PyZeroDivisionError, _>("division by zero"));
    }
    Ok(a / b)
}
```

## 高度な機能

### GIL の管理

```rust
#[pyfunction]
fn cpu_intensive_task(py: Python, data: Vec<f64>) -> PyResult<f64> {
    // GIL を解放して並列処理を行う
    py.allow_threads(|| {
        // ここでは GIL なしで実行される
        data.iter().sum::<f64>()
    })
}
```

### Python オブジェクトの操作

```rust
#[pyfunction]
fn call_python_function(py: Python, func: &PyAny, arg: i32) -> PyResult<i32> {
    let result = func.call1((arg,))?;
    result.extract()
}
```

## サンプルアプリケーション

このリポジトリには、数当てゲームのサンプルがあります：

```rust
#[pyfunction]
fn guess_the_number() {
    println!("Guess the number!");

    let secret_number = rand::rng().random_range(1..101);

    loop {
        println!("Please input your guess.");

        let mut guess = String::new();

        io::stdin()
            .read_line(&mut guess)
            .expect("Failed to read line");

        let guess: u32 = match guess.trim().parse() {
            Ok(num) => num,
            Err(_) => continue,
        };

        println!("You guessed: {}", guess);

        match guess.cmp(&secret_number) {
            Ordering::Less => println!("Too small!"),
            Ordering::Greater => println!("Too big!"),
            Ordering::Equal => {
                println!("You win!");
                break;
            }
        }
    }
}
```

## プロジェクトのビルドと実行

```bash
# インストール
maturin develop

# Python から実行
python -c "import guessing_game; guessing_game.guess_the_number()"
```

## 参考リンク

- [PyO3 公式ドキュメント](https://pyo3.rs/)
- [maturin 公式ドキュメント](https://www.maturin.rs/)
- [PyO3 GitHub リポジトリ](https://github.com/PyO3/pyo3)
- [Rust のドキュメント](https://doc.rust-lang.org/book/)

このプロジェクトは、PyO3 と maturin を使用して Rust のパフォーマンスと Python の使いやすさを組み合わせる方法を学ぶための出発点として最適です。
