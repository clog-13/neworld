# 07_快速上手Lua

Lua 是一个小巧精妙的脚本语言，诞生于巴西的大学实验室，这个名字在葡萄牙语里的含义是“美丽的月亮”。从作者所在的国家来看，NGINX 诞生于俄罗斯，Lua 诞生于巴西，OpenResty 诞生于中国，这三门同样精巧的开源技术都出自金砖国家，而不是欧美，也是挺有趣的一件事。

Lua 在设计之初，就把自己定位为一个简单、轻量、可嵌入的胶水语言，没有走大而全的路线。 Lua 的使用其实非常广泛。很多的网游，比如魔兽世界，都会采用 Lua 来编写插件；而键值数据库 Redis 则是内置了 Lua 来控制逻辑。

虽然 Lua 自身的库比较简单，但它可以方便地调用 C 库，大量成熟的 C 代码都可以为其所用。比如在 OpenResty 中，很多时候都需要调用 NGINX 和 OpenSSL 的 C 函数，而这都得益于 Lua 和 LuaJIT 这种方便调用 C 库的能力。

## 环境和 hello world

不用专门去安装标准 Lua 环境，因为 OpenResty 已经不再支持标准 Lua，而只支持 LuaJIT。这里介绍的 Lua 语法，也是和 LuaJIT 兼容的部分，而不是基于最新的 Lua。

可以新建一个 1.lua 文件，并用 luajit 来运行其中的 hello world 代码：

```bash
$ cat 1.lua
 print("hello world")

$ luajit 1.lua
 hello world
```

当然，还可以**使用 resty 来直接运行**，它最终也是用 LuaJIT 来执行的：

```bash
$ resty -e 'print("hello world")'
 hello world
```

## 数据类型

Lua 中的数据类型不多，可以通过 type 函数来返回一个值的类型，比如下面这样的操作：

```bash
$ resty -e 'print(type("hello world")) 
 print(type(print)) 
 print(type(true)) 
 print(type(360.0))
 print(type({}))
 print(type(nil))
 '
```

会打印出如下内容：
```
 string
 function
 boolean
 number
 table
 nil
```
### 字符串

在 Lua 中，字符串是不可变的值，如果要修改某个字符串，就等于创建了一个新的字符串。好处是即使同一个字符串出现了很多次，在内存中也只有一份；劣势是，如果想修改、拼接字符串，会额外地创建很多不必要的字符串。

比如下面这段代码，是把 1 到 10 这些数字当作字符串拼接起来。对了，在 Lua 中，我们使用两个点号来表示字符串的相加：

```bash
$ resty -e 'local s  = ""
 for i = 1, 10 do
     s = s .. tostring(i)
 end
 print(s)'
```

这里我们循环了 10 次，但只有最后一次是我们想要的，而中间新建的 9 个字符串都是无用的。它们不仅占用了额外的空间，也消耗了不必要的 CPU 运算。

当然，在后面的性能优化章节，我们会有对应的方法来解决它。

另外，在 Lua 中，有三种方式可以表达一个字符串：单引号、双引号，以及长括号（[[]]）。前面两种都比较好理解，别的语言一般也这么用，那么长括号有什么用处呢？

我们看一个具体的示例：

```bash
$ resty -e 'print([[string has \n and \r]])'
 string has \n and \r
```

可以看到，长括号中的字符串不会做任何的转义处理。

也许会问另外一个问题：如果上面那段字符串中包括了长括号本身，又该怎么处理呢？答案很简单，就是**在长括号中间增加一个或者多个 = 符号**：

```bash
$ resty -e 'print([=[ string has a [[]]. ]=])'
  string has a [[]].
```

### 布尔值

在 Lua 中，**只有 nil 和 false 为假**，其他都为真，**包括 0 和空字符串也为真**：

```bash
$ resty -e 'local a = 0
 if a then
   print("true")
 end
 a = ""
 if a then
   print("true")
 end'
```

这种判断方式和很多常见的开发语言并不一致，所以，为了避免在这种问题上出错，可以显式地写明比较的对象，比如下面这样：

```bash
$ resty -e 'local a = 0
 if a == false then
   print("true")
 end
 '
```

### 数字

Lua 的 number 类型，是用双精度浮点数来实现的。LuaJIT 支持 dual-number（双数）模式，也就是说， LuaJIT 会根据上下文来用整型来存储整数，而用双精度浮点数来存放浮点数。

LuaJIT 还支持长长整型的大整数。

### 函数

函数在 Lua 中是一等公民，可以把函数存放在一个变量中，也可以当作另外一个函数的入参和出参。

比如，下面两个函数的声明是完全等价的：

```bash
function foo()
 end
```

和

```bash
foo = function ()
 end
```

### table

table 是 Lua 中唯一的数据结构：

```bash
$ resty -e 'local color = {first = "red"}
print(color["first"])'
 red
```

### 空值

在 Lua 中，空值就是 nil。**如果定义了一个变量，但没有赋值**，它的默认值就是 nil：

```bash
$ resty -e 'local a
 print(type(a))'
 nil
```

当真正进入 OpenResty 体系中后，会发现很多种空值，比如 ngx.null 等等。

## 常用标准库

很多时候，我们学习一门语言，其实就是在学习它的标准库。

Lua 比较小巧，内置的标准库并不多。而且，在 OpenResty 的环境中，Lua 标准库的优先级是很低的。对于同一个功能，**推荐优先使用 OpenResty 的 API 来解决，然后是 LuaJIT 的库函数，最后才是标准 Lua 的函数**，**OpenResty的API > LuaJIT的库函数 > 标准Lua的函数**。

### string 库

字符串操作是我们最常用到的，也是坑最多的地方。有一个简单的原则，那就是如果涉及到正则表达式的，请一定要使用 OpenResty 提供的 ngx.re.* 来解决，不要用 Lua 的 string.* 处理。因为，Lua 的正则不符合 PCRE 的规范。

其中 string.byte(s [, i [, j ]])，是比较常用到的一个 string 库函数，它返回字符 s[i]、s[i + 1]、s[i + 2]、······、s[j] 所对应的 ASCII 码。i 的默认值为 1，即第一个字节，j 的默认值为 i。

下面我们来看一段示例代码：

```bash
$ resty -e 'print(string.byte("abc", 1, 3))
 print(string.byte("abc", 3)) -- 缺少第三个参数，第三个参数默认与第二个相同，此时为 3
 print(string.byte("abc"))    -- 缺少第二个和第三个参数，此时这两个参数都默认为 1
 '
```

它的输出为：
```
 979899
 99
 97
```

### table 库

在 OpenResty 的上下文中，对于 Lua 自带的 table 库，除了 table.concat 、table.sort 等少数几个函数，大部分不推荐使用。

table.concat一般用在字符串拼接的场景下，比如下面这个例子。它可以避免生成很多无用的字符串。

```bash
$ resty -e 'local a = {"A", "b", "C"}
 print(table.concat(a))'
```

### math 库

Lua math 库由一组标准的数学函数构成。在 OpenResty 的实际项目中，很少用 Lua 去做数学方面的运算，不过其中和随机数相关的 math.random() 和 math.randomseed() 两个函数比较常用，比如下面的这段代码，它可以在指定的范围内，随机地生成两个数字。

```bash
$ resty -e 'math.randomseed (os.time()) 
 print(math.random())
 print(math.random(100))'
```

## 虚变量

设想这么一个场景，当一个函数返回多个值的时候，有些返回值我们并不需要，Lua 提供了一个虚变量（dummy variable）的概念， 按照惯例以一个下划线来命名，用来表示丢弃不需要的数值，仅仅起到占位的作用。

下面我们以 string.find 这个标准库函数为例，来看虚变量的用法。这个标准库函数会返回两个值，分别代表开始和结束的下标。

如果我们只需要获取开始的下标，那么很简单，只声明一个变量来接收 string.find 的返回值即可：

```bash
$ resty -e 'local start = string.find("hello", "he")
 print(start)'
 1
```

但如果只想获取结束的下标，那就必须使用虚变量了：

```
$ resty -e 'local  _, end_pos = string.find("hello", "he")
 print(end_pos)'
 2
```

除了在返回值里使用，虚变量还经常用于循环中，比如下面这个例子：

```bash
$ resty -e 'for _, v in ipairs({4,5,6}) do
     print(v)
 end'
 4
 5
 6
```

而当有多个返回值需要忽略时，可以重复使用同一个虚变量。

## Q&A

**Q：用当前时间戳作为种子的，这种方法是否有问题呢？又该如何生成好的种子呢？**

```
math.randomseed(tostring(os.time()):reverse():sub(1, 6)) 
```

**即将时间值转换为字符串，然后将字符串倒序，然后取其前n位作为种子**。因为当时间变化很小的时候，产生随机数的序列很相似。所以通过这种方法使得即使时间变化很小，由于reverse操作，时间的高位变成低位，低位变成高位，随机数种子的值变化会很大。

但os.time返回当前时间的秒数，不能阻止在同一秒内产生相同的随机数序列。

另一种方法是说**对计算机的一些操作，如键盘、鼠标操作，会产生一些随机数，这些随机数叫熵**。用户可以通过读取/dev/random和/dev/urandom文件来获取这些随机数。只不过**读取/dev/random时，如果文件里的熵不足时会阻塞**。**读取/dev/urandom时，不会阻塞**，但不能保证是合适的数据（熵不足时怎么处理未测试）。

通过cat /proc/sys/kernel/random/entropy_avail操作可以查看有多少熵可以用。 另外可以利用芯片电磁噪声来生成随机数。

如果有加密的需求，从 /dev/random 和 /dev/urandom 读取会更安全。



**Q：如果在init_by_lua 中引入某个模块，不加local，作为全局变量，是不是说这个模块就可以在以后rewrite,access 等阶段直接拿来使用？这样做相比较于在各个阶段自己引入模块，是否减少了require的次数，提高了性能？**

不会提高性能，模块在单个 worker 中只会加载一次，和是否加了 local 无关。设置为全局变量，很容易出错，比如重名什么的。在OpenResty 中建议所有变量都 local。
