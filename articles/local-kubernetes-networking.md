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

それは、ローカル環境だからです。公式ドキュメントはあくまで**Kubernetesのドキュメントであり、Kubernetes自体を動かす環境固有の条件については言及していない**のです。

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

よって、minikubeの外側と内側の経路を作るのは `minikube service` コマンドとなります。

![](https://storage.googleapis.com/zenn-user-upload/093527f92fd4-20211219.png)

つまり、minikube内のPodへ外部からアクセスするには2段階の経路が必要になります。
なぜ2段階に分かれているのでしょうか、Serviceに直接アクセスではダメなのでしょうか？その理由を探ります。

## minikubeと仮想化技術

Kubernetesはminikube環境の中に閉じ込められていると書きましたが、minikube環境とはどういう実体なのでしょうか。

そもそもKubernetesは基本的にLinuxマシンで動かす想定のソフトウェアです。よって、WindowsやMacでは何らかの仮想化が必要です。
仮想的なLinuxマシンを動かすとなると、DockerやVirtualBox, Hyperkitなどが選択肢としてあがります。そこでminikubeはどの仮想化技術を使うか選択できる仕組みを持っていて、それぞれの技術はDriverと呼ばれます。次のドキュメントにDriverの一覧が書かれています。

[Drivers | minikube](https://minikube.sigs.k8s.io/docs/drivers/)

ページを見ると各OSごとに多くのDriverが用意されていることがわかります。

どのDriverを使っているのかは `minikube start` 時に出力されます。たとえばmacOSでhyperkit driverが選ばれた場合は次のような表示になります。
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
コマンドのソースを読むと、`NeedsPortForward` 関数の判断により必要に応じてPort Forwardingを自動的にやってくれています。

https://github.com/kubernetes/minikube/blob/69d5d34eeb4d5f3ad963c544773dd088d6550821/cmd/minikube/cmd/service.go#L95

## まとめ

minikubeはKubernetesに必要なLinux用のコンポーネントを仮想化技術を使って動かしています。
よって、**Kubernetesの外側にさらに仮想化の枠がついている** ため、Kubernetesの外部（ローカル環境のホストPC）からPodへアクセスするには2段階の経路を作る必要があります。

たとえばDocker driverを使うと、Dockerコンテナの上でKubernetesのコンポーネントが動きます。
Kubernetes自体がコンテナを管理するツールなのに、そのKubernetes自体がコンテナの中にあって……？？？となり難易度が高いですね。
その上Docker以外のdriverも多くあり、それぞれについてネットワークの経路を開ける方法が異なってたりします。

![](https://storage.googleapis.com/zenn-user-upload/2495bb6454f6-20211219.png)

さらに、最近はApple Silicon(M1)の登場により仮想化技術の互換性がなくなったり、[Docker Desktopの企業利用の有料化](https://www.docker.com/blog/updating-product-subscriptions/)の影響でWindows/MacにおけるDockerのプラクティスが定まらず情報が錯綜していて初学者にはつらい状況です。

とにかく、このあたりのネットワークの情報を調べるコツは、**どこまでがKubernetesの話で、どこからが仮想化技術(minikubeのDriver)の話なのかを意識すること** 、そして自分が何のDriverを使っているかを知っておくことだと思います。

Apple SiliconもDocker desktopの有料化も主に仮想化技術の方の話題であって、Kubernetes本体のServiceをはじめとするネットワークの概念はここ最近では変わっていません。
Kubernetesをまずはローカル環境で学習しようと試みてネットワーク周りで混乱するのはKubernetesの難しさではなく、その下にある仮想化技術の難しさかもしれません。

## その他細かい話

### kubectl port-forward

[Get Started](https://minikube.sigs.k8s.io/docs/start/)では `minikube service` コマンド以外の方法として、`kubectl port-forward` コマンドというのも紹介されています。

`kubectl port-forward` コマンドはまた別の方法を使って経路を作っています。
Serviceの名前で指定はできますが、内部的にはPodへの直通経路を開けている形になっているようです。

![](https://storage.googleapis.com/zenn-user-upload/309af368d76b-20211219.png)

詳しくは[kubectl port-forwardの仕組み | 日々修行](https://ytsuboi.jp/archives/614) の記事が参考になります。

### minikube tunnel

さらに応用として[minikube tunnel](https://minikube.sigs.k8s.io/docs/commands/tunnel/)コマンドがあります。
これを使うとなんと、`minikube service` コマンドや `kubectl port-forward` コマンドを使わずともServiceに直接アクセスができます。

// TODO:

### WSL2, Lima

[WSL2](https://docs.microsoft.com/ja-jp/windows/wsl/install)はWindows上で仮想Linuxマシンを動かす技術のひとつです。WSL2でminikubeを使いたい場合、複数のやり方があります。

- Docker desktopを使う方法
- WSL2上にLinux版Dockerを入れる方法

https://kubernetes.io/blog/2020/05/21/wsl-docker-kubernetes-on-the-windows-desktop/

WSL2自体も仮想マシンです。よってWSL2の内部はほぼLinuxです。
なので、WSL2のシェル上でminikubeを実行すると、Linux環境でminikubeを実行したのと近い環境になります。

さらに、WSL2側の工夫により `localhost` アドレスでWSL2内部に直接アクセスできる機能があります。これを使うとWindows側のWebブラウザからWSL2のLinux内で起動しているアプリケーションに http://localhost/... で直接アクセスできます。まるで仮想マシンなどないかのように感じてしまいますが、WSL2は仮想マシンであるという前提を忘れず常に意識することで、Kubernetesローカル環境の理解もしやすいはずです。

https://devblogs.microsoft.com/commandline/whats-new-for-wsl-in-insiders-preview-build-18945/#use-localhost-to-connect-to-your-linux-applications-from-windows

次に、[Lima](https://github.com/lima-vm/lima)はmacOS上で仮想Linuxマシンを動かす技術です。

LimaもWSL2似ています。LimaはmacOS上でLinuxの仮想マシンを実行していますが、WSL2と同様に `localhost` アドレスを自動的にLima内のLinuxアプリケーションに通してくれる機能が標準搭載されているため、mac側のWebブラウザから http://localhost/... で直接アクセスできます。

https://github.com/lima-vm/lima/blob/master/README.md#lima-linux-virtual-machines-on-macos-in-most-cases