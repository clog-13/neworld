# &'static 和 T: 'static

`'static` 在 Rust 中是相当常见的，例如字符串字面值就具有 `'static` 生命周期:

```rust
fn main() {
  let mark_twain: &str = "Samuel Clemens";
  print_author(mark_twain);
}
fn print_author(author: &'static str) {
  println!("{}", author);
}
```

除此之外，特征对象的生命周期也是 `'static`，例如[这里](https://course.rs/compiler/fight-with-compiler/lifetime/closure-with-static.html#特征对象的生命周期)所提到的。

除了 `&'static` 的用法外，我们在另外一种场景中也可以见到 `'static` 的使用:

```rust
use std::fmt::Display;
fn main() {
    let mark_twain = "Samuel Clemens";
    print(&mark_twain);
}

fn print<T: Display + 'static>(message: &T) {
    println!("{}", message);
}
```

在这里，很明显 **`'static` 是作为生命周期约束来使用了**。

## `&'static`

`&'static` 对于生命周期有着非常强的要求：**一个引用必须要活得跟剩下的程序一样久，才能被标注为 `&'static`**。

对于字符串字面量来说，它直接被打包到二进制文件中，永远不会被 `drop`，因此它能跟程序活得一样久，自然它的生命周期是 `'static`。

但是，**`&'static` 生命周期针对的仅仅是引用，而不是持有该引用的变量，对于变量来说，还是要遵循相应的作用域规则** :

```rust
use std::{slice::from_raw_parts, str::from_utf8_unchecked};

fn get_memory_location() -> (usize, usize) {
  // “Hello World” 是字符串字面量，因此它的生命周期是 `'static`.
  // 但持有它的变量 `string` 的生命周期就不一样了，它完全取决于变量作用域，对于该例子来说，也就是当前的函数范围
  let string = "Hello World!";
  let pointer = string.as_ptr() as usize;
  let length = string.len();
  (pointer, length)
  // `string` 在这里被 drop 释放
  // 虽然变量被释放，无法再被访问，但是数据依然还会继续存活
}

fn get_str_at_location(pointer: usize, length: usize) -> &'static str {
  // 使用裸指针需要 `unsafe{}` 语句块
  unsafe { from_utf8_unchecked(from_raw_parts(pointer as *const u8, length)) }
}

fn main() {
  let (pointer, length) = get_memory_location();
  let message = get_str_at_location(pointer, length);
  println!(
    "The {} bytes at 0x{:X} stored: {}",
    length, pointer, message
  );
  // 如果大家想知道为何处理裸指针需要 `unsafe`，可以试着反注释以下代码
  // let message = get_str_at_location(1000, 10);
}
```

上面代码有两点值得注意：

- `&'static` 的引用确实可以和程序活得一样久，因为我们通过 `get_str_at_location` 函数直接取到了对应的字符串
- 持有 `&'static` 引用的变量，它的生命周期受到作用域的限制

## `T: 'static`

相比起来，这种形式的约束就有些复杂了。

首先，在以下两种情况下，`T: 'static` 与 `&'static` 有相同的约束：`T` 必须活得和程序一样久。

```rust
use std::fmt::Debug;

fn print_it<T: Debug + 'static>( input: T ) {
    println!( "'static value passed in is: {:?}", input );
}

fn print_it1( input: impl Debug + 'static ) {
    println!( "'static value passed in is: {:?}", input );
}

fn main() {
    let i = 5;

    print_it(&i);
    print_it1(&i);
}
```

以上代码会报错，原因很简单: `&i` 的生命周期无法满足 `'static` 的约束，如果大家将 `i` 修改为常量，那自然一切 OK。

来稍微修改下 `print_it` 函数:

```rust
use std::fmt::Debug;

fn print_it<T: Debug + 'static>( input: &T) {
    println!( "'static value passed in is: {:?}", input );
}

fn main() {
    let i = 5;

    print_it(&i);
}
```

这段代码竟然不报错了！原因在于我们约束的是 `T`，但是使用的却是它的引用 `&T`，换而言之，我们根本没有直接使用 `T`，因此编译器就没有去检查 `T` 的生命周期约束！它只要确保 `&T` 的生命周期符合规则即可，在上面代码中，它自然是符合的。

再来看一个例子:

```rust
use std::fmt::Display;

fn main() {
  let r1;
  let r2;
  {
    static STATIC_EXAMPLE: i32 = 42;
    r1 = &STATIC_EXAMPLE;
    let x = "&'static str";
    r2 = x;
    // r1 和 r2 持有的数据都是 'static 的，因此在花括号结束后，并不会被释放
  }

  println!("&'static i32: {}", r1); // -> 42
  println!("&'static str: {}", r2); // -> &'static str

  let r3: &str;

  {
    let s1 = "String".to_string();

    // s1 虽然没有 'static 生命周期，但是它依然可以满足 T: 'static 的约束
    // 充分说明这个约束是多么的弱。。
    static_bound(&s1);

    // s1 是 String 类型，没有 'static 的生命周期，因此下面代码会报错
    r3 = &s1;

    // s1 在这里被 drop
  }
  println!("{}", r3);
}

fn static_bound<T: Display + 'static>(t: &T) {
  println!("{}", t);
}
```

## static 到底针对谁？

大家有没有想过，到底是 `&'static` 这个引用还是该引用指向的数据活得跟程序一样久呢？

**答案是引用指向的数据**，**而引用本身是要遵循其作用域范围的**，我们来简单验证下：

```rust
fn main() {
    {
        let static_string = "I'm in read-only memory";
        println!("static_string: {}", static_string);

        // 当 `static_string` 超出作用域时，该引用不能再被使用，但是数据依然会存在于 binary 所占用的内存中
    }

    println!("static_string reference remains alive: {}", static_string);
}
```

以上代码不出所料会报错，原因在于虽然字符串字面量 "I'm in read-only memory" 的生命周期是 `'static`，但是持有它的引用并不是，它的作用域在内部花括号 `}` 处就结束了。

## 总结

总之， `&'static` 和 `T: 'static` 大体上相似，相比起来，后者的使用形式会更加复杂一些。

至此，相信大家对于 `'static` 和 `T: 'static` 也有了清晰的理解，那么我们应该如何使用它们呢？

作为经验之谈，可以这么来:

- 如果你需要添加 `&'static` 来让代码工作，那很可能是设计上出问题了
- 如果你希望满足和取悦编译器，那就使用 `T: 'static`，很多时候它都能解决问题

## Q&A

- 从正常的设计逻辑上看`let string = "Hello World!";` string 是局部变量，不是应该变量被释放，值也应该被释放？为何不释放，释放了应该也不会引发什么问题吧？

  - `string` 变量只是在某一个范围内持有该数据，超出这个范围后，变量被释放，但是数据依然存在，只不过不再被使用。

  - Rust中的`变量和值`，`变量和数据`，这两种说法是不同的，**值不总是等价于实际数据**，基本上可以认为，**变量的值总在栈中，但变量实际的数据，可能在栈中，可能在堆中，可能在全局内存段**。

    例如，`let s = 3u8;`，变量s的值是3，实际数据也是3，此时值等价于实际数据，3在栈中。

    再例如，`let s = "Hello World!";`，变量是s，实际数据是存放在全局内存段中的"hello world!"，变量保存的值是切片引用&str，引用是一种胖指针，指向全局内存段的实际数据。所以这里变量的值不等价于数据。

    变量退出作用域，销毁的是它的值。所以，s退出作用域，于是，被销毁的是其值，即切片引用&str，而不是销毁它的数据。

    如果实际数据保存在堆，那么变量退出作用域时，销毁值的同时是否会也销毁堆中的数据，这由实现Drop的代码来决定。

    例如，`let v = String::from("hello");`，变量v的值是一个String类型的值，String类型内部一个Vec，总的来说它的值是一个Struct，其值保存在栈中，这个Struct中有一个指针指向堆中的数据，堆中的数据其字符数据部分(即hello的每一个字符)拷贝自于全局内存段。当v被释放时，Struct被Drop，Drop的时候会将堆中的内存也释放，这是Vec实现Drop时决定的。