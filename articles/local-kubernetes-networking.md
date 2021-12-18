---
title: "ローカルKubernetes環境のネットワークを攻略する"
emoji: "☸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes"]
published: false
---

この記事は[Kubernetes Advent Calendar その2](https://qiita.com/advent-calendar/2021/kubernetes) 21日目の記事です。

---

Kubernetes学習のために、まずはローカル環境を構築して触ってみようと考える方は多いでしょう。しかし、ローカル環境でもそれ固有の難しさがあります。
本記事ではその難しさ（特にネットワークまわり）を整理し、ローカルKubernetes環境を攻略します。

## ローカルKubernetesを構築するツール

Kubernetesは本来**単一のPCで動かすツールではありません。** よってローカルで動作させるには何かしらローカル環境構築を補助してくれるツールを使います。代表的なものとして **minikube** や **kind** などがあります。

どれを使えば良いのか迷った場合はひとまず[minikube](https://minikube.sigs.k8s.io/)を使うとよいでしょう。
minikube公式ドキュメントの[Get Started](https://minikube.sigs.k8s.io/docs/start/)を読むとインストール、初期化、アプリのデプロイ、削除までを一通り体験できます。

たしかに上から順番にコマンドを実行していくと `hello-minikube` アプリを確認できますが、**アプリ1つ公開するのになぜこんなにコマンドが多いのか？** と疑問に感じます。
特に、Kubernetesをはじめて触る方はお手軽にローカル環境でやるつもりが初手から混乱してしまいそうです。

そこで、アプリへのアクセス経路という観点で順序立てて仕組みを見ていきましょう。

## Podへのアクセス経路

Kubernetes上でアプリが動く基本単位は**Pod**です。Kubernetes内には必ずPodが存在していてWebブラウザからアクセスできるということは、WebブラウザからPodへの経路があるはずです。

![](https://storage.googleapis.com/zenn-user-upload/debc834c9048-20211218.png)

しかし、実際にはKubernetesの外側からPodに直接アクセスできません。つまり、経路を作ってくれている何かが間にいる、ということです。

![](https://storage.googleapis.com/zenn-user-upload/d52e98febacf-20211218.png)

その答え（のひとつ）が**Service**です。

## Service

ServiceはPodへのアクセス経路を作ってくれるものです。公式のチュートリアルにも以下の記載があります。

> 各Podには固有のIPアドレスがありますが、それらのIPは、Serviceなしではクラスターの外部に公開されません。Serviceによって、アプリケーションはトラフィックを受信できるようになります。

https://kubernetes.io/ja/docs/tutorials/kubernetes-basics/expose/_print/#pg-8ef4dad8f743b191a9e8c6f891cb191a

よって、さっきの図に当てはめるとここにServiceが入ります。

![](https://storage.googleapis.com/zenn-user-upload/83f2ebb5e3ae-20211218.png)

minikubeのGet Startedで登場した以下のコマンドはServiceを作る処理というわけです。

```
kubectl expose deployment hello-minikube --type=NodePort --port=8080
```

しかし、[Get Started](https://minikube.sigs.k8s.io/docs/start/)の続きを読むと分かる通り、Serviceを作って終わりではありません。
Webブラウザからアクセスするためにはまだ必要な要素が残っているようです。

## Serviceだけでは足りない！？

確かに公式ドキュメントには「Serviceによって、アプリケーションはトラフィックを受信できるようになります」と書いてあるのに、Serviceでは足りないのはなぜでしょうか？

それは、ローカル環境だからです。公式ドキュメントはあくまで**Kubernetes本体のドキュメントであり、その下にある環境固有の条件については言及していない**のです。

実際はKubernetesクラスタはminikube環境に閉じ込められており、次のような構造になっています。

![](https://storage.googleapis.com/zenn-user-upload/c72988c24d9d-20211218.png)

よって、minikubeの外側からServiceへの経路が別途必要になるわけです。

![](https://storage.googleapis.com/zenn-user-upload/b54a013613bc-20211218.png)

それが次のコマンドです。

```
minikube service hello-minikube
```

実行すると次のような出力が出て、書いてあるURLにアクセスするとアプリが表示されます。

```
|-----------|----------------|-------------|---------------------------|
| NAMESPACE   | NAME             | TARGET PORT   | URL                         |
| ----------- | ---------------- | ------------- | --------------------------- |
| default     | hello-minikube   | 8080          | http://192.168.64.2:32423   |
| ----------- | ---------------- | ------------- | --------------------------- |
🎉  Opening service default/hello-minikube in default browser...
```

また、この方法以外にも次のコマンドも紹介されています。

```
kubectl port-forward service/hello-minikube 7080:8080
```

よって、minikubeの外側と内側の経路を作るのは `minikube service` コマンドまたは `minikube port-forward` コマンドとなります。

![](https://storage.googleapis.com/zenn-user-upload/d6c4c6dce402-20211219.png)

つまり、minikube内のPodへ外部からアクセスするには2段階の経路が必要になります。
なぜ2段階に分かれているのでしょうか、Serviceに直接アクセスではダメなのでしょうか？その理由を探ります。

## minikubeと仮想化技術

そもそもKubernetesは基本的にLinuxマシンで動かす想定のソフトウェアです。よって、WindowsやMacでは何らかの仮想化が必要です。

仮想的なLinuxマシンを動かすとなると、DockerやVirtualBox, WSL2, limaなどが選択肢としてあがります。minikubeはどの仮想化技術を使うか選択できる仕組みを持っていて、それぞれの技術はDriverと呼ばれます。次のドキュメントにDriverの一覧が書かれています。

[Drivers | minikube](https://minikube.sigs.k8s.io/docs/drivers/)

ページを見ると各OSごとに多くのDriverが用意されていることがわかります。
つまり、仮想化技術は色々あってそれぞれにおいて**外から内部への経路を通す方法が異なる** のです。

たとえば、Docker

### WSL2の場合

WSL2の場合は

https://devblogs.microsoft.com/commandline/whats-new-for-wsl-in-insiders-preview-build-18945/#use-localhost-to-connect-to-your-linux-applications-from-windows
