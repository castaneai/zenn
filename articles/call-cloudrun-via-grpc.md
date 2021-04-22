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

- Googleサービス用のIDトークンを発行する
- 発行したIDトークンを `Authorization: Bearer ID_TOKEN` の形式でヘッダーに付けてリクエストを送る

IDトークンの発行方法はGCP内部からと[そうでない場合](https://cloud.google.com/run/docs/authenticating/service-to-service?hl=ja#calling-from-outside-gcp)でやり方が分かれますが、ここでは同じGCPプロジェクト内部での通信を仮定します。


## サービス間通信(gRPC)でCloud Runの認証を通す

Google Cloudのドキュメントにサービス間呼び出しの例があります。
しかし、この例はHTTPの呼び出しになっていてgRPCではありません。

https://cloud.google.com/run/docs/authenticating/service-to-service?hl=ja

では、gRPCではどうするのでしょうか？
実はかなりシンプルに実現ができます。
自前でIDトークンを取得して、ヘッダーにセットして…といった処理を内部でやってくれるインターフェースが用意されています。
**TokenSource** というインターフェースです。

https://pkg.go.dev/golang.org/x/oauth2#TokenSource

https://pkg.go.dev/google.golang.org/grpc/credentials/oauth#NewOauthAccess

https://github.com/grpc/grpc-go/blob/v1.37.0/credentials/oauth/oauth.go#L129

```go

```