---
title: "Kubernetes上のGoアプリケーションにデバッガーを接続する"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "go", "skaffold"]
published: false
---

Go製のアプリケーション対して[Delve](https://github.com/go-delve/delve)というツールを使ってデバッグができます。
この手法を応用すると、Kubernetes上で動くGoアプリケーションに対してもデバッガーを接続できます。本記事ではその方法を紹介し、さらにKubernetes上でのデバッグを容易にするツール Skaffold debug を紹介します。

# デバッグ対象のサンプルアプリ

デバッグ対象のアプリケーションとして次のようなかんたんなHTTP Serverを仮定します。

```go
// main.go
package main

import (
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		_, _ = w.Write([]byte("hello"))
	})
	log.Printf("listening on :8080...")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

# Delveのおさらい

はじめにDelveのおさらいです。DelveはGoアプリケーション用のデバッガーであり、CLIツールを使って直接デバッグをすることもできます。

```shell
# Delveのインストール
go install github.com/go-delve/delve/cmd/dlv@latest

# ソースコードを指定してデバッグ開始
dlv debug main.go

# or ビルド済みのバイナリを指定してデバッグ開始
 dlv exec ./main
```

DelveのCLIを直接使うとGDBのように対話式でブレークポイントを置いたりステップ実行をしたりできます。

```sh
$ dlv debug main.go
Type 'help' for list of commands.
(dlv) break main.go:10
Breakpoint 1 set at 0x1025de674 for main.main.func1() ./main.go:10
(dlv) c
2022/12/06 12:51:57 listening on :8080...
> main.main.func1() ./main.go:10 (hits goroutine(4):1 total:1) (PC: 0x1025de674)
     5:		"net/http"
     6:	)
     7:	
     8:	func main() {
     9:		http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
=>  10:			_, _ = w.Write([]byte("hello"))
    11:		})
    12:		log.Printf("listening on :8080...")
    13:		log.Fatal(http.ListenAndServe(":8080", nil))
    14:	}
```

## リモートデバッグ

DelveにはCLIによる対話式デバッグの他に、外部からデバッガーを接続する **リモートデバッグ** の機能があります。
Delve CLIのコマンドに `--listen` 等のいくつかのパラメータを加えるとリモートデバッグが可能です。

```shell
# TCPポート 2345 番で外部デバッガーからの接続を受け入れる
dlv --listen=127.0.0.1:2345 --continue --headless=true --accept-multiclient --api-version=2 exec ./main
```

Delveに対して接続できるデバッガーはいくつかあり、例えば [Go for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=golang.Go) などです。Go for Visual Studio Codeの場合、次のような設定ファイル (`launch.json`)となります。

```json
{
  "configurations": [{
    "name": "Connect to server",
    "type": "go",
    "request": "attach",
    "mode": "remote",
    "remotePath": "${workspaceFolder}",
    "port": 2345,
    "host": "127.0.0.1"
  }]
}
```

## コンテナ上で実行されるアプリケーションのデバッグ

Kubernetes上に乗せるには、Goアプリケーションのコンテナを作る必要があります。Dockerfileを使うとGoのアプリケーションは次のような構成になることが多いかと思います。

```Dockerfile
FROM golang:alpine as builder
WORKDIR /go/src/app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /bin/main main.go

FROM scratch
COPY --from=builder /bin/main /main
ENTRYPOINT [ "/main" ]
```

このコンテナにDelveを入れることで、コンテナ内であってもデバッグが可能です。Dockerfileを次のように編集します。

```Dockerfile
FROM golang:alpine as builder
WORKDIR /go/src/app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /bin/main main.go
# Delve のインストール
RUN CGO_ENABLED=0 go install -ldflags "-s -w -extldflags '-static'" github.com/go-delve/delve/cmd/dlv@latest

FROM scratch
COPY --from=builder /bin/main /main
# Delveのバイナリを持ってくる
COPY --from=builder /go/bin/dlv /dlv
# Delveのリモートデバッグ機能で起動
ENTRYPOINT [ "/dlv", "--listen=:2345", "--continue", "--headless=true", "--accept-multiclient", "--api-version=2", "exec", "/main" ]
```

このようにDelveのリモートデバッグ機能はコンテナ内であっても通用します。この状態でTCP/2345をポートフォワーディングすると、コンテナの外部からデバッガーを接続できます。

```shell
docker build -t main-debug -f Dockerfile .
docker run --rm -p 2345:2345 -p 8080:8080 main-debug

# デバッガー接続後、main:10 にブレークポイントを設置
# curlでアクセスしてみる
curl http://localhost:8080
# ブレークポイントで止まるはず
```

ここでひとつ注意点として、ソースコードのパスをビルドしたコンテナのパスに合わせる必要があります。上記のDockerfileでは `WORKDIR /go/src/app` と指定したため、バイナリに埋め込まれたデバッグ情報上のソースコードのパスも `/go/src/app` が基準となります。
よってブレークポイントを認識させるために、デバッガーの設定でパスを指定する必要があります。

```diff
{
  "configurations": [{
    "name": "Connect to server",
    "type": "go",
    "request": "attach",
    "mode": "remote",
-    "remotePath": "${workspaceFolder}",
+    "remotePath": "/go/src/app",
    "port": 2345,
    "host": "127.0.0.1"
  }]
}
```

## Kubernetes上で実行されるアプリケーションのデバッグ

コンテナ上のアプリに対してリモートデバッグする方法がわかったので、次はいよいよこれをKubernetesに持っていきます。
コンテナイメージには上記のDockerfileから作ったものをそのまま使います。

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: main
spec:
  containers:
  - name: main
    image: main-debug
    ports:
    - containerPort: 8080
    - containerPort: 2345
```

このPodに対して `kubectl port-forward` でポートフォワーディングをすると手元の環境からKubernetes上のアプリケーションにデバッガーを接続できます。

```shell
kubectl port-forward pods/main 2345:2345 &
kubectl port-forward pods/main 8080:8080 &

# デバッガー接続後、main:10 にブレークポイントを設置
# curlでアクセスしてみる
curl http://localhost:8080
# ブレークポイントで止まるはず
```

# Skaffold debug の活用

ここまででKubernetes上で動くGoアプリケーションでもリモートデバッグできました。
しかし、Delveを仕込んだ `Dockerfile` を準備する手間がかかる上に、コードの変更をする度にコンテナのビルド（+push）、Kubernetesへのマニフェストの反映 (`kubectl apply`)を繰り返すのは面倒です。

そこで、[Skaffold](https://skaffold.dev/) というツールが助けになります。Skaffold はコードの変更に応じて自動的にイメージのビルドとKubernetesへの反映を行うツールです。skaffoldを利用することで手動でのコンテナイメージのビルドと、`kubectl apply` による反映が不要になります。便利ですね！

さらに、Skaffoldには `skaffold debug` というコマンドがあります。このコマンドをGoアプリケーションのコンテナに対して使うと、 **コンテナイメージに自動的にDelveを差し込み、port-forwardもやってくれます。** したがって、Delveを仕込んだ `Dockerfile` の準備は不要となります。

Skaffold では `skaffold.yaml` というファイルに設定を書きます。設定の詳細は[公式ドキュメント](https://skaffold.dev/docs/references/yaml/) が詳しいです。

```yaml
# skaffold.yaml
apiVersion: skaffold/v4beta1
kind: Config
metadata:
  name: debug-example
build:
  local:
    # 筆者の環境ではRancher Desktop上のKubernetesを利用したため、イメージのpushはしない
    # pushが必要な環境の場合は true に変更する
    push: false 
  artifacts:
  - image: main
    docker:
      dockerfile: Dockerfile
manifests:
  rawYaml:
  - pod.yaml
```

デバッガーDelveの仕込みはすべてSkaffold側がやってくれるので、 `Dockerfile` は次のようにシンプルなもので大丈夫です。

```Dockerfile
FROM golang:alpine as builder
WORKDIR /go/src/app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /bin/main main.go

FROM scratch
COPY --from=builder /bin/main /main
ENTRYPOINT [ "/main" ]
```

KubernetesのPodの定義もスタンダードな形で大丈夫ですが、一点だけ環境変数 `GOTRACEBACK=1` が増えています。これはSkaffold Debugに「このコンテナはGoアプリケーションである」と認識させるために設定しています。詳しくは[Skaffold debugのドキュメント](https://skaffold.dev/docs/workflows/debug/#go-runtime-go) を参照ください。

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: main
spec:
  containers:
  - name: main
    image: main
    imagePullPolicy: Never
    env:
    - name: GOTRACEBACK
      value: "1"
    ports:
    - containerPort: 8080
```

以上の3ファイル（`skaffold.yaml`, `Dockerfile`, `pod.yaml`) が揃った状態で `skaffold debug` を実行するとデバッガーが接続可能なコンテナが自動的に起動します。コードの変更に応じて再ビルドもされます。

```shell
➜ skaffold debug
Listing files to watch...
 - main
Generating tags...
 - main -> main:latest
Some taggers failed. Rerun with -vdebug for errors.
Checking cache...
 - main: Not found. Building
Starting build...
Building [main]...
Target platforms: [linux/arm64]

...
Build [main] succeeded
Tags used in deployment:
 - main -> main:aa8e1e92a82f6e74b27b4fbfbc1cea28a60c56179649f06ad5082e66066f1d2b
Starting deploy...
 - pod/main created
Waiting for deployments to stabilize...
 - pods: pods not ready: [main]
 - pods is ready.
Deployments stabilized in 5.141 seconds
Press Ctrl+C to exit
Not watching for changes...
[install-go-debug-support] Installing runtime debugging support files in /dbg
[install-go-debug-support] Installation complete
[main] API server listening at: [::]:56268
[main] 2022-12-06T09:01:47Z warning layer=rpc Listening for remote connections (connections are not authenticated nor encrypted)
[main] 2022/12/06 09:01:47 listening on :8080...
Port forwarding pod/main in namespace default, remote port 56268 -> http://127.0.0.1:56268
```

ログにある通りTCPポート 56268でデバッガーが待ち受けていることがわかります。
よってTCP/56268 に向けてデバッガーを接続すればリモートデバッグが可能になります。

これでデバッグ用の特別なイメージやManifestを用意せずともGoアプリケーションのデバッグを行うことができました。Skaffold 素晴らしい！

# （おまけ）Cloud Code

上記のSkaffoldをさらに応用したツールとして[Cloud Code](https://cloud.google.com/code) というものがあります。
これはGCP (Google Cloud)に特化したエディタ（IDE）の拡張です。現在はJetBrainsとVisual Studio Code用のものが用意されています。

エディタのUI上からGoogle Cloudのリソース（主にGKEやCloud Run）の管理ができます。
また、 **内部的にSkaffoldを使っているため** コードの変更に応じて自動的に再デプロイしてくれます。

![](https://storage.googleapis.com/zenn-user-upload/c895f05ed844-20221221.png)
画像: [Google Cloudのブログ記事](https://cloud.google.com/blog/products/devops-sre/announcing-cloud-code-accelerating-cloud-native-application-development) より引用

また、デバッグ機能も搭載されておりやはり内部的にはSkaffold debugが使われているようです。よって、GKE上で開発する際はCloud Codeを使うとより管理が楽になるかもしれません。

しかし、内部的に何が起こっているのか把握しないままクラウドのリソースを操作するのは少し怖さも感じます。筆者の個人的な感想ですがCloud Codeを使うなら一度はSkaffold単体を使って何が起こっているか把握してからにしたほうがいいのかなと思います。
