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
本記事ではその難しさ（特にネットワークまわり）を整理し、ローカルKubernetes環境を攻略していきます。

## ローカルKubernetesを構築するツール

Kubernetesは本来単一のPCで動かすツールではありません。よってローカルで動作させるには何かしらローカル環境構築を補助してくれるツールを使います。代表的なものとして minikube や kind があります。

迷った場合はひとまず[minikube](https://minikube.sigs.k8s.io/)を使うとよいでしょう。
minikube公式ドキュメントの[Get Started](https://minikube.sigs.k8s.io/docs/start/)を読むとインストール、初期化、アプリのデプロイ、削除までを一通り体験できます。

たしかに上から順番にコマンドを実行していくと `hello-minikube` アプリを確認できますが、**アプリにWebブラウザからアクセスするだけでなぜこんなにコマンドが多いのか？** と疑問に感じるかもしれません。特にKubernetesをはじめて触る方はお手軽にローカル環境でやるつもりが初手から混乱してしまいそうです。

そこで、アプリへのアクセス経路という観点で順序立てて仕組みを見ていきましょう。

## Podへのアクセス経路

Kubernetes上でアプリが動く基本単位は**Pod**です。Kubernetes内のアプリにWebブラウザからアクセスできるということは、WebブラウザからPodへの経路があるはずです。

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

それは、ローカル環境だからです。公式ドキュメントはあくまで**Kubernetesのドキュメントであり、Kubernetes自体を動かす環境固有の条件については言及してません。**

実際はKubernetesクラスタはminikube環境に閉じ込められており、次のような構造になっています。

![](https://storage.googleapis.com/zenn-user-upload/c72988c24d9d-20211218.png)

よって、minikubeの外側からServiceへの経路が別途必要です。

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

よって、minikubeの外側と内側の経路を作るのは `minikube service` コマンドとなります。

![](https://storage.googleapis.com/zenn-user-upload/093527f92fd4-20211219.png)

つまり、minikube内のPodへ外部からアクセスするには2段階の経路が必要になります。
なぜ2段階に分かれているのでしょうか、Serviceに直接アクセスではダメなのでしょうか？その理由を探ります。

## minikubeと仮想化技術

Kubernetesはminikube環境の中に閉じ込められていると書きましたが、minikube環境とはどういう実体なのでしょうか。

そもそもKubernetesは基本的にLinux上で動かす想定のソフトウェアです。よって、WindowsやmacOSでは何らかの仮想化が必要です。
仮想的なLinuxマシンを動かすとなると、DockerやVirtualBox, Hyperkitなどが選択肢としてあがります。そこでminikubeはどの仮想化技術を使うか選択できる仕組みを持っていて、それぞれの技術はDriverと呼ばれます。次のドキュメントにDriverの一覧が書かれています。

[Drivers | minikube](https://minikube.sigs.k8s.io/docs/drivers/)

ページを見ると各OSごとに複数のDriverがあることがわかります。

どのDriverを使っているかは `minikube start` 時に出力されます。たとえばmacOSでhyperkit driverが選ばれた場合は次のような表示になります。
何も指定しない場合、minikubeが推奨するdriverを自動で選んでくれますが、これはフラグで変更することもできます。

```
$ minikube start
😄  Darwin 11.6 上の minikube v1.16.0
✨  hyperkitドライバーが自動的に選択されました
👍  コントロールプレーンのノード minikube を minikube 上で起動しています
🔥  hyperkit VM (CPUs=2, Memory=6000MB, Disk=20000MB) を作成しています...
...
```

つまり、仮想化技術は色々あってそれぞれ**外から内部への経路を通す方法が異なる** のです。
たとえば同じWindowsであっても、Hyper-V driverとDocker driverでは経路を通す方法が異なります。
この経路を通す処理を裏でいい感じにやってくれるのが `minikube service` コマンドです。
コマンドのソースを読むと、`NeedsPortForward` 関数の判断により必要に応じてPort Forwardingをしてくれるのがわかります。

https://github.com/kubernetes/minikube/blob/69d5d34eeb4d5f3ad963c544773dd088d6550821/cmd/minikube/cmd/service.go#L95

## Docker driverを使う

minikubeのDriverは多くありますが、現在は**Docker driver**を使うのが良さそうです。
[ドキュメントのDriversページ](https://minikube.sigs.k8s.io/docs/drivers/) でも (preferred) と書かれていますし、Windows, macOS, Linuxすべてに存在するからです。

Docker driverを使うと、Dockerコンテナの上でKubernetesのコンポーネントが動きます。
Kubernetesはコンテナを管理するツールなのにそれ自体がコンテナの中にある、というのはややこしいですが、Dockerに閉じ込めることで複数のOSで統一した管理ができるのです。

![](https://storage.googleapis.com/zenn-user-upload/2495bb6454f6-20211219.png)

Dockerもまた基本Linuxで動かす前提のソフトウェアなのでWindows, macOSで使うにはやはり何らかの仮想化技術が必要です。[Docker Desktop](https://www.docker.com/products/docker-desktop)はそのあたりをいい感じに裏で構築してくれるソフトウェアです。Docker Desktopを使えば自前で仮想化の構築をせずにDockerが使える状態にしてくれます。

しかし、[Docker Desktopの企業利用の有料化](https://www.docker.com/blog/updating-product-subscriptions/)の影響でWindows/macOSにおいてはDocker Desktopを使わずにDockerを動かしたいという需要も高まっています。
そこで、Docker desktopを使わず **Linuxの仮想マシンを自前でひとつ立てておいて、その上でLinux版のDockerを使う** という手法が注目されています。代表的な例がWSL2やLimaです。

Docker driverの時点でかなり階層が深いのに、さらにそれを仮想マシンで閉じ込めるので大変ややこしいですね。こうなるとPodへのアクセス経路はどうなるのでしょうか？

![](https://storage.googleapis.com/zenn-user-upload/02e5b50135f6-20211219.png)

実は、WSL2やLimaには仮想マシンをできるだけ意識せずにネットワーク到達ができる工夫がなされています。
よって3段階目の経路を別途用意する必要はありません。WSL2とLimaのネットワーク機能について見ていきます。

## WSL2, Limaのネットワーク機能

[WSL2](https://docs.microsoft.com/ja-jp/windows/wsl/install)はWindows上で仮想Linuxマシンを動かす技術のひとつです。
よってWSL2のシェル上でminikubeを実行すると、Linux上でminikubeを実行したのと近い環境になります。

WSL2には `localhost` アドレスからWSL2内部に直接アクセスできる機能があります。よって、`minikube service` コマンドの実行がWSL2内部であっても普通に外側からアクセスできます。

https://devblogs.microsoft.com/commandline/whats-new-for-wsl-in-insiders-preview-build-18945/#use-localhost-to-connect-to-your-linux-applications-from-windows

つづいて、[Lima](https://github.com/lima-vm/lima)はmacOS上で仮想Linuxマシンを動かす技術です。

LimaもWSL2と似ています。WSL2と同様に `localhost` アドレスをLima内に通してくれる機能が標準搭載されているため、DockerがLima内部にあっても、`minikube service` コマンドを使ってmacOS側のWebブラウザから直接アクセスできます。

Limaには標準でDockerがインストールされていませんが、少し手順を踏めばDockerとminikubeが導入できます。次のIssue commentでLimaへのminikube導入方法が紹介されています。

https://github.com/kubernetes/minikube/issues/12508#issuecomment-974646099

## まとめ

minikubeは仮想化技術を使ってローカルKubernetes環境を実現しています。
その仮想化部分の事情によりKubernetesの外部（ローカル環境のホストPC）からPodへアクセスするには2段階の経路を作る必要があります。

minikubeは現在Docker driverを推奨していますが、windows/macOS上でDockerを動かす方法も多様化しており今後の動向を見ておく必要があるでしょう。
WSL2やLimaなどの仮想マシンを作る手法には仮想マシンを意識せずに済むような工夫が入っているのでDocker Desktopに頼らない方法として最近は注目されています。

とにかく、このあたりの情報を調べるコツは、自分が何のDriverを使っているか知り、**どこまでがKubernetesの話で、どこからが仮想化技術(minikubeのDriver)の話なのかを意識すること** です。ローカル環境でKubernetesの学習をしてネットワーク周りで混乱するのはKubernetesの難しさではなく、その下にある仮想化技術の難しさかもしれません。

## 補足情報

### kind

本記事ではminikubeをメインに解説しましたが、同様にローカルにKubernetes環境を作るものとして[kind](https://kind.sigs.k8s.io/)も挙げました。kindは Kubernetes in Docker の略から名付けられたようで、minikubeのDocker driverと近いやり方を採用しています。
しかし、`minikube service` に当たる便利なコマンドはなく、クラスターの作成時に事前に経路を開けるなどの工夫が必要となります[^kind]。
よってLinux以外のOSではminikubeより難易度が高いと思います。

[^kind]: https://kind.sigs.k8s.io/docs/user/loadbalancer/ の最初にmacOSとWindowsについての制約が書かれています。

### kubectl port-forward

[Get Started](https://minikube.sigs.k8s.io/docs/start/)では `minikube service` コマンド以外の方法として、`kubectl port-forward` コマンドも紹介されています。
このコマンドは `minikube service` とはまた別の方法を使って経路を作っています。
Serviceの名前で指定はできますが、内部的にはPodへの直通経路を開けている形になっているようです。

![](https://storage.googleapis.com/zenn-user-upload/309af368d76b-20211219.png)

詳しくは[kubectl port-forwardの仕組み | 日々修行](https://ytsuboi.jp/archives/614) の記事が参考になります。

### minikube tunnel

応用的なものとして[minikube tunnel](https://minikube.sigs.k8s.io/docs/commands/tunnel/)コマンドがあります。
これを使うと `minikube service` コマンドや `kubectl port-forward` コマンドを使わずともServiceに直接アクセスができます。
まるで魔法のようですが、裏ではホストOSごとに分かれた複雑な処理を行っています[^1]。

LoadBalancerというServiceの種類があり、それを使ったアクセスを実現するためにはこの `minikube tunnel` が必要です。ですが、そもそもNodePortやLoadBalancerが何なのかわからない段階では使う必要はないと思います。

[^1]: https://github.com/kubernetes/minikube/tree/master/pkg/minikube/tunnel の `route_linux.go`, `route_darwin.go`, `route_windows.go` などを見ると雰囲気がわかります
