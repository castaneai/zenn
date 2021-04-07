---
title: "認証付きCloud RunのgRPCサービス間アクセス"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudrun", "grpc"]
published: false
---

Cloud RunがgRPCに対応し、サービス間通信がやりやすくなりました。
しかしながら、Cloud Runはデフォルトで認証が入っていてgRPCサービスとして呼び出すには認証を通す必要があります。

## Cloud Runの認証のしくみ

- GoogleからIDトークンを発行する
- 発行したIDトークンを `Authorization: Bearer ID_TOKEN` の形式で

## サービス間通信(gRPC)でCloud Runの認証を通す

```go

```