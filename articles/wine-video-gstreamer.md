---
title: "Wineの動画再生のしくみ、DirectShowとGStreamerの関係"
emoji: "🍷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wine", "gstreamer", "directshow", "linux", "windows"]
published: false
---

## Wineとは

Windows用のアプリをLinux/macOS上で動かすソフトです。
長年開発されており、結構な数のWindows向けアプリケーションが動きます。

## Wineの動画再生

そんなWineですが、動画や音声といったマルチメディア系の処理は苦手なようです。
たとえば検索エンジンで「Wine 動画再生」と調べると、だいたい動くが動画再生がおかしい、だったり、追加で特殊な作業をしないと動画が再生されない、といった記事がいくつもヒットします。

これはWineが未熟というより **動画・音声コーデックの世界とWindows APIの歴史が複雑すぎる** からだと思っています。そんな複雑なOS向けに作られたアプリをまた別のOSで動かさなければならないので、そりゃ大変だわ！って感じですね。

筆者もWineでゲームを動かそうとしたものの動画がうまく再生されず、なんとか解決できないかと最近調べたことがありました。その過程でWineの動画再生のしくみが見えてきたので一部を解説します。

## 前提条件

WineはLinuxとmacOSで動きますがLinuxとmacOSでは動画や音声の再生のAPIも異なります。macOSについては実際に動かしたことがないのでLinuxでの再生に限定して説明します。

また、筆者が検証した環境とソフトウェアのバージョンは次のとおりです。

- Ubuntu 20.10
- Wine 6.3
- GStreamer 1.18.0

## 最小限のサンプルアプリ

動画再生の仕組みを調べるには、できるだけ動画再生以外の余計な情報は入らないほうが良いでしょう。
そこで、次のようなアプリを準備しました。引数で与えた動画ファイル(MPEG-1)を再生するだけの単純なWindowsアプリです。実際のコードはほとんど[MPEGファイルを再生する:Geekなぺーじ](https://www.geekpage.jp/programming/directshow/renderfile.php)を真似しました。

https://github.com/castaneai/mpegplaytest/blob/master/main.cpp

Wineで実行するため、Windows環境でビルドしてできあがったexeファイルだけをLinuxに持っていく形となります。

たとえば https://filesamples.com/formats/mpeg で配布されているサンプルのmpegファイルを再生すると以下のように動画が再生されます[^mpeg]。OKボタンを押すと終了します。

```sh
wine ./mpegplaytest.exe sample_640x360.mpeg
```

![](https://storage.googleapis.com/zenn-user-upload/o10hhtcm165obvumjedjnrhpk9jd)

[^mpeg]: Wine6.3だと動画のフレームレートが安定しなくてカクカクしますが、この現象はすでにWineのバグチケットに上がっていて現在調査中みたいです。

## DirectShow

このアプリが使っているのは[DirectShow](https://ja.wikipedia.org/wiki/DirectShow)というフレームワークになります。
DirectShowを使うと様々な形式の動画や音声を統一的なAPIで扱えます。[公式ドキュメント](https://docs.microsoft.com/ja-jp/windows/win32/directshow/directshow-system-overview)の図を引用すると次のような構造になっているようです。

![](https://storage.googleapis.com/zenn-user-upload/zbvoj9j3h0fxksl0z03by5336mg7)

あまり細かく理解する必要はないですが、内部にFilterというものがあり、異なる動画形式の入力であっても適切なFilterを見つけて生の映像データに変換をしてくれて最終的にいい感じに画面に出力されるという仕組みです。

Windowsのアプリで動画を再生するならば大抵はこのDirectShowか、後継の[Media Foundation](https://ja.wikipedia.org/wiki/Media_Foundation)を使っているはずです[^mf]。

[^mf]: Media Foundationについては調査できてないのでここではDirectShowに絞って解説します。

## LinuxでDirectShowを...!?

当たり前ですが、LinuxにDirectShowはありません。Windowsの機能だからです。

WineはWindowsのアプリを動作させるものなのでDirectShowを使ったアプリももちろん動かしたいところです。
しかし、DirectShowと同じものを全部Linux用に書き直すのはとてつもなく大変でしょう。動画や音声には様々な形式がありAPIも日々進化しているからです。
そこで **GStreamer** の出番です。

https://ja.wikipedia.org/wiki/GStreamer

GStreamerはLinuxをはじめとした環境で長年使われている動画・音声を扱うフレームワークです。
様々な形式の動画や音声を統一的なAPIで扱えます。[公式ドキュメント](https://gstreamer.freedesktop.org/documentation/tutorials/basic/dynamic-pipelines.html?gi-language=c)の図を引用すると次のような構造になっているようです。


![](https://storage.googleapis.com/zenn-user-upload/oftemz4vk8ytk59s6lseceh67gah)

内部にはElementという細かな部品があり、入力が異なる形式であっても各種Elementが変換してくれて最終的にいい感じに画面等に出力してくれます。

あれ・・？さっきも同じようなものを見た気がしますね。そう、**DirectShowとGStreamerは似たもの同士** だったのです。

## WineGStreamer

Wineの話に戻ります。Wineで動画や音声を再生したい場合、DirectShowのAPIを再実装する必要があります。しかし、DirectShowの内部は複雑でさまざまな形式の動画に対応するのは大変です。
そこで、**外面はDirectShowに見せかけておいて、内部ではGStreamerを使う** という方法が生み出されました。

Windows用に作られたアプリはもちろんDirectShow APIを使っているので、入口はDirectShow APIである必要があります。しかし、その内部で画面に移すための生の映像データに変換する部分はGStreamerに任せちゃったらいいじゃん！という考え方です。

Wineではこの役割を担う **WineGStreamer** というライブラリが作られました。ソースコードでいうと `dlls/winegstreamer` ディレクトリ以下になります。

https://github.com/wine-mirror/wine/tree/master/dlls/winegstreamer

本来のWindowsのDirectShowでは、次のように入力に合わせたDirectShow Filterが選択され適切な変換をしてくれるイメージです[^1]。

![](https://storage.googleapis.com/zenn-user-upload/hd56kui3ext4mw60lhdlv312sbr1)

一方、WineGStreamerは **GStreamerに変換処理をさせるDirectShow Filter** を間に挟むことでGStreamerとの共存を実現しています[^2]。

![](https://storage.googleapis.com/zenn-user-upload/i6frm1sjwauutl54wkkcnhzy6m2e)

もうすこし具体的に見てみましょう。WineGStreamerのメイン処理の `DllGetClassObject` ではDirectShowフィルターを登録する処理を行っています。その中に `deocodebin_parser` というフィルタの登録が見られます。

```c
HRESULT WINAPI DllGetClassObject(REFCLSID clsid, REFIID iid, void **out)
{
    struct class_factory *factory;
    HRESULT hr;

    TRACE("clsid %s, iid %s, out %p.\n", debugstr_guid(clsid), debugstr_guid(iid), out);

    if (!init_gstreamer())
        return CLASS_E_CLASSNOTAVAILABLE;

    if (SUCCEEDED(hr = mfplat_get_class_object(clsid, iid, out)))
        return hr;

    if (IsEqualGUID(clsid, &CLSID_AviSplitter))
        factory = &avi_splitter_cf;
    else if (IsEqualGUID(clsid, &CLSID_decodebin_parser))
        factory = &decodebin_parser_cf;

...
```

https://github.com/wine-mirror/wine/blob/4d5824112e13160e538013a25f1c13a124565180/dlls/winegstreamer/main.c#L151

decodebin とはGStreamerのプラグインのひとつで、**「入力に対して自動で形式を判別して適切なElementへ接続してくれる」** という便利なものです。

https://gstreamer.freedesktop.org/documentation/playback/decodebin.html?gi-language=c

したがって、Windows用アプリがDirectShowに流した動画や音声は形式が何であれGStreamerのdecodebinに丸投げして変換は任せるという挙動になります。

[^1]: 実際はひとつのFilterを通るだけではなく、何層ものFilterを通りますがイメージということで簡略化しています。
[^2]: GStreamerに頼らずWine独自で書かれたFilterも存在しますが、メインで使うことはあまりなさそうなので省略します。

## GStreamerからWineへ返す経路

様々な形式の動画をGStreamerに丸投げしましたが、最終的な画面の出力にはWine側のAPI呼び出しが必要です。
つまりGStreamerでいい感じに変換された生の映像データをWineに返す必要があります。

GStreamerはAPIを使ってプログラムから動画や音声のデータを受け渡しできます。
よってWineGStreamerは `decodebin_parser` からGStreamerに変換前のデータを流し、変換後のデータが `qz_sink` という要素[^quartz]を通して再びWine側に返ってくるパイプラインを構築します。

![](https://storage.googleapis.com/zenn-user-upload/990g1a92pale9zyudou5pk4riehx)

[^quartz]: WineGStreamer周辺には quartz や qz という単語が出現しますが、これはWindowsにおけるDirectShowの実装が入ったDLLの名前です。

## GStreamerが動いていることを確認

実際にGStreamerで動いていることを確認するために最初に紹介したサンプルアプリ(mpegplaytest.exe)を使ってみます。

GStreamerは環境変数 `GST_DEBUG` を指定することで様々なログを出力できます。

https://gstreamer.freedesktop.org/documentation/tutorials/basic/debugging-tools.html?gi-language=c#the-debug-log

Wine内部でGStreamerが使われているか確認するため、`GST_DEBUG=5` などの環境変数を入れた状態で起動してみると、動画再生中たくさんのログが出るはずです。また、その中から `creating element` などの文字列を抽出するとWineの内部でどのようなGStreamer Elementが使われているか確認できたりします。

```sh
$ GST_DEBUG=5 wine ./mpegplaytest.exe sample_640x360.mpeg 2>&1 | grep 'creating element'

0:00:00.038414127 38205 0x7d77da60 INFO     GST_ELEMENT_FACTORY gstelementfactory.c:363:gst_element_factory_create: creating element "bin"
0:00:00.039412009 38205 0x7d77da60 INFO     GST_ELEMENT_FACTORY gstelementfactory.c:363:gst_element_factory_create: creating element "decodebin"
0:00:00.040163321 38205 0x7d77da60 INFO     GST_ELEMENT_FACTORY gstelementfactory.c:360:gst_element_factory_create: creating element "typefind" named "typefind"
0:00:00.067474252 38205 0x7d98e458 INFO     GST_ELEMENT_FACTORY gstelementfactory.c:363:gst_element_factory_create: creating element "protonvideoconverter"
0:00:00.068273564 38205 0x7d98e458 INFO     GST_ELEMENT_FACTORY gstelementfactory.c:363:gst_element_factory_create: creating element "mpegpsdemux"
0:00:00.069750308 38205 0x7bb89a18 INFO     GST_ELEMENT_FACTORY gstelementfactory.c:363:gst_element_factory_create: creating element "multiqueue"
0:00:00.072502475 38205 0x7bb89a18 INFO     GST_ELEMENT_FACTORY gstelementfactory.c:363:gst_element_factory_create: creating element "mpegvideoparse"
0:00:00.079244932 38205 0x7bb69cb8 INFO     GST_ELEMENT_FACTORY gstelementfactory.c:363:gst_element_factory_create: creating element "protonvideoconverter"
0:00:00.236462223 38205 0x7bb69cb8 INFO     GST_ELEMENT_FACTORY gstelementfactory.c:363:gst_element_factory_create: creating element "nvmpegvideodec"
0:00:00.383085677 38205 0x7bb69cb8 INFO     GST_ELEMENT_FACTORY gstelementfactory.c:363:gst_element_factory_create: creating element "deinterlace"
0:00:00.384082339 38205 0x7bb69cb8 INFO     GST_ELEMENT_FACTORY gstelementfactory.c:363:gst_element_factory_create: creating element "videoconvert"
0:00:00.384719941 38205 0x7bb69cb8 INFO     GST_ELEMENT_FACTORY gstelementfactory.c:363:gst_element_factory_create: creating element "videoflip"
0:00:00.384958342 38205 0x7bb69cb8 INFO     GST_ELEMENT_FACTORY gstelementfactory.c:363:gst_element_factory_create: creating element "videoconvert"
```

内部ではそのままGStreamerが動いているため、[GstShark](https://developer.ridgerun.com/wiki/index.php?title=GstShark)などのGStreamerのための補助ツールも使えます。
Wineで起動しているアプリの動画再生がおかしい・・・という時に原因を切り分けるのに役立ちます。

以下の図は実際にうちのPCでwine内の動画再生をGstSharkでプロットしたものになります。

![](https://storage.googleapis.com/zenn-user-upload/wp00pywmww30eufucraqiccy8fqk)

## まとめ

Wineの中でも特に複雑な部類になりそうな動画再生まわりについて掘り下げました。
Windows用のDirectShowとLinux用のGStreamer、異なるOSの似ているフレームワークが上手くコラボレーションしているという事実は面白いですよね。

この記事を書いている時点でもMPEG-1動画のフレームレートがおかしかったりとまだまだ完璧とは言えないWineGStreamerですが、2021年3月現在でも活発に動画や音声まわりの改善は行われています。今後のWineの発展が楽しみですね。

そして、この記事をきっかけにWineの実装に興味を持ってもらえたり、その先で不具合の報告やWine本体への貢献につながったりしたら嬉しいと思ってます！🍷🍷
