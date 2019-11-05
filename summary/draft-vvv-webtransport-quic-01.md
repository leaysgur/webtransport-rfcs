> Read [original](https://tools.ietf.org/html/draft-vvv-webtransport-quic-01) / [markdown](../markdown/draft-vvv-webtransport-quic-01.md)

---

# WebTransport over QUIC

## Abstract

- QUICの上でWebTransportを実現するQuicTransportについて

## 1. Introduction

- QUICを使ってWebTransportを実装することもできる
- QUIC
  - UDPベースで多重化されるセキュアなトランスポート
  - HTTP/3を支える技術でもある
  - ブラウザでもサーバーでも動作するので、WebTransportにもぴったり
- そのためのQuicTransportについて解説
  - QUICの実装があれば、ちょっとの手間で実装できるはず

### 1.1. Terminology

- 用語の解説

## 2. Protocol Overview

- QuicTransportは、単一のQUICコネクションと対応する
  - つまりコネクション同士、QuicTransport同士は疎になる
  - プールされることもないし、輻輳制御も干渉しない
- QuicTransportはQUICの拡張のような立ち位置なので、高次の機能は提供しない
  - プーリング
  - セッション確立時のメタデータのやりとり
  - リダイレクト
  - など、QUICが本来提供しない機能
  - そういうことが気になる場合は、HTTP/3でWebTransportを使うとよいかもしれない
- 接続開始にあたり、特定のアドレスに向けてQUICのコネクションを確立する
  - ALPNを使ってエンドポイントを検証
  - パスやオリジンなどの情報もあわせてサーバーに送る
- ストリームはQUICのストリームを使う
- データグラムはデータグラム拡張を使う
  - draft-pauly-quic-datagram-05

## 3. Connection Establishment

- QuicTransportのセッションを確立するために、QUICのコネクション確立が必要
- クライアント視点では、サーバーからTLSの`Finished`メッセージが届き、`Client Indication`を送信したら確立
- サーバー視点では、クライアントの`Client Indication`を処理したら確立

### 3.1. Identifying as QuicTransport

- TLSのALPNを使う
  - RFC7301
- `wq-vvv-01`という値

### 3.2. Client Indication

- クライアントのオリジンを検証するために送る特別なメッセージ
  - Stream 2で送る
  - クライアントが最初に単方向ストリームを初期化するときに
- 以下のフィールドをもつ
  - `Key`: 16bit
  - `Length`: 16bit（`Value`の長さ）
  - `Value`
- Stream 2の`FIN`で完了を表す
  - それまではサーバーはアプリケーションデータを処理してはいけない
- `Client Indication`は65535バイトを超えてはならない
- サーバーは`initial_max_streams_uni`パラメータを少なくとも`1`にする必要がある
  - これが`0`なら、クライアントはコネクションを閉じなければならない
- サーバーは解釈できないフィールドを無視する
- すべてのフィールドはユニークで、そうでない場合はコネクションを閉じるかもしれない

#### 3.2.1. Origin Field

- オリジンポリシーを満たすために必要
- `Key`: `0x0000`
- `Value`: オリジン

#### 3.2.2. Path Field

- オリジン以降のパス部分は、同じホスト・ポートで多重化するために必要
- `Key`: `0x0001`
- `Value`: クエリパラメータがあれば`?`でつなぎ、ないなら`/`になる
- ホスト名は、TLS拡張の`server_name`で使われる
  - RFC6066

### 3.3. 0-RTT

- QUICの0-RTTはQuicTransportでも利用できる
  - TLSが確立される前にデータを送ることができる
  - レイテンシを下げられるが、リプレイ攻撃に弱い
- いつでも利用できるわけでなく、クライアントに要求されたときだけ利用できる
- QUICとTLS 1.3でそうであるように、サポートは任意

## 4. Streams

- QuicTransportのストリームは、QUICのストリームが使われる
  - 単方向でも双方向でも
- Read/Write/Closeなどの操作もそのままマッピングされる
- QUICのストリームIDもそのまま公開される

## 5. Datagrams

- QUICのDATAGRAMフレームを使ってデータグラムを送信する
- アプリケーションから受け取ったデータグラムはそのまま送信される

## 6. QuicTransport URI Scheme

- QuicTransportは専用のURIスキームがある
- `quic-transport://`
  - そのあとにhost, path, query, fragmentなど
- 現時点では`#`以下のfragmentは用途がないので無視する

## 7. Transport Properties

- WebTransportとしての特性は4つあったが、それのサポート状況
  - Stream independence: サポート
  - Partial reliability: サポート
  - Pooling support: サポートしない
  - Connection mobility: 実装に依存する

## 8. Security Considerations

- QuicTransportのセキュリティは、QUICの、それを支えるTLSのセキュリティ要件に準ずる
- QUICはクライアント・サーバーのプロトコルで、ハンドシェイクをしていないクライアントはデータを送信できない
  - サーバー側から接続を閉じることもできる
  - DoS耐性があるといえる
- QUICは輻輳制御の仕組みがあるので、帯域があふれることもないはず
  - draft-ietf-quic-recovery
- キープアライブの仕組みがあり、サーバーが`ACK`フレームを送らないと、クライアントの接続もタイムアウトするようになっている
- ALPNのチェックが必須であり、そうしないと、TLSのハンドシェイクが失敗する
- 正しいALPNがサーバーで受け取られるまで、クライアントにはどのコネクションでエラーになったかわからない
- クライアントもサーバーも、コネクションの数を制限してもよい

## 9. IANA Considerations

- 本文なし

### 9.1. ALPN Value Registration

- ALPNには`wq-vvv-01`を使う
  - `0x77 0x71 0x2d 0x76 0x76 0x76 0x2d 0x30 0x31`

### 9.2. Client Indication Fields Registry

- `Client Indication`のフィールド

### 9.3. URI Scheme Registration

- URIスキームに`quic-transport`を追加
