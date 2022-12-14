# Crate 和 Package

Rust 为我们提供了强大的包管理工具：

- **项目(Package)**：可以用来构建、测试和分享包
- **工作空间(WorkSpace)**：对于大型项目，可以进一步将多个包联合在一起，组织成工作空间
- **包(Crate)**：一个由多个模块组成的树形结构，可以作为三方库进行分发，也可以生成可执行文件进行运行
- **模块(Module)**：可以一个文件多个模块，也可以一个文件一个模块，模块可以被认为是真实项目中的代码组织单元

## 定义

#### Crate（包）

对于 Rust 而言，crate 是一个独立的**可编译单元**，**它编译后会生成一个可执行文件或者一个库**。

一个 crate 会**将相关联的功能打包在一起**，使得该功能可以很方便的在多个项目中分享。例如标准库中没有提供但是在三方库中提供的 `rand` crate（包），它提供了随机数生成的功能，我们只需要将该 crate 通过 `use rand;` 引入到当前项目的作用域中，就可以在项目中使用 `rand` 的功能：`rand::XXX`。

同一个包中不能有同名的类型，但是在不同包中就可以。例如，虽然 `rand` 包中，有一个 `Rng` 特征，可是我们依然可以在自己的项目中定义一个 `Rng`，前者通过 `rand::Rng` 访问，后者通过 `Rng` 访问，对于编译器而言，这两者的边界非常清晰，不会存在歧义。

## 项目 Package

`Package` 翻译成项目，你也可以理解为工程、软件包。

由于 `Package` 就是一个项目，因此它包含有独立的 `Cargo.toml` 文件，以及因为功能性被组织在一起的一个或多个**包(Crate)**。一个 `Package` 只能包含**一个库(library)类型的包**，但是可以包含**多个二进制可执行类型的包(bin crate)**。

#### 二进制 Package

创建一个二进制 `Package`：

```console
$ cargo new my-project
     Created binary (application) `my-project` package
$ ls my-project
Cargo.toml
src
$ ls my-project/src
main.rs
```

**`src/main.rs` 是二进制包的根文件，该二进制包的包名跟所属 `Package` 相同，在这里都是 `my-project`**，所有的代码执行都从该文件中的 `fn main()` 函数开始。

#### 库 Package

再来创建一个库类型的 `Package`：

```console
$ cargo new my-lib --lib
     Created library `my-lib` package
$ ls my-lib
Cargo.toml
src
$ ls my-lib/src
lib.rs
```

首先，如果你试图运行 `my-lib`，会报错：

```console
$ cargo run
error: a bin target must be available for `cargo run`
```

原因是**库类型的 `Package` 只能作为三方库被其它项目引用，而不能独立运行，只有之前的二进制 `Package` 才可以运行**。

与 `src/main.rs` 一样，Cargo 知道，如果一个 `Package` 包含有 `src/lib.rs`，意味它包含有一个库类型的同名包 `my-lib`，该包的根文件是 `src/lib.rs`。

#### 易混淆的 Package 和包

看完上面，相信大家看出来为何 `Package` 和包容易被混淆了吧？因为你用 `cargo new` 创建的 `Package` 和它其中包含的 carte（包）是同名的！

**`Package` 是一个项目工程，而包只是一个编译单元**，基本上也就不会混淆这个两个概念了：**`src/main.rs` 和 `src/lib.rs` 都是编译单元，因此它们都是包**。

#### 典型的 `Package` 结构

上面创建的 `Package` 中仅包含 `src/main.rs` 文件，**意味着它仅包含一个二进制同名包 `my-project`**。如果一个 `Package` 同时拥有 `src/main.rs` 和 `src/lib.rs`，**那就意味着它包含两个包：库包和二进制包，这两个包名也都是 `my-project`** —— 都与 `Package` 同名。

一个真实项目中典型的 `Package`，**会包含多个二进制包**，这些包文件被放在 `src/bin` 目录下，**每一个文件都是独立的二进制包**，**同时也会包含一个库包，该包只能存在一个 `src/lib.rs`**：

```css
.
├── Cargo.toml
├── Cargo.lock
├── src
│   ├── main.rs
│   ├── lib.rs
│   └── bin
│       └── main1.rs
│       └── main2.rs
├── tests
│   └── some_integration_tests.rs
├── benches
│   └── simple_bench.rs
└── examples
    └── simple_example.rs
```

- 唯一库包：`src/lib.rs`
- 默认二进制包：`src/main.rs`，**编译后生成的可执行文件与 `Package` 同名**
- 其余二进制包：`src/bin/main1.rs` 和 `src/bin/main2.rs`，它们会分别生成一个文件同名的二进制可执行文件
- 集成测试文件：`tests` 目录下
- 基准性能测试 `benchmark` 文件：`benches` 目录下
- 项目示例：`examples` 目录下

这种目录结构基本上是 Rust 的标准目录结构。
