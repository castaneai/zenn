---
title: "Pion WebRTC で RTCP Feedback を受け取る"
emoji: "️🎞️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["webrtc", "rtp", "pion", "go"]
published: true
---

WebRTCでは[RTCP Feedback](https://tools.ietf.org/id/draft-dt-rmcat-feedback-message-04.html)という仕組みが利用できます。
これは、リアルタイムに送受信している映像や音声の通信状況を相手に伝えることで品質や転送量の制御に役立てることができるというものです。詳しい内容については[好奇心旺盛な人のためのWebRTC - メディア・コミュニケーション](https://webrtcforthecurious.com/ja/docs/06-media-communication/#rtcp)を読むのがオススメです。

このRTCP Feedbackですが、Webブラウザ同士であれば（多分）内部でいい感じにやってくれているのであまり意識することはありません。しかし、[pion/webrtc](https://github.com/pion/webrtc)のように非Webブラウザから映像をストリーミングしたい場合はRTCP Feedbackによる制御を独自で実装する必要があるかもしれません。
今回はこのRTCP Feedbackを実際に受け取るプログラムを書いてストリーミングの状況をみてみます。

## pion/webrtc を体験する

[pion/webrtc]()を使うとGoで動くプログラムからWebRTCの映像をお手軽に配信できます。

たとえば[pion/rtwatch](https://github.com/pion/rtwatch)というデモでは、手元にある動画ファイルをWebRTCで配信できます。READMEにしたがってGStreamerのインストールを済ませた後、
`main.go` を実行し、 http://localhost:8080 にアクセスすると動画再生画面が開きます。
一見は普通のvideoタグですが、裏では main.go からWebRTC経由で配信されています。

![](https://storage.googleapis.com/zenn-user-upload/ktm8gqi5g1bsqbxth8jp1a3mmyxw)


## 実際にRTCP Feedbackパケットを受け取る

さて、このrtwatchのデモアプリではどのようなRTCP Feedbackが飛んできているのでしょうか。
pion/webrtcには受信したRTCPのパケットを取得できる実装があるので、それを使います。

RTCPを受信するには `webrtc.RTPSender`[^rtcpsender] という構造体を使います。
動画は映像と音声でトラックが分かれていますが、今回はひとまず映像トラックからRTCP Senderを取り出します（AddTrackの返り値から取得できます）。

取り出した後は、goroutineを使って裏でずっと受け取り続けるようにしています。

```diff
diff --git a/main.go b/main.go
index fda0a0d..53047de 100644
--- a/main.go
+++ b/main.go
@@ -1,6 +1,7 @@
 package main
 
 import (
+       "encoding/hex"
        "encoding/json"
        "flag"
        "fmt"
@@ -198,10 +199,23 @@ func serveWs(w http.ResponseWriter, r *http.Request) {
        } else if _, err = peerConnection.AddTrack(audioTrack); err != nil {
                log.Print(err)
                return
-       } else if _, err = peerConnection.AddTrack(videoTrack); err != nil {
+       }
+       rtpSender, err := peerConnection.AddTrack(videoTrack)
+       if err != nil {
                log.Print(err)
                return
        }
+       go func() {
+               for {
+                       rtcpBuf := make([]byte, 1500)
+                       n, _, err := rtpSender.Read(rtcpBuf)
+                       if err != nil {
+                               log.Print(err)
+                               return
+                       }
+                       log.Printf("RTCP packet received: %s", hex.Dump(rtcpBuf[:n]))
+               }
+       }()
 
        defer func() {
                if err := peerConnection.Close(); err != nil {
```

実行すると早速受け取ったRTCPパケットの中身が次々と出力されました。

```
$ go run main.go -container-path=bbb.mp4
Video file 'bbb.mp4' is now available on ':8080', have fun! 
2021/05/16 17:34:26 RTCP packet received: 00000000  81 ce 00 02 00 00 00 01  42 bb 9e 16              |........B...|
2021/05/16 17:34:26 RTCP packet received: 00000000  81 ce 00 02 00 00 00 01  42 bb 9e 16              |........B...|
2021/05/16 17:34:26 RTCP packet received: 00000000  81 c9 00 07 00 00 00 01  42 bb 9e 16 00 00 00 00  |........B.......|
00000010  00 00 fc a4 00 00 00 23  00 00 00 00 00 00 00 00  |.......#........|
2021/05/16 17:34:27 RTCP packet received: 00000000  81 c9 00 07 00 00 00 01  42 bb 9e 16 00 00 00 00  |........B.......|
00000010  00 00 fd bc 00 00 00 2c  00 00 00 00 00 00 00 00  |.......,........|
2021/05/16 17:34:29 RTCP packet received: 00000000  81 c9 00 07 00 00 00 01  42 bb 9e 16 00 00 00 00  |........B.......|
00000010  00 00 ff 06 00 00 00 29  00 00 00 00 00 00 00 00  |.......)........|
2021/05/16 17:34:30 RTCP packet received: 00000000  81 c9 00 07 00 00 00 01  42 bb 9e 16 00 00 00 00  |........B.......|
00000010  00 01 00 46 00 00 00 33  00 00 00 00 00 00 00 00  |...F...3........|
2021/05/16 17:34:31 RTCP packet received: 00000000  81 c9 00 07 00 00 00 01  42 bb 9e 16 00 00 00 00  |........B.......|
00000010  00 01 01 37 00 00 00 29  00 00 00 00 00 00 00 00  |...7...)........|
2021/05/16 17:34:31 RTCP packet received: 00000000  81 c9 00 07 00 00 00 01  42 bb 9e 16 00 00 00 00  |........B.......|
00000010  00 01 01 48 00 00 00 29  00 00 00 00 00 00 00 00  |...H...)........|
00000020  8f ce 00 05 00 00 00 01  00 00 00 00 52 45 4d 42  |............REMB|
00000030  01 0f 88 f6 42 bb 9e 16                           |....B...|
2021/05/16 17:34:31 RTCP packet received: 00000000  81 c9 00 07 00 00 00 01  42 bb 9e 16 00 00 00 00  |........B.......|
00000010  00 01 01 74 00 00 00 28  00 00 00 00 00 00 00 00  |...t...(........|
00000020  8f ce 00 05 00 00 00 01  00 00 00 00 52 45 4d 42  |............REMB|
00000030  01 0f 96 ff 42 bb 9e 16                           |....B...|
```

[^rtcpsender]: パケットを受信する機能を持つのに"Sender"という名前なのは違和感があるかもしれませんが、おそらく映像の送信者側であるという意味でのSenderです。

## RTCP Feedbackパケットを構造体に変換する

これでRTCP Feedbackを受け取れましたが、我々は人間なのでバイナリをそのまま渡されても困ります。ここから人間の読みやすい構造に解釈する必要があります。
そこで[pion/rtcp](https://github.com/pion/rtcp)パッケージの出番です。`rtcp.Unmarshal` を使うと生のRTCPバイナリを適切な構造体に変換できます。

RTCP受信のgoroutineを次のように変えてみます。

```go
go func() {
    for {
        rtcpBuf := make([]byte, 1500)
        n, _, err := rtpSender.Read(rtcpBuf)
        if err != nil {
            log.Print(err)
            return
        }
        packets, err := rtcp.Unmarshal(rtcpBuf[:n])
        if err != nil {
            log.Print(err)
            return
        }
        for _, packet := range packets {
            log.Printf("RTCP packet received: %T", packet)
        }
    }
}()
```

実行すると今度は意味のありそうな単語が次々と出力されました！

```sh
$ go run main.go -container-path=bbb.mp4
Video file 'bbb.mp4' is now available on ':8080', have fun! 
2021/05/16 17:51:42 RTCP packet received: *rtcp.PictureLossIndication
2021/05/16 17:51:42 RTCP packet received: *rtcp.PictureLossIndication
2021/05/16 17:51:42 RTCP packet received: *rtcp.PictureLossIndication
2021/05/16 17:51:42 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:51:42 RTCP packet received: *rtcp.PictureLossIndication
2021/05/16 17:51:43 RTCP packet received: *rtcp.PictureLossIndication
2021/05/16 17:51:44 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:51:45 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:51:45 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:51:46 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:51:47 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:51:48 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:51:48 RTCP packet received: *rtcp.ReceiverEstimatedMaximumBitrate
2021/05/16 17:51:48 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:51:48 RTCP packet received: *rtcp.ReceiverEstimatedMaximumBitrate
```

ここで最初の方で引用した[好奇心旺盛な人のためのWebRTC - メディア・コミュニケーション](https://webrtcforthecurious.com/ja/docs/06-media-communication/#rtcp)をもう一度読んでみると、まさにそのままの単語が登場します。

- Picture Loss Indication (**PLI**)
- Receiver Report
- Receiverr Estimated Maximum Bitrate (**REMB**)

この中でもわかりやすい例が **REMB** パケットです。
REMBパケットは映像を受信した側から送信側へ「これくらいのビットレートで送っていいよ」と受信可能な推定ビットレートを伝えるものなので中にビットレートの数値が入っています。

REMBの推定ビットレートを出力するように実装を追加してみます。

```go
go func() {
    for {
        rtcpBuf := make([]byte, 1500)
        n, _, err := rtpSender.Read(rtcpBuf)
        if err != nil {
            log.Print(err)
            return
        }
        packets, err := rtcp.Unmarshal(rtcpBuf[:n])
        if err != nil {
            log.Print(err)
            return
        }
        for _, packet := range packets {
            switch rtcpPacket := packet.(type) {
            case *rtcp.ReceiverEstimatedMaximumBitrate:
                log.Printf("Estimated Bitrate: %d", rtcpPacket.Bitrate)
            default:
                log.Printf("RTCP packet received: %T", packet)
            }
        }
    }
}()
```

実行すると、開始数秒後から推定ビットレートが出力されました。少しずつ上がってることがわかります。
この情報を使ってエンコード側のプログラムにビットレート変更の処理を入れるなどをすればRTCP Feedbackを使った独自の制御が実装できます。

```sh
$ go run main.go -container-path=bbb.mp4
Video file 'bbb.mp4' is now available on ':8080', have fun! 
2021/05/16 17:59:12 RTCP packet received: *rtcp.PictureLossIndication
2021/05/16 17:59:12 RTCP packet received: *rtcp.PictureLossIndication
2021/05/16 17:59:13 RTCP packet received: *rtcp.PictureLossIndication
2021/05/16 17:59:13 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:59:13 RTCP packet received: *rtcp.PictureLossIndication
2021/05/16 17:59:14 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:59:14 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:59:15 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:59:16 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:59:17 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:59:18 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:59:18 Estimated Bitrate: 1884352
2021/05/16 17:59:18 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:59:18 Estimated Bitrate: 1913576
2021/05/16 17:59:19 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:59:19 Estimated Bitrate: 1943256
2021/05/16 17:59:19 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:59:19 Estimated Bitrate: 1973400
2021/05/16 17:59:19 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:59:19 Estimated Bitrate: 2004008
2021/05/16 17:59:19 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 17:59:19 Estimated Bitrate: 2035096
```

## ひどいネットワークを再現してみる

RTCP Feedbackの変化をより明確にするため、映像送信処理に意図的にパケットロスを仕込んでみます。
rtwatchではGStreamerを使っているので、[netsim](https://gstreamer.freedesktop.org/documentation/netsim/index.html)というElementを挟むと簡単にネットワークの遅延やパケットロスを再現できます。

極端にひどいネットワークのほうが結果がわかりやすいので、今回は `netsim drop-probability=0.5` を間に挟んで50％のパケットロスを再現させてみます。

```diff
diff --git a/gst/gst.go b/gst/gst.go
index 521d42f..3697440 100644
--- a/gst/gst.go
+++ b/gst/gst.go
@@ -33,7 +33,7 @@ var pipelinesLock sync.Mutex
 
 // CreatePipeline creates a GStreamer Pipeline
 func CreatePipeline(containerPath string, audioTrack, videoTrack *webrtc.TrackLocalStaticSample) *Pipeline {
-       pipelineStr := fmt.Sprintf("filesrc location=\"%s\" ! decodebin name=demux ! queue ! x264enc bframes=0 speed-preset=veryfast key-int-max=60 ! video/x-h264,stream-format=byte-stream ! appsink name=video demux. ! queue ! audioconvert ! audioresample ! opusenc ! appsink name=audio", containerPath)
+       pipelineStr := fmt.Sprintf("filesrc location=\"%s\" ! decodebin name=demux ! queue ! x264enc bframes=0 speed-preset=veryfast key-int-max=60 ! video/x-h264,stream-format=byte-stream ! netsim drop-probability=0.5 ! appsink name=video demux. ! queue ! audioconvert ! audioresample ! opusenc ! appsink name=audio", containerPath)
 
        pipelineStrUnsafe := C.CString(pipelineStr)
        defer C.free(unsafe.Pointer(pipelineStrUnsafe))
```

これでrtwatchを再実行してブラウザから映像を視聴すると、明らかに映像の品質が悪く動きもカクカクしています。

[![Image from Gyazo](https://i.gyazo.com/da209e7e012ec12ba6e9bfa83aa512e8.gif)](https://gyazo.com/da209e7e012ec12ba6e9bfa83aa512e8)

Go側の出力を見てみると、Picture Loss Indication (PLI)が大量発生していました。[好奇心旺盛な人のためのWebRTC - メディア・コミュニケーション](https://webrtcforthecurious.com/ja/docs/06-media-communication/#rtcp)によると、PLIは次のような説明でした。

> これらのメッセージは、送信者にフルキーフレームを要求します。 PLIは、デコーダにパーシャルフレームが到着し、デコードできない場合に使用します。 これは、パケットロスが多い場合や、デコーダがクラッシュした場合などに起こります。

「パケットロスが多い場合」とあるので、狙ったとおりの結果が出たことになります！

```
$ go run main.go -container-path=bbb.mp4
Video file 'bbb.mp4' is now available on ':8080', have fun! 
2021/05/16 18:23:35 RTCP packet received: *rtcp.PictureLossIndication
2021/05/16 18:23:35 RTCP packet received: *rtcp.PictureLossIndication
2021/05/16 18:23:36 RTCP packet received: *rtcp.PictureLossIndication
2021/05/16 18:23:36 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 18:23:36 RTCP packet received: *rtcp.PictureLossIndication
2021/05/16 18:23:36 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 18:23:36 Estimated Bitrate: 612260
2021/05/16 18:23:36 RTCP packet received: *rtcp.PictureLossIndication
2021/05/16 18:23:36 Estimated Bitrate: 612260
2021/05/16 18:23:36 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 18:23:36 Estimated Bitrate: 546252
2021/05/16 18:23:36 RTCP packet received: *rtcp.PictureLossIndication
2021/05/16 18:23:36 Estimated Bitrate: 546252
2021/05/16 18:23:36 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 18:23:36 Estimated Bitrate: 442774
2021/05/16 18:23:36 RTCP packet received: *rtcp.PictureLossIndication
2021/05/16 18:23:36 Estimated Bitrate: 442774
2021/05/16 18:23:37 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 18:23:37 Estimated Bitrate: 416826
2021/05/16 18:23:37 RTCP packet received: *rtcp.PictureLossIndication
2021/05/16 18:23:37 Estimated Bitrate: 416826
2021/05/16 18:23:37 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 18:23:37 Estimated Bitrate: 416826
2021/05/16 18:23:37 RTCP packet received: *rtcp.PictureLossIndication
2021/05/16 18:23:37 Estimated Bitrate: 416826
2021/05/16 18:23:37 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 18:23:37 Estimated Bitrate: 416826
2021/05/16 18:23:37 RTCP packet received: *rtcp.PictureLossIndication
2021/05/16 18:23:37 Estimated Bitrate: 416826
2021/05/16 18:23:37 RTCP packet received: *rtcp.ReceiverReport
2021/05/16 18:23:37 Estimated Bitrate: 416826
2021/05/16 18:23:37 RTCP packet received: *rtcp.PictureLossIndication
```

## Pion Interceptor

ここまでで、pion/webrtcでRTCP Feedbackを受け取って独自の制御を追加する方法がわかりました。
ここからの発展として[pion/interceptor](https://github.com/pion/interceptor)というモジュールが公開されています。

InterceptorはすべてのRTP/RTCPパケットの受信前後に任意の処理を挟むことができる仕組みです。
これを使うとRTCP Feedbackの内容を見て何らかの制御を行う部分を分離できます。
今回のrtwatchの例のようにアプリ側に直接goroutineを書くよりもコードの見通しがよくなりそうで良いですね！

たとえば、**NACK** というパケット損失を通知するRTCP Feedbackに対して失ったパケットを再送する NACK Interceptorの実装がすでに公開されているので参考になります。

https://github.com/pion/interceptor/blob/master/pkg/nack/responder_interceptor.go

