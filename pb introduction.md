# Protocol-Buffers简介

`Protocol buffers`用于序列化结构化的数据， 是一种灵活，高效，自动化的机制，（如`XML`）但是它更小，更快，更简单。

可以定义数据如何被结构化，然后使用特定的生成的源代码轻松地将结构化数据在各种数据流中写入和读取。

可以更新数据结构，而不会破坏根据“旧”格式编译的已部署程序。

# Protocol buffers如何工作

通过在`.proto`文件中定义`protocol buffers`消息类型来指定希望如何构建序列化信息。每个`protocol buffers`消息都包含一系列**名称-值**对，`pb`消息是一个小的逻辑信息记录。

以下是`.proto`文件的一个非常基本的示例，该文件定义了包含有关人员信息的消息：

```go
message Person {
  required string name = 1;
  required int32 id = 2;
  optional string email = 3;
 
  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }
 
  message PhoneNumber {
    required string number = 1;
    optional PhoneType type = 2 [default = HOME];
  }
 
  repeated PhoneNumber phones = 4;
}
```

每种消息类型都有一个或多个唯一编号的字段，

- 每个字段都有一个**名称**和一个**值类型**，
- 其中值类型可以是：数字（整数或浮点数），布尔值，字符串，原始字节，其他`protocol buffers`消息类型

可以指定：可选字段（optional），必填字段（required），重复字段（repeated）

一旦定义了消息，就可以在`.proto`文件上运行相应编程语言的`protocol buffers`编译器来生成数据访问类，他们**为每个字段提供了简单的访问器，如`name()`和`set_name()`**，以及**将整个结构序列化或解析为原始字节的方法**。

> 例如，如果选择的语言是C++，在上面的例子上运行**编译器**将生成一个名为`Person`的类。然后，可以在应用程序中使用此类来填充，序列化和检索`Person`的`protocol buffers`消息。代码如下：

```c++
Person person;
person.set_name("John Doe");
person.set_id(1234);
person.set_email("jdoe@example.com");
fstream output("myfile", ios::out | ios::binary);
person.SerializeToOstream(&output);
```

可以在以下位置阅读消息：

```c++
fstream input("myfile", ios::in | ios::binary);
Person person;
person.ParseFromIstream(&input);
cout << "Name: " << person.name() << endl;
cout << "E-mail: " << person.email() << endl;
```

可以在不破坏向后兼容性的情况下为邮件格式添加新字段，**旧的二进制文件在解析时只是忽略新字段**。因此，如果通信协议使用`protocol buffers`作为数据格式，则可以扩展协议，而无需担心破坏现有代码。

将在[API参考](https://developers.google.com/protocol-buffers/docs/reference/overview)部分找到有关使用生成的`protocol buffers`代码的完整参考，在[`protocol buffers`编码](https://developers.google.com/protocol-buffers/docs/encoding)中找到有关`protocol buffers`消息如何编码的更多信息。

# PB vs XML

`protocol buffers`并不总是比`XML`更好的解决方案。例如:

1. `protocol buffers`不是使用标记对基于文本的文档（例如`HTML`）建模的好方法，**因为无法轻松地将结构与文本交错**。
2. `XML`是人类可读和可编辑的; `protocol buffers`在它原生的格式中不是人类可读和可编辑的。
3. `XML`在某种程度上也是自我描述的。只有拥有消息定义（如`.proto文件`）时，`protocol buffers`才有意义。

## 一点小历史

`protocol buffers`最初是在Google开发的，用于处理索引服务器请求/响应协议。**在`protocol buffers`之前**，有一种请求和响应的格式，它使用请求和响应的手动编组/解组，并支持许多版本的协议。 这导致一些非常丑陋的代码，如：

```go
if (version == 3) {
   ...
 } else if (version > 4) {
   if (version == 5) {
     ...
   }
   ...
 }
```

明确格式化的协议也使新协议版本的推出变得复杂，因为开发人员必须确保请求的发起者和处理请求的实际服务器之间的所有服务器都能理解新协议，然后才能切换以开始使用新协议。

`protocol buffers`开发用于解决这些问题：

1. 可以轻松引入新字段，而中间服务器不需要检查数据就可以简单地解析它并传递数据而无需了解所有字段。
2. 格式更具自我描述性，可以用各种语言（C++，Java等）处理。

但是，用户仍然需要自己手写解析代码。随着系统的发展，它获得了许多其他功能和用途：

1. **自动生成序列化和反序列化代码避免了手动解析的需要**。
2. 除了用于短生命周期的RPC（远程过程调用）请求之外，大家已经开始使用`protocol buffers`作为一种方便的自描述格式用于持久存储数据。
3. 首先服务的RPC接口被声明为`protocol`文件的一部分，然后使用`protocol`编译器生成`stub`类，**用户可以通过服务接口的实际实现来覆盖这些`stub`类**。