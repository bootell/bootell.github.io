---
title: 'PHP的gRPC客户端与服务端实践'
date: 2020-08-13 11:14:00
updated: 2020-08-13 11:14:00
tags: PHP
---


PHP 服务之间互相调用，有很多选择，可以通过 HTTP 接口，或者 RPC 像 [yar](https://github.com/laruence/yar)。考虑到今后多语言的扩展性与通用性，选取了 gRPC 作为方式。


<!--more-->


### gRPC通信

gRPC 是一种可在任何环境中运行的现代开源高性能 RPC 框架，可以像调用本地对象一样直接调用另一台不同的机器上服务端应用的方法。

对比之前通过 HTTP 使用 Json 格式发送数据，说有以下优点：

- 使用 `*.proto` 文件生成相关代码，格式更规范，过程调用简化操作，不会受到 HTTP 资源方法语义的限制；
- 使用 Protobuf 编译成二进制，在编组速度和代码大小方面提供了更高效的数据交换；
- 通过 HTTP/2 进行高效网络传输，引入双向流式传输、流控制、报头压缩等，减少资源使用量，从而缩短应用与服务之间的响应时间。

gRPC 允许四种方法：

- 单向，即客户端发送一个请求给服务端，从服务端获取一个应答
- 服务端流式，即客户端发送一个请求给服务端，可获取一个数据流用来读取一系列消息
- 客户端流式，即客户端用提供的一个数据流写入并发送一系列消息给服务端
- 双向流式，即两边都可以分别通过一个读写数据流来发送一系列消息

若使用fpm模式运行PHP，暂时只能方便实现单向 RPC。



### gRPC环境配置

PHP 安装 gRPC 支持，参考文档：[Quick Start - grpc.io](https://grpc.io/docs/languages/php/quickstart/)

```shell
# 安装扩展，有 pecl 和 composer，两种方式，pecl 性能更好
# gRPC扩展
pecl install grpc
composer require grpc/grpc
# Protobuf扩展
pecl install protobuf
composer require google/protobuf

# protoc编译器下载
curl --fail -L -O https://github.com/protocolbuffers/protobuf/releases/download/v3.12.4/protoc-3.12.4-linux-x86_64.zip
unzip protoc-3.12.4-linux-x86_64.zip 'bin/protoc' -d /usr/local
# 编译器生成php组件
apt-get install automake libtool
git clone -b v1.31.0 https://github.com/grpc/grpc
cd grpc && git submodule update --init
make grpc_php_plugin
mv bins/opt/grpc_php_plugin /usr/local/bin
```



### gRPC 使用


编写 `Hello.proto` 文件

```protobuf
syntax = "proto3";
  
package test;
  
// The greeting service definition.
service Hello {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloResponse) {}
}
  
// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}
  
// The response message containing the greetings
message HelloResponse {
  string message = 1;
}
```

生成 php 文件

```shell
protoc -I ./protos_dir --php_out=./php_dir --grpc_out=./GPBMetadata_dir --plugin=protoc-gen-grpc=$(which grpc_php_plugin) $file
```

由于生成的文件命名空间都是根目录，所以需要在 `composer.json` 中添加自动加载，然后执行 `composer dump-autoload`

```json
"autoload": {
  "psr-4": {
    "GPBMetadata\\": "GPBMetadata_dir/GPBMetadata",
    "Test\\": "php_dir/Test"
  }
}
```

Nginx 对 gRPC的支持，参考文档：[gRPC Support with NGINX](https://www.nginx.com/blog/nginx-1-13-10-grpc/)

```nginx
server {
    listen 8000 http2;
    server_name grpc.localhost;
    index index.php;、
    root /var/www/grpc;

    add_trailer grpc-status $sent_http_grpc_status;
    add_trailer grpc-message $sent_http_grpc_message;

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass php-fpm:9000;
        try_files $uri =404;
    }
}
```

使用 yii2 框架实现的完整例子：[bootell/yii2-grpc-demo](https://github.com/bootell/yii2-grpc-demo)
