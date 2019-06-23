---
title: 为什么我们改用 gRPC
date: 2019-06-22 16:34:38
tags: [gRPC]
categories: [翻译]
---

> 原文链接：[Why We’re Switching to gRPC](https://eng.fromatob.com/post/2019/05/why-were-switching-to-grpc/)

本文由 [Levin Fritz](https://github.com/lfritz) 于2019年5月27日发布。

当你使用微服务架构时，你需要做一个非常基本的决定：你的服务之间如何互相通信？默认的选择似乎是通过 HTTP 发送 JSON ——使用所谓的 REST APIs，尽管大多数人并不认真对待 REST 原则。在 fromAtoB 我们就是这么开始的。但最近我们决定将 gRPC 作为我们的标准。

[gRPC](https://grpc.io/) 是由谷歌开发的远程过程调用系统，现已开源。尽管它已经存在了很多年，但我很少在网上看到关于为什么人们使用或不使用它的信息，所以我决定写一篇文章来解释我们使用 gRPC 的原因。

gRPC 的一个显著优势是它使用了一种高效的二进制编码，这使它比 JSON/HTTP 更快。虽然越快越好，但对我们来说有两个方面更为重要：清晰的接口规范和对流的支持。

## gRPC 接口规范

当你创建一个新的 gRPC 服务时，第一步始终是在一个 `.proto` 文件中定义接口。下面的代码展示了它的样子——它是我们一小部分 API 的简化版本。该示例定义了一个远程过程调用 "Lookup" 和其输入输出的类型。

```protobuf
syntax = "proto3";

package fromatob;

// FromAtoB 是 fromAtoB 后端 API 的简化版本。
service FromAtoB {
	rpc Lookup(LookupRequest) returns (Coordinate) {}
}

// LookupRequest 是一个按名称查找城市坐标的请求。
message LookupRequest {
	string name = 1;
}

// Coordinate 坐标通过经纬度来确定地球上的位置。
message Coordinate {
	// Latitude 是位置的纬度，范围是 [-90, 90]。
	double latitude = 1;

	// Longitude 是位置的经度，范围是 [-180, 180]。
	double longitude = 2;
}
```

通过这个文件，你可以用 `protoc` 编译器生成客户端和服务器代码，并开始编写提供或使用 API 的代码。

所以，为什么这是件好事而不仅仅是额外工作呢？再看一眼上面的示例代码。即使你从未使用过 gRPC 或 Protocol Buffers，它也非常易读：例如，很明显，要发送 `Lookup` 请求，你应该发送一个 `name`（一个字符串），然后得到一个 `Coordinate`（由 `latitude` 和 `longitude` 组成）。事实上，你可以像本示例一样通过添加一些简单的注释，将 `.proto` 文件变成 API 服务的文档。

当然，真实服务的规格要大得多，但不会复杂的多。不过就是多为方法提供些 `rpc` 声明，为数据类型提供些 `message` 声明。

通过 `protoc` 生成的代码还确保客户端或服务器发送的数据符合规范。这为调试提供了很大的帮助。我记得有两次，我正在处理生成错误格式的 JSON 数据的服务，因为该格式没在任何地方做验证，所以问题只出现在了用户界面。找出问题的唯一方式就是调试前端 JavaScript 代码——这对从来没用过前端 JavaScript 框架的后端工程师来说绝非易事！

## Swagger / OpenAPI

原则上，通过使用 [Swagger](https://swagger.io/) 或 [OpenAPI](https://www.openapis.org/)，HTTP/JSON 也可以获得同样的优点。这有个和上面 gRPC API 等效的示例：

```yaml
openapi: 3.0.0

info:
  title: fromAtoB 后端 API 的简化版本
  version: '1.0'

paths:
  /lookup:
    get:
      description: 按名称查找城市坐标。
      parameters:
        - in: query
          name: name
          schema:
            type: string
          description: City name.
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Coordinate'
        '404':
          description: Not Found
          content:
            text/plain:
              schema:
                type: string

components:
  schemas:
    Coordinate:
      type: object
      description: Coordinate 坐标通过经纬度来确定地球上的位置。
      properties:
        latitude:
          type: number
          description: Latitude 是位置的纬度，范围是 [-90, 90]。
        longitude:
          type: number
          description: Longitude 是位置的经度，范围是 [-180, 180]。
```

将其与上面 gRPC 的规范进行比较。OpenAPI 的易读性更差！它更冗长，结构也更复杂（8个缩进级别，gRPC 只有1个）。

使用 OpenAPI 规范进行验证也比使用 gRPC 更困难。至少对内部服务来说，这意味着要么根本没写，要么跟不上 API 的发展而变得无用。

## Streaming

今年早些时候，我开始为我们的搜索设计新的 API（比如“获取2019年6月1日从柏林到巴黎的所有连接”）。我用 HTTP 和 JSON 构建了第一版 API 之后，我的一位同事指出某些情况下我们需要流式传输结果，这意味着我们应该在得到第一个结果的同时把他们发送出去。我的 API 仅仅返回一个 JSON 数组，所以在集齐所有结果之前，服务器不会发送任何数据。

我们通过在前端轮询的方式使用 API 。通过发送 POST 请求的方式设置搜索，然后发送重复的 GET 请求检索结果。响应中包含指示搜索是否完成的字段。这么做可行，但不是特别优雅，它需要服务器使用 Redis 之类的来存储中间结果。新的 API 将由多个更小的服务实现，我不想强制它们实现此逻辑。

这时我们准备尝试 gRPC 。要用 gRPC 发送远程过程调用的结果，只需在 `.proto` 文件中添加 `stream` 关键字。以下是 `Search` 函数的定义：

```protobuf
rpc Search (SearchRequest) returns (stream Trip) {}
```

`protoc` 编译器生成的代码包含一个具有 `Send` 函数的对象和一个具有 `Recv` 函数的对象，我们的服务器代码通过调用 `Send` 该函数逐一发送 `Trip` 对象，客户端代码通过调用 `Recv` 函数来检索它们。从程序员的角度来看，这比实现轮询 API 要容易的多。

## 警告

我想提一些 gRPC 的缺点。它们都和工具有关，而不是协议本身。

当使用 HTTP/JSON 构建 API 时，可以使用 curl 、httpie 或 Postman 进行简单的手动测试。gRPC 也有一个类似的工具叫 [grpcurl](https://github.com/fullstorydev/grpcurl) ，但它不是开箱即用的：你必须在服务器端添加 [gRPC server reflection](https://github.com/grpc/grpc/blob/master/doc/server-reflection.md) 扩展或为每个命令指定 `.proto` 文件。我们发现在服务器中添加一个小命令行工具可以更方便的发送简单请求。`protoc` 生成的客户端代码也使这非常简单。

对我们来说，更大的问题是我们在 HTTP 服务中使用的 Kubernetes 负载均衡器在 gRPC 中不能很好地工作。本质上，gRPC 要求在应用程序级别而不是 TCP 连接级别进行负载均衡。为了解决这个问题，我们按照这个教程：[gRPC Load Balancing on Kubernetes without Tears](https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/) 设置了 [Linkerd](https://linkerd.io/)。

## 结论

尽管构建 gRPC API 需要多做一点前期工作，但我们发现具有明确的 API 规范和对流式传输的良好支持可以弥补这一点。对我们来说，gRPC 将成为我们构建的任何新的内部服务的默认选项。
