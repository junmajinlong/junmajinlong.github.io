## Protobuf协议简介

Protobuf是类似于json、xml的用于序列化和反序列化的一种更高效的协议。

json和xml是一种编码解码协议，它们用于数据的序列化和反序列化。它们序列化的结果是用户友好的文本格式，人类可直接阅读这类文本，因此也经常用于数据存储、数据展示和文件配置。

Protobuf是一种更为高效的用于序列化和反序列化的协议，它序列化的结果是二进制格式而不是文本(Protobuf文件是用户友好的文本代码，通过编译器文本代码编译成二进制)，因此对人类不友好，但对计算机更友好，它序列化的二进制体积比json和xml要小的多，因此在计算机之间传输的效率要高的多。

由于Protobuf序列化的结果是二进制，用户无法直接读取序列化的结果，因此Protobuf只用于序列化和反序列化，而不用于数据展示。当然，也有一些人会用Protobuf来定义配置文件。

Protobuf常配合gRPC使用，客户端将按照Protobuf协议定义的数据进行序列化后发送给对服务端，服务端按照Protobuf协议进行反序列化。

## proto3编写和编译示例

> 本文不介绍protobuf协议的详细内容，只是一个简单的proto文件的示例，protobuf协议的具体语法，参考官方手册<https://developers.google.com/protocol-buffers/docs/proto3>。

Protobuf有两个协议版本，proto2和proto3，现在主要使用proto3协议。

Protobuf文件以`.proto`为后缀。

如果使用vscode，建议安装上`vscode-proto3`插件，同时安装好Protobuf编译器protoc以及格式化程序clang-format工具：

```bash
# ubuntu
sudo apt install protobuf-compile libprotobuf-dev clang-format

# windows，据测试，在win下装了clang-format依然无法格式化
# 1.下载win版本的protobuf并解压将bin目录加入到PATH环境变量：
#   https://github.com/protocolbuffers/protobuf/releases
# 2.下载完整的llvm，将bin目录加入PATH环境变量，可使用clang-format.exe:
#   https://github.com/llvm/llvm-project/releases
#   https://github.com/llvm/llvm-project/releases/download/llvmorg-15.0.0/LLVM-15.0.0-win64.exe
```

然后可编写proto文件。下面是一个`.proto`文件的内容示例：

```protobuf
// 双斜线和/**/是注释语法


// 指定协议版本，必须写在非注释行的第一行
syntax = "proto3";

// 指定该文件的包名
package voting;

// 定义服务和方法，编译为客户端和服务端代码后，
// 这里面定义的方法名将同时存在于客户端和服务端
service Voting {
  rpc Vote(VotingRequest) returns (VotingResponse);
}

// message定义类型，编译后一般转换为对应语言的Struct或某种Class类型
message VotingRequest {
  string url = 1;
  enum Vote {
    UP = 0;
    DOWN = 1;
  }
  
  Vote vote = 2;
}

message VotingResponse {
  string confirmation = 1;
}
```

写好`.proto`文件后，接下来是将其进行编译，编译后得到的结果是二进制。但一般不直接编译为二进制，而是将proto中定义的数据结构和服务都转换为各种语言对应的代码，方便在后续的代码中直接使用这些数据结构。

Protobuf官方已经直接支持一些语言，如`PHP Python Ruby Java C++`等，可以直接使用官方的`protoc`将proto文件转换为这些语言对应的代码。

```bash
# 将voting.proto编译并转换为Python对应的代码，生成的代码文件在当前目录下
$ protoc.exe --python_out=. voting.proto
```

对于Protobuf官方目前还不支持的语言，比如Rust，需要通过该语言的第三方库进行转换。当然，官方支持的语言也依然有不少编译转换的第三方库，参考<https://github.com/protocolbuffers/protobuf/blob/main/docs/third_party.md>。
