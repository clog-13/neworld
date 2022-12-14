# 泛型 Generics

我们在编程中，经常有这样的需求：用同一功能的函数处理不同类型的数据，例如两个数的加法，无论是整数还是浮点数，甚至是自定义类型，都能进行支持。在不支持泛型的编程语言中，通常需要为每一种类型编写一个函数：

```rust
fn add_i8(a:i8, b:i8) -> i8 {
    a + b
}
fn add_i32(a:i32, b:i32) -> i32 {
    a + b
}
fn add_f64(a:f64, b:f64) -> f64 {
    a + b
}

fn main() {
    println!("add i8: {}", add_i8(2i8, 3i8));
    println!("add i32: {}", add_i32(20, 30));
    println!("add f64: {}", add_f64(1.23, 1.23));
}
```

在开始讲解 Rust 的泛型之前，先来看看什么是多态。

在编程的时候，我们经常利用多态。通俗的讲，多态就是好比坦克的炮管，既可以发射普通弹药，也可以发射制导炮弹（导弹），也可以发射贫铀穿甲弹，甚至发射子母弹，没有必要为每一种炮弹都在坦克上分别安装一个专用炮管。所以在编程开发中，我们也需要这样“通用的炮管”，这个“通用的炮管”就是多态。

实际上，**泛型就是一种多态**。

```rust
fn add<T>(a:T, b:T) -> T {
    a + b
}

fn main() {
    println!("add i8: {}", add(2i8, 3i8));
    println!("add i32: {}", add(20, 30));
    println!("add f64: {}", add(1.23, 1.23));
}
```

这段代码虽然很简洁，但是并不能编译通过，现在只要对泛型有个大概的印象即可。

## 泛型详解

上面代码的 `T` 就是**泛型参数**，泛型参数的名称你可以任意起，但是出于惯例，我们都用 `T` ( `T` 是 `type` 的首字母)来作为首选。

**使用泛型参数，有一个先决条件，必需在使用前对其进行声明**：

```rust
fn largest<T>(list: &[T]) -> T {}
```

该泛型函数的作用是从列表中找出最大的值，其中列表中的元素类型为 T。**首先 `largest<T>` 对泛型参数 `T` 进行了声明，然后才在函数参数中进行使用该泛型参数 `list: &[T]`** （`&[T]`： [数组切片](https://course.rs/basic/compound-type/array.html#数组切片)）。

总之，我们可以这样理解这个函数定义：函数 `largest` 有泛型类型 `T`，它有个参数 `list`，其类型是元素为 `T` 的数组切片，最后，该函数返回值的类型也是 `T`。

具体的泛型函数实现如下：

```rust
fn largest<T>(list: &[T]) -> T {
    let mut largest = list[0];

    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest(&number_list);
    println!("The largest number is {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest(&char_list);
    println!("The largest char is {}", result);
}
```

运行后报错：

```console
error[E0369]: binary operation `>` cannot be applied to type `T` // `>`操作符不能用于类型`T`
 --> src/main.rs:5:17
  |
5 |         if item > largest {
  |            ---- ^ ------- T
  |            |
  |            T
  |
help: consider restricting type parameter `T` // 考虑对T进行类型上的限制 :
  |
1 | fn largest<T: std::cmp::PartialOrd>(list: &[T]) -> T {
  |             ++++++++++++++++++++++
```

因为 `T` 可以是任何类型，但不是所有的类型都能进行比较，因此上面的错误中，编译器建议我们给 `T` 添加一个类型限制：**使用 `std::cmp::PartialOrd` 特征（Trait）对 `T` 进行限制**，该特征的目的就是让**类型实现可比较的功能**。

还记得我们一开始的 `add` 泛型函数吗？如果你运行它，会得到以下的报错：

```console
error[E0369]: cannot add `T` to `T` // 无法将 `T` 类型跟 `T` 类型进行相加
 --> src/main.rs:2:7
  |
2 |     a + b
  |     - ^ - T
  |     |
  |     T
  |
help: consider restricting type parameter `T`
  |
1 | fn add<T: std::ops::Add<Output = T>>(a:T, b:T) -> T {
  |         +++++++++++++++++++++++++++
```

**同样的，不是所有 `T` 类型都能进行相加操作，因此需要用 `std::ops::Add<Output = T>` 对 `T` 进行限制**：

```rust
fn add<T: std::ops::Add<Output = T>>(a:T, b:T) -> T {
    a + b
}

fn largest<T: std::cmp::PartialOrd + Copy>(list: &[T]) -> T {
    let mut largest: T = list[0];
    for &item: T in list.iter() {
        if item > largest {
            largest = item;
        }
    }
    largest
}
或者
fn largest<T: std::cmp::PartialOrd> (list: &[T]) -> &T {
    let mut largest: &T =  &list[0];
    for item: &T in list.iter(){
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

## 结构体中使用泛型

结构体中的字段类型也可以用泛型来定义，下面代码定义了一个坐标点 `Point`，它可以存放任何类型的坐标值：

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let integer = Point { x: 5, y: 10 };
    let float = Point { x: 1.0, y: 4.0 };
}
```

这里需要特别的注意：

- **提前声明**，跟泛型函数定义类似，首先我们在使用泛型参数之前必需要进行声明 `Point<T>`，接着就可以在结构体的字段类型中使用 `T` 来替代具体的类型

如果想让 `x` 和 `y` 类型不同，就需要使用不同的泛型参数：

```rust
struct Point<T,U> {
    x: T,
    y: U,
}
fn main() {
    let p = Point{x: 1, y :1.1};
}
```

## 枚举中使用泛型

提到枚举类型，`Option` 永远是第一个应该被想起来的：

```rust
enum Option<T> {
    Some(T),
    None,
}
```

`Option<T>` 是一个拥有泛型 `T` 的枚举类型，它第一个成员是 `Some(T)`，存放了一个类型为 `T` 的值。得益于泛型的引入，我们可以在任何一个需要返回值的函数中，去使用 `Option<T>` 枚举类型来做为返回值，用于返回一个任意类型的值 `Some(T)`，或者没有值 `None`。

对于枚举而言，如果是 `Option` 是卧龙，那么 `Result` 就一定是凤雏，得两者可得天下：

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

这个枚举和 `Option` 一样，主要用于函数返回值，与 `Option` 用于值的存在与否不同，`Result `关注的主要是值的正确性。

如果函数正常运行，则最后返回一个 `Ok(T)`，`T` 是函数具体的返回值类型，如果函数异常运行，则返回一个 `Err(E)`，`E` 是错误类型。例如打开一个文件：如果成功打开文件，则返回 `Ok(std::fs::File)`，因此 `T` 对应的是 `std::fs::File` 类型；而当打开文件时出现问题时，返回 `Err(std::io::Error)`，`E` 对应的就是 `std::io::Error` 类型。

## 方法中使用泛型

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

fn main() {
    let p = Point { x: 5, y: 10 };

    println!("p.x = {}", p.x());
}
```

使用泛型参数前，依然需要提前声明：`impl<T>`，只有提前声明了，我们才能在`Point<T>`中使用它，**这样 Rust 就知道 `Point` 的尖括号中的类型是泛型而不是具体类型**。需要注意的是，这里的 `Point<T>` 不再是泛型声明，而是一个完整的结构体类型，因为**我们定义的结构体就是 `Point<T>` 而不再是 `Point`**。

除了结构体中的泛型参数，我们还能在该结构体的方法中定义额外的泛型参数，就跟泛型函数一样：

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

impl<T, U> Point<T, U> {
    fn mixup<V, W>(self, other: Point<V, W>) -> Point<T, W> {
        Point {
            x: self.x,
            y: other.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 5, y: 10.4 };
    let p2 = Point { x: "Hello", y: 'c'};

    let p3 = p1.mixup(p2);
    // `p1.mixup(p2)`可以理解为`mixup(p1, p2)`，这也是实际的调用方式，只不过后者的调用形式是编译器作用后的结果，即我们以` p1.mixup(p2)`的形式通过编译器完成了函数调用`mixup(p1, p2)`，这和C++中是一样的。
	// self是由编译器传入的，和其它OOP里的self或this类似，区别只是显式或隐式传入。

    println!("p3.x = {}, p3.y = {}", p3.x, p3.y);
}
```

这个例子中，`T,U` 是定义在结构体 `Point` 上的泛型参数，`V,W` 是单独定义在方法 `mixup` 上的泛型参数，它们并不冲突，说白了，你可以理解为，一个是结构体泛型，一个是函数泛型。

`p1.mixup(p2)`可以理解为`mixup(p1, p2)`，这也是实际的调用方式，只不过后者的调用形式是编译器作用后的结果，即我们以` p1.mixup(p2)`的形式通过编译器完成了函数调用`mixup(p1, p2)`，这和C++中是一样的。

self是由编译器传入的，和其它OOP里的self或this类似，区别只是显式或隐式传入。

#### 为具体的泛型类型实现方法

对于 `Point<T>` 类型，你不仅能定义基于 `T` 的方法，还能针对特定的具体类型，进行方法定义：

```rust
impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

这段代码意味着 `Point<f32>` 类型会有一个方法 `distance_from_origin`，而其他 `T` 不是 `f32` 类型的 `Point<T> `实例则没有定义此方法。这个方法计算点实例与坐标`(0.0, 0.0)` 之间的距离，并使用了只能用于浮点型的数学运算符。

这样我们就能针对特定的泛型类型实现某个特定的方法，对于其它泛型类型则没有定义该方法。

## const 泛型（Rust 1.51 版本引入的重要特性）

在之前的泛型中，可以抽象为一句话：针对类型实现的泛型，所有的泛型都是为了抽象不同的类型，那有没有针对值的泛型？

在[数组](https://course.rs/basic/compound-type/array.html)那节，有提到过很重要的一点：`[i32; 2]` 和 `[i32; 3]` 是不同的数组类型，比如下面的代码：

```rust
fn display_array(arr: [i32; 3]) {
    println!("{:?}", arr);
}
fn main() {
    let arr: [i32; 3] = [1, 2, 3];
    display_array(arr);

    let arr: [i32;2] = [1,2];
    display_array(arr);
}
```

运行后报错：

```console
error[E0308]: mismatched types // 类型不匹配
  --> src/main.rs:10:19
   |
10 |     display_array(arr);
   |                   ^^^ expected an array with a fixed size of 3 elements, found one with 2 elements
                          // 期望一个长度为3的数组，却发现一个长度为2的
```

结合代码和报错，`[i32; 3]` 和 `[i32; 2]` 确实是两个完全不同的类型，因此无法用同一个函数调用。

首先，让我们修改代码，让 `display_array` 能打印任意长度的 `i32` 数组：

```rust
fn display_array(arr: &[i32]) {
    println!("{:?}", arr);
}
fn main() {
    let arr: [i32; 3] = [1, 2, 3];
    display_array(&arr);

    let arr: [i32;2] = [1,2];
    display_array(&arr);
}
```

很简单，只要使用数组切片，然后传入 `arr` 的不可变引用即可。

接着，将 `i32` 改成所有类型的数组：

```rust
fn display_array<T: std::fmt::Debug>(arr: &[T]) {
    println!("{:?}", arr);
}
fn main() {
    let arr: [i32; 3] = [1, 2, 3];
    display_array(&arr);

    let arr: [i32;2] = [1,2];
    display_array(&arr);
}
```

也不难，唯一要注意的是需要对 `T` 加一个限制 `std::fmt::Debug`，该限制表明 `T` 可以用在 `println!("{:?}", arr)` 中，因为 `{:?}` 形式的格式化输出需要 `arr` 实现该特征。

通过引用，我们可以很轻松的解决处理任何类型数组的问题，但是如果在某些场景下引用不适宜用或者干脆不能用呢？你们知道为什么以前 Rust 的一些数组库，在使用的时候都限定长度不超过 32 吗？因为它们会为每个长度都单独实现一个函数，简直。。。毫无人性。难道没有什么办法可以解决这个问题吗？

好在，现在咱们有了 const 泛型，也就是针对值的泛型，正好可以用于处理数组长度的问题：

```rust
fn display_array<T: std::fmt::Debug, const N: usize>(arr: [T; N]) {
    println!("{:?}", arr);
}
fn main() {
    let arr: [i32; 3] = [1, 2, 3];
    display_array(arr);

    let arr: [i32; 2] = [1, 2];
    display_array(arr);
}
```

如上所示，我们定义了一个类型为 `[T; N]` 的数组，其中 `T` 是一个基于类型的泛型参数，这个和之前讲的泛型没有区别，而重点在于 `N` 这个泛型参数，它是一个基于值的泛型参数！因为它用来替代的是数组的长度。

`N` 就是 const 泛型，定义的语法是 `const N: usize`，表示 const 泛型 `N` ，它基于的值类型是 `usize`。

在泛型参数之前，Rust 完全不适合复杂矩阵的运算。

#### const 泛型表达式

假设我们某段代码需要在内存很小的平台上工作，因此需要限制**函数参数**占用的内存大小，此时就可以使用 const 泛型表达式来实现：

```rust
// 目前只能在nightly版本下使用
#![allow(incomplete_features)]
#![feature(generic_const_exprs)]

fn something<T>(val: T) where
    Assert<{ core::mem::size_of::<T>() < 768 }>: IsTrue,
    //       ^-----------------------------^ 这里是一个 const 表达式，换成其它的 const 表达式也可以
{
    // ...
}
// 1. core::mem::size_of::<T>() 这里返回一个类型占用内存的大小值
// 2. { core::mem::size_of::() < 768 } 这里是一个表达式，表达式返回一个bool值
// 类似于：let x = {2 < 3};
// 3. <{ core::mem::size_of::<T>() < 768 }>: IsTrue 这句就是约束，这个内存大小一定要小于768（返回True）

fn main() {
    something([0u8; 0]); // ok
    something([0u8; 512]); // ok
    something([0u8; 1024]); // 编译错误，数组长度是1024字节，超过了768字节的参数长度限制
}

// ---

pub enum Assert<const CHECK: bool> {
    //
}

pub trait IsTrue {
    //
}

impl IsTrue for Assert<true> {
    //
}
```

`N`是可以直接在函数体里使用的。

```
fn display_array<T: std::fmt::Debug, const N: usize>(arr: [T; N]) {
    println!("{}",N)
}
```

#### const fn

#todo

## 泛型的性能

在 Rust 中泛型是零成本的抽象，在使用泛型时完全不用担心性能上的问题。

但是任何选择都是权衡得失的，既然我们获得了性能上的巨大优势，那么又失去了什么呢？Rust 是在编译期为泛型对应的多个类型，生成各自的代码，因此**损失了编译速度和增大了最终生成文件的大小**。

Rust 通过在编译时进行泛型代码的 **单态化**(*monomorphization*)来保证效率。单态化是一个通过填充编译时使用的具体类型，将通用代码转换为特定代码的过程。

编译器所做的工作正好与我们创建泛型函数的步骤相反，编译器寻找所有泛型代码被调用的位置并针对具体类型生成代码。

让我们看看一个使用标准库中 `Option` 枚举的例子：

```rust
let integer = Some(5);
let float = Some(5.0);
```

当 Rust 编译这些代码的时候，它会进行单态化。编译器会读取传递给 `Option<T>` 的值并发现有两种 `Option<T>`：一种对应 `i32` 另一种对应 `f64`。为此，它会将泛型定义 `Option<T>` 展开为 `Option_i32` 和 `Option_f64`，接着将泛型定义替换为这两个具体的定义。

编译器生成的单态化版本的代码看起来像这样：

```rust
enum Option_i32 {
    Some(i32),
    None,
}

enum Option_f64 {
    Some(f64),
    None,
}

fn main() {
    let integer = Option_i32::Some(5);
    let float = Option_f64::Some(5.0);
}
```

我们可以使用泛型来编写不重复的代码，而 Rust 将会为每一个实例编译其特定类型的代码。这意味着在使用泛型时没有运行时开销；当代码运行，它的执行效率就跟好像手写每个具体定义的重复代码一样。这个单态化过程正是 Rust 泛型在运行时极其高效的原因。

## 习题

```
struct A;          // Concrete type `A`.
struct S(A);       // Concrete type `S`.
struct SGen<T>(T); // Generic type `SGen`.

fn reg_fn(_s: S) {}

fn gen_spec_t(_s: SGen<A>) {}

fn gen_spec_i32(_s: SGen<i32>) {}

fn generic<T>(_s: SGen<T>) {}

fn main() {
    // Using the non-generic functions
    let temp_a = A; // 实例化A
    let temp_s = S(temp_a); // 实例化S
    // 调用
    reg_fn(temp_s);
    reg_fn(S(A));          // Concrete type.
    gen_spec_t(SGen(A));   // Implicitly specified type parameter `A`.
    gen_spec_i32(SGen(6)); // Implicitly specified type parameter `i32`.

    // Explicitly specified type parameter `char` to `generic()`.
    generic::<char>(SGen('a'));

    // Implicitly specified type parameter `char` to `generic()`.
    generic(SGen('c'));
}
```

```
struct Val<T> {
    val: T,
}

impl<T> Val<T> {
    fn value(&self) -> &T {
        &self.val
    }
}


fn main() {
    let x = Val{ val: 3.0 };
    let y = Val{ val: "hello".to_string()};
    println!("{}, {}", x.value(), y.value());
}
```

```
struct Point<T, U> {
    x: T,
    y: U,
}

impl<T, U> Point<T, U> {
    fn mixup<V, W>(self, other: Point<V, W>) -> Point<T, W> {
        Point {
            x: self.x,
            y: other.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 5, y: 10 };
    let p2 = Point { x: "Hello", y: '中'};

    let p3 = p1.mixup(p2);

    assert_eq!(p3.x, 5);
    assert_eq!(p3.y, '中');
}
```