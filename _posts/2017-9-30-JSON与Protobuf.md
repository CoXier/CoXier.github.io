---
layout:     post
title:      "JSON 与 Protobuf"
author:     "CoXier"
header-img: "img/post-bg.jpg"
tags:

- 计算机基础
- 数据结构
---

今天是 9.30 日，国庆节的最后一天。这个工作周，最大的收获是接触了 protobuf ，同时也复习了 JSON。在我看来，一些东西应该是能乱记于心，这是(计算机)基础，发展之本。

# 一、JSON
`JSON`（JavaScript Object Notation）是一种**轻量级**的数据交换语言，具有一定的可读性，但是如果随着数据层级关系和内容数量增加，可读性会变得很糟糕。是作为数据交换语言，需要在 Server 和 Client 之间传递，所以在一定程度上，JSON 数据格式是独立于语言语法的。

> JSON 的官方 MIME 类型是 application/json，文件扩展名是 .json。

## 1.1 JSON 数据格式
JSON 数据格式由两个基本数据结构组成。

* 对象 Object ：一个对象从 `{` 开始，以 `}` 结束。一个对象包含一系列非排序的键值对(key - value)，键值对之间以 `,` 隔开
* 键值对 key - value : key 和 value 之间以 `:` 隔开。
  > "key" : value

  key 是一个字符串；value 可以是字符串、数值、布尔值、null、有序列表，甚至是一个 Object

一个有序列表从 `[` 开始，以 `]` 结束，包含了数据格式相同的数据。

举个例子：

```JavaScript
{
  "school" : "Happy Child",
  "student" :
  [
    {
      "student_name" : "Bob",
      "student_age" : 1
    },
    {
      "student_name" : "Alice",
      "student_age" : 2
    }
  ],
  "location" : "US",
}

```
这个 json 实际上描述了： 美国 Happy Child 幼儿园有学生 Bob 和 Alice , Bob 的年龄是 5 岁，Alice 的年龄是 6 岁。

如果我现在想在此基础上增加一个信息：该校的学生数量是 2。就可以这样写：

```JavaScript
{
  "school" : "Happy Child",
  "student" :
  [
    {
      "student_name" : "Bob",
      "student_age" : 1
    },
    {
      "student_name" : "Alice",
      "student_age" : 2
    }
  ],
  "location" : "US",
  "student_count" : 2
}
```

## 1.2 JSON 数据解析
前面说到 JSON 数据格式和语言相对独立，很多语言如 Java, Js, Python 都有 JSON 解析器，以 Java 为例来解析 JSON。在 Java 界说起 JSON 解析库，不得不提到 Google 的 [Gson](https://github.com/google/gson)。

模仿一个完整的 CS 请求，Node.js 写本地 http-server，Java 请求 server 获取 JSON 数据，然后解析 JSON 数据，转换成相应的 Java 对象。

server:

```JavaScript
const http = require('http');

const hostname = '127.0.0.1';
const port = 3000;

var jsonFile = require('./data.json')

const server = http.createServer((req, res) => {
  res.statusCode = 200;
  res.setHeader('Content-Type', 'application/json');
  res.write(JSON.stringify(jsonFile));
  res.end('\n');
});

server.listen(port, hostname, () => {
  console.log(`Server running at http://${hostname}:${port}/`);
});
```
http-server 运行在本地 3000 端口。

client :

```Java
OkHttpClient client = new OkHttpClient();
Request request = new Request.Builder()
        .url("http://127.0.0.1:3000/")
        .build();
try {
    Response response = client.newCall(request).execute();
    Gson gson = new Gson();
    School school = gson.fromJson(response.body().string(), School.class);
} catch (IOException e) {
    e.printStackTrace();
}
```
# 二、 Protobuf
Protobuf（Protocol Buffer） 是 Google 开发的用来描述结构化数据的数据协议，在网络数据传输、数据存储有重要应用。Pb 和 JSON 类似也相对于语言、平台对立，以 `.proto` 文件描述 Pb 协议。上层应用要使用 Pb 协议时，使用相应的 pbc(Protocol Buffer Compiler)进行编译，将 `.proto` 文件转换成相应的 `.cc`，`.java` 等文件。

**由于历史原因， Pb 截至目前有两个版本 2.x 和 3.x ， 3.x 支持的语言平台更多，而且语法更简单。**

## 2.1 定义 proto 文件
还是 1.1 中的例子，定义两个文件 `student.proto` 和 `school.proto`。具体的语法规则，可以参考 [Language Guide(proto3)](https://developers.google.com/protocol-buffers/docs/proto3#packages-and-name-resolution)

`demo_student.proto` :

```Java
syntax = "proto3";
option java_multiple_files = true;

message Student {
    string name = 1;
    int32 age = 2;
}
```

`demo_school.proto` :

```Java
syntax = "proto3";
option java_multiple_files = true;

import "demo_student.proto";

message School {
    string school = 1;
    string location = 2;
    int32 student_count = 3;
    repeated Student student = 4;
}
```

## 2.2 编译 proto 文件

编译 proto 文件需要用到 Protocol Buffer Compiler，下面先安装 `protoc`

安装编译工具
```bash
brew install autoconf automake libtool curl make unzip
```
如果不是 Mac ，根据系统的包管理工具安装。

生成 configure 脚本

```bash
./autogen.sh
```

最后安装即可：

```bash
$ ./configure
$ make
$ make check
$ sudo make install
$ sudo ldconfig # refresh shared library cache. (Mac 可以忽略此步骤)
```

安装完 protoc 后就可以编译了。编译命令：

> ```
> protoc --proto_path=IMPORT_PATH 
> --cpp_out=DST_DIR 
> --java_out=DST_DIR 
> --python_out=DST_DIR 
> --go_out=DST_DIR 
> --ruby_out=DST_DIR 
> --javanano_out=DST_DIR 
> --objc_out=DST_DIR 
> --csharp_out=DST_DIR 
> path/to/file.proto
> ```

虽然 protobuf 既提供了 `java` 也提供了 `javanao`，但 `javanao`生成的 java 文件很小，适用Android 平台。以 JavaNano 为例，生成 `School.java` 和 `Student.java`

## 2.3 生成运行环境

有了 2.3 中的 class 是无法直接运行的，需要相应的运行环境，这里和 2.2 中保持一致均使用 `javanao`

```bash
mvn test
mvn install
mvn package
```

生成 `protobuf-javanano-3.2.0.jar`，这就是需要的 jar 文件。

## 2.4 测试

写个 序列化的例子来演示一下如何使用 javanano。

```java
School school = new School();
school.location = "US";
school.school = "Happy Child";
school.studentCount = 2;
Student bob = new Student();
bob.age = 1;
bob.name = "Bob";

Student alice = new Student();
alice.age = 2;
alice.name = "Alice";

school.student = new Student[]{bob, alice};

byte[] result = MessageNano.toByteArray(school);

try {
    School newSchool = School.parseFrom(result);
} catch (InvalidProtocolBufferNanoException e) {
    e.printStackTrace();
}
```

本文对应的代码：https://github.com/CoXier/ComputerBasic

