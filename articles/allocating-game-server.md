---
title: "Game Serverの「割り当て」設計"
emoji: "🎮"
type: "tech"
topics: ["game", "gamebackend"]
published: false
---

オンラインゲームのサーバーに関する設計の情報はWebアプリに比べるとまだまだ少ない印象です。Zennの記事執筆の練習も兼ねてゲームサーバーの管理、特に今回は新しいGame Serverを割り当てるという部分に絞ってまとめます。

## API ServerとGame Server

オンラインゲームのサーバーを作るにあたって **API Server** と **Game Server** [^1]を分けて設計することがよくあります。
API Serverは主にログイン処理やホーム画面と言った共通的な処理を担当し、
Game Serverはゲームのメイン部分（たとえばFPSゲームであれば戦闘部分）を担当するといった組み合わせです。

![](https://storage.googleapis.com/zenn-user-upload/v277ztks4w5voo9zwa8kzubdwaae)

API Serverは前述の通りゲームのメイン部分ではなくリアルタイム性も少ないので
実装が容易でスケールしやすいHTTP Serverで組むことが多いです。
一方で、Game Serverはリアルタイム性が要求され通信プロトコルも専用で設計されることが多いです。このあたりの話は次の資料を見るとわかりやすいでしょう。

[Building the Game Server both API and Realtime via c#](https://www.slideshare.net/neuecc/building-the-game-server-both-api-and-realtime-via-c)

この資料にあるように、セキュリティや多人数接続を考えるとP2P（ゲームクライアント同士が直接接続）よりもDedicated Server（この記事でいうGame Server）を用意するのが主流です。

[^1]: 「API Server」や「Game Server」という呼び方は標準化されたものではないですが本記事ではこう呼びます

## Game Serverの割り当て

Game Serverはリアルタイム性が求められるためプレイが終わるまではゲームの状態を保持しつづける必要があります。ここがステートレス（リクエストに対してレスポンスを返して終わり）のHTTP Serverとは異なります。

しかし、**ゲームが開始してはじめてサーバーが必要になる** ため、常に起動し続ける必要はありません[^2]。サーバーリソースには限りがあるためGame Serverは必要なときだけ立てて使い終わったら消すのが理想的です。よって「今からゲーム開始したいのでGame Serverひとつください！」という要求を飛ばしてGame Serverを割り当てる処理を作ります。

プレイヤーは大抵はじめにAPI Serverを通してログイン処理などを行ってからゲームに参加します。となると、Game Serverの割り当てはAPI Serverを通して行うことになります。

![](https://storage.googleapis.com/zenn-user-upload/sjrga32ar3xiyrpfxlu4lou1am8o)

[^2]: MMORPG等のゲーム世界がずっと維持されているタイプのゲームは例外

## どこから割り当てるか

Game Serverはどこから割り当てるべきでしょうか。
単純に考えると用意したインフラ環境からVM（やコンテナ）を1台分払い出して渡すイメージです。

より高度なゲームになると「どこから割り当てるか」もチューニングする必要がでてきます。
たとえば、複数の国からプレイできるゲームであれば割り当てを要求したプレイヤーと地理的に近いサーバーを割り当てる必要があるかもしれません。
具体的な技術だとAmazon GameLiftの[Create Game Sessions with Queue](https://docs.aws.amazon.com/ja_jp/gamelift/latest/developerguide/gamelift-sdk-client-queues.html)やGoogle Cloud Game Serversの[Realm](https://cloud.google.com/game-servers/docs/concepts/overview#realm)等が当てはまります。


## Dangling Game Server問題

しかし、上で挙げた方法では問題が生じる可能性があります。
新しいGame Serverを割り当てた後、API Server内部でエラーが発生し割り当てたGame Serverの情報をプレイヤーに返せなかったらどうなるでしょうか？また、返せたとしてもプレイヤーが何らかの理由でゲーム参加前に消えてしまったらどうなるでしょうか？

ゲームのルール的には不参加プレイヤーとして扱えば済みますが、割り当てたサーバーが残ったままです。このまま1人目のプレイヤーが接続しない場合使われていないGame Serverが残り続けてリソースの無駄遣いになってしまいます。
この現象はGoogle主導のGame Server管理OSSである[Agonesのissueでも話題になり](https://github.com/googleforgames/agones/issues/607)、Dangling Game Serverと呼ばれています。

プレイヤーやAPI Serverがネットワーク越しで通信をするためどうしてもこのような問題は発生します。そこで、AgonesのissueではGame Serverに接続中の人数を定期的にチェックして、一定時間以上誰も参加しない場合はDangling serverとみなして再び落とすという解決策が提案されています。

## 割り当てたGame Serverの管理

2人目のプレイヤーが同じゲームに参加する場合を考えると割り当てたGame Serverの情報をどこかに保存しておく必要があります。割り当てと同時にDBにGame Serverの情報を書き込むのが一般的でしょうか。そうすることでAPI Serverは2人目のゲームへの参加要求を受け取るとDBを見て最初に作られたGame Serverへと誘導できます[^3]。

![](https://storage.googleapis.com/zenn-user-upload/x1waww4eot9jio0nz0hixr1asj46)

ここではひとつ考慮すべき点があって、最初はプレイヤー数0のまま記録しておくことです。割り当てと同時に1人目のプレイヤーの参加はほぼ確定してるので最初から1でいいのでは、と思いそうになりますが


[^3]: ゲームによってはここで適切なマッチメイキングが必要ですね














