---
title: "Wineへコントリビュートする方法"
emoji: "🍷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wine"]
published: true
---

[Wine](https://www.winehq.org/)はWindows用のアプリをLinux/macOS上で動かすソフトです。
[LGPLライセンスでソースが公開されており](https://source.winehq.org/source/LICENSE)、基本誰でも[^1]コントリビュートできます。しかし、20年近く続いている巨大なプロジェクトでもあるので現代風にGitHubにPull Request出して終わり！という風にはできません。いくつかの手順を踏む必要があります。

コントリビュートのやり方については公式サイトの[Developer FAQ](https://wiki.winehq.org/Developer_FAQ)と[Submitting Patches](https://wiki.winehq.org/Submitting_Patches)に書かれています。
基本はこのガイドに従いつつ、実際にパッチを送信してみて感じた注意点や便利なツールなどを解説していきます。

## 対象のBugを見つける

まずはどんな貢献をしたいか決めることが重要です。Wineでは[Bugzilla](https://bugs.winehq.org/)でバグ一覧が管理されています。ここで修正したいバグを見つけられると良いでしょう。
修正対象のバグが明確だと、コードレビュアーもどういった意図でこの変更が入ったかわかりやすくなります。

[^1]: Microsoft Windowsのソースコードを見たことのある人は貢献できません https://wiki.winehq.org/Developer_FAQ#Who_can.27t_contribute_to_Wine.3F

## ソースコードをcloneする

[Wine本体のソースコードはGitで管理されている](https://wiki.winehq.org/Source_Code)ため、次のようにcloneできます。

```
git clone git://source.winehq.org/git/wine.git
```

そして、次にコミット時のユーザー名・メールアドレスを決めておきます。
**ユーザー名は本名である必要があります** [^2]。

```
git config user.name "Your Name"
git config user.email "me@example.com"
```

[^2]: Microsoftのコードを逆アセンブルするなどの違反行為を防ぐために責任をもたせるという意味合いがあるみたいです https://wiki.winehq.org/Submitting_Patches#Check_your_Git_setup


## ビルド・検証環境を準備する

さっそくWineのソースコードを修正したいところですが、WineはCで作られた巨大なソフトウェアでありビルドの依存関係が非常に多いです。[Building Wine](https://wiki.winehq.org/Building_Wine)というページで列挙されているのですが、これを気にしながら環境を構築するだけでも大変な作業です。

そこで、 **Dockerコンテナの中でWineをビルドできる環境** を用意しました。
これは実際に筆者がWineにパッチを送る際の動作検証に使ったものです。もしよければ利用してみてください。

https://github.com/castaneai/winebuilder

このwinebuilderを使うと、`make build` と `make up` の2コマンドで環境構築が完了します。
また、VNC接続に対応しているため、Wineで起動したGUIアプリの動作を確認することも可能です。
筆者はLinuxマシンで検証しましたが、Dockerを使っているのでおそらくWindows/Macでも実行可能だと思います。

```sh
# Wineの最新ソースをpullする
cd wine/
git pull origin master

# 環境をビルドする
cd ../
make build
make up

# Wineのビルド環境が揃ったコンテナにbashで入る
make bash

# コンテナの中に入ったら、wineをビルドする
CC='ccache gcc' PKG_CONFIG_PATH=/usr/lib/i386-linux-gnu/pkgconfig ./configure
make -j16
```

Wineは多くのソースコードを含むのでビルド時間が長いのですが、`make -j` オプションで並列数を上げると高速化できます。[このブログ記事](https://www.codeweavers.com/blog/aeikum/2019/1/8/working-on-wine-part-2-wines-build-process)によると、ビルドには20〜60分かかるとのことですが、筆者のPC(Ryzen 7 5800X)でビルドすると5分程度で終わりました。現代のCPUは凄いですね。
また、2回目以降は変更されたソース周辺のモジュールのみのビルドになるので速攻で終わります。

## コードを修正、デバッグする

ビルド環境構築が終わったら、いよいよ実際にコードを修正していきます。
コードの修正が終わったら `make -j` でもう一度ビルドをして `./wine test.exe` のようにビルド完了したwineで何らかのアプリを起動する形になります。

Wineは基本GUIアプリを動かすものなので、ステップ実行によるデバッグよりも原始的なprintデバッグのほうがやりやすいと思います。
大抵のソースでは `TRACE()` や `WARN()` といったデバッグログを出す用のマクロが用意されており、これをprintfのように呼び出すことでログ出力ができます。TRACEログを出力するためには環境変数 `WINEDEBUG=+all` などが必要です。WINEDEBUGによるログ制御について詳しくは[Debug Channels](https://wiki.winehq.org/Debug_Channels)を参照してください。

```c
TRACE("test message: %d", value);
WARN("test message: %d", value);
```

## テストを実行する

また、Wineには数多くのテストコードも用意されています。各モジュールのディレクトリにはほぼ `tests/` というディレクトリがあり、そこにテストコードが入っています。

テストを実行するには `tools/runtest` というスクリプトを利用します。引数が多いですが `./tools/runtest -h` と実行してUsageを出すと説明があります。
たとえば `dsound` モジュールの `tests/dsound.c` のテストを実行したい場合は次のようなコマンドになります。

```
./tools/runtest -P wine -T . -M dsound.dll -p dlls/dsound/tests/dsound_test.exe dlls/dsound/tests/dsound.c
```

## パッチを送信する

現代的なGitHubなどのプロジェクトであれば、コードを修正した後は通常通りcommitしてpushしてPull Requestを出してとさくさく進むのですが、**Wineではメーリングリストにパッチ（変更差分）を送信する方式になります。**
やや面倒ではありますが、公式サイトに丁寧な解説があるので基本はこれに従えばOKです。

[Submitting Patches - WineHQ Wiki](https://wiki.winehq.org/Submitting_Patches)

簡単に流れを箇条書きするとこのようになります。

- [Wine開発者メーリングリスト (wine-devel)](https://www.winehq.org/forums)に登録する
- `git commit -s` で `Signed-off-by` 付きのcommitをする
- `git format-patch origin` でパッチファイルを作成する
- `git send-email` でパッチファイルをWineの開発者メーリングリストへ送信する

この方式はWine固有ではなく、Linuxカーネル等でも採用されているので、日本語だと次の記事なども参考になります。

- [とあるエンジニアの備忘log: パッチを投稿しよう！](https://masahir0y.blogspot.com/2013/06/blog-post_4.html)
- [nekomatu's Blog: linux(kernel)にgit send-emailでパッチを送る方法](https://nekomatu.blogspot.com/2014/05/howto-send-patch-to-linux.html)

### 英語力がないと厳しい？

そんなことはないです。筆者は普通に[DeepL](https://www.deepl.com/ja/translator)で機械翻訳した文章を突っ込みましたが、レビューで「意味がわからん」とは言われなかったので伝わっていると思います。

もし説明があまりに複雑になるのなら、そもそもパッチの内容が複雑すぎる可能性が高いです。英文を見直す前にパッチの変更内容を母国語で簡潔に短く説明できるかどうかを確認したほうがよいでしょう。


## パッチ送信後の動き（テストとレビュー）

上記のドキュメントに従ってパッチをWineの開発者メーリングリストに送信すると数分後に[Patches list](https://source.winehq.org/patches/)というページに送信したコミットが登録されます。すると自動的にCIサーバー的な場所でテストが実行され、しばらく経つとテストの結果が出ます。この表でいうと `Testbot` というカラムの部分です。ここが `OK` になればテスト成功です。逆に `Failed` になった場合はテスト失敗です。どのテストに失敗したかは Failed をクリックすると詳細を見れます。

また、送信したパッチは最初 `New` というステータスですが、しばらく（1〜2日ほど？）待つとここが `Assigned` に変わります。これはレビュアーが割り当てられたということなので、コードレビューの結果を待ちましょう。一週間以内には大抵レビューされると思います。

## パッチ送信後に修正をしたい場合

パッチ送信した結果、テストに失敗したりレビューで指摘を受けた場合はパッチを修正する必要があります。
その場合は手元でコードを修正し、バージョン2のパッチを送信します。

次のコマンドで `[PATCH v2]` のようにバージョンが上がったパッチを作成できます。

```sh
# version 2ということを示すために -v2 optionsをつける
git format-patch origin -v2
```

バージョンが上がったパッチを開発者メーリングリストに再送することで修正版のパッチ送信は完了です。
これを繰り返しテストとレビューが通ったら、レビュアーから `Signed-off-by` を付けてもらえます。そうなれば数日後に本流のmasterブランチにマージされます！
マージされたら次のようなメールが届きます。

```
Thank you for your contribution to Wine!

This is an automated notification to let you know that your patch has
been reviewed and its status set to "Committed".

This means that your patch has been approved, and committed to the
main git tree. Congratulations!
```
