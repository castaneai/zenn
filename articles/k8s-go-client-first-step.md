---
title: "Kubernetes APIをGoから使う最初の一歩"
emoji: "☸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "go"]
published: true
---

GoでKubernetes APIを使用する場合は[client-go](https://github.com/kubernetes/client-go)というライブラリを使います。

よって、通常通り次のように `go get` するわけですが、いくつか注意点があります。

```
go get k8s.io/client-go
```

## バージョン番号のルール

はじめに注意すべき点はclient-goのバージョン番号は**Kubernetesのバージョン番号と一致していない**ことです[^1]。
たとえば、Kubernetes v1.x.y に対応するclient-goのバージョン番号は **v0.x.y** になります。先頭のメジャーバージョンだけが0と1で異なります。

よってKubernetes v1.18.15 のクラスターが対象であれば次のようなバージョン指定になります。

```
go get k8s.io/client-go@v0.18.15
```

特殊なバージョン形式にはなりますが、Kubernetesバージョンと一致させたタグも一応あるみたいです。次のように `@kuberentes-x.y.z` というタグを指定すると利用できます。

```
go get k8s.io/client-go@kubernetes-1.18.15
```

[^1]: https://github.com/kubernetes/client-go#versioning

## api と apimachinery

client-goを使ってさっそくAPIを使ってみると、client-goだけでは補完できない型が現れます。
たとえばnamespaceの一覧を取得する次のAPIでは引数に `metav1.ListOptions`, 返り値に `v1.NamespaceList` という型が出てきます。

```go
// NamespaceInterface has methods to work with Namespace resources.
type NamespaceInterface interface {
	...
	List(ctx context.Context, opts metav1.ListOptions) (*v1.NamespaceList, error)
}
```

これらの型を使うためには、追加でライブラリの取得が必要になるのですが、実は2つの型はそれぞれ異なるパッケージに所属しています。

| 型 | 所属パッケージ | repository |
|:---|:---|:--|
|`metav1.ListOptions`|`k8s.io/apimachinery/pkg/apis/meta/v1`| https://github.com/kubernetes/apimachinery |
|`v1.NamespaceList`|`k8s.io/api/core/v1`| https://github.com/kubernetes/api |

どちらもKubernetes内部の型定義ではありそうですが、何が違うのでしょうか？
筆者も最初は全然わからなかったのですが、Twitterで @go_vargo さんから教えていただきました。

https://twitter.com/go_vargo/status/1399314172601462790

`api` はKubernetes APIに存在するものをそのまま表していて、一方で `apimachinery` はAPIリクエストのための構造体など補助的なものを表すとみてよさそうです。

というわけで、必要に応じて `k8s.io/api` と `k8s.io/apimachinery` パッケージも追加してあげるとKubernetes APIを使ったコードがビルド可能になります。バージョン番号のルールはclient-goと同じです。

```
go get k8s.io/api@v0.18.15
go get k8s.io/apimachinery@v0.18.15
```

それでは楽しいKubernetesライフを！
