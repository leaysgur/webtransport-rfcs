> Read [original](https://tools.ietf.org/html/draft-vvv-webtransport-http3-01) / [markdown](../markdown/draft-vvv-webtransport-http3-01.md)

---

# WebTransport over HTTP/3

## Abstract

- HTTP/3の上でWebTransportを実現するHttp3Transportについて

## 1. Introduction

- HTTP/3を使ってWebTransportを実装することもできる
- HTTP/3
  - 複数のHTTPを単一のQUICコネクションで捌ける
  - ここで対象とするのはHTTPではないデータ
- 複数のHttp3Transportが他のHTTPのトラフィックと一緒に、HTTP/3コネクションを流れる

### 1.1. Terminology

- 用語の解説

## 2. Protocol Overview

- Http3Transportは、パス（URI）ごとに認証され識別される
- Http3Transportを使うためには、`http3_transport_support`パラメータをネゴシエーションする
- Http3Transportセッションは、拡張した`CONNECT`リクエストを送る
  - これをサーバーが許可すれば、Http3Transportがつながる
  - そのときにHTTP/3コネクションを識別する`SID`が使われる
- セッション確立後、ピアはいくつかの方法でデータの送受信ができる
  - クライアント: ストリームの所有権をHttp3Transportに渡す特別な無限長のHTTP/3フレームを使用した双方向ストリーム
  - サーバー: 双方向ストリーム
    - HTTP/3はサーバー起因の双方向ストリームを定義してないのでできる
  - クライアント+サーバー: 特殊なストリームタイプを使用した単方向ストリーム
  - データグラムは、QUIC DATAGRAMフレームを使って送る
- Http3Transportは、対応するCONNECTストリームが閉じられると終了する

## 3. Session IDs

- 単一のHTTP/3コネクション内で、複数のHttp3Transportを作るために、それぞれユニークなIDが必要
  - それがSID
- 62bitの数字で、セッションが閉じられても再利用されることはない
- クライアントが0からはじめてSIDをインクリメントしていく
- SIDはアプリケーションに公開されるべきではない

## 4. Session Establishment

- 本文なし

### 4.1. Establishing a Transport-Capable HTTP/3 Connection

- Http3Transportのサポートを示唆するために、`http3_transport_support`パラメータを送る必要がある
  - クライアントもサーバーも
  - 値は空
- これをネゴシエーションすることなしに、Http3Transportの機能を使うことはできない
  - ネゴシエーションは、HTTP/3ではなくQUICのパラメータで行われる
- `http3_transport_support`がネゴシエーションされると、QUIC DATAGRAM拡張もネゴシエーションされる
  - `initial_max_bidi_streams`パラメータは`0`以上の値で、HTTP/3の設定を上書きする

### 4.2. Extended CONNECT in HTTP/3

- `SETTINGS_ENABLE_CONNECT_PROTOCOL`パラメータによって、`CONNECT`メソッドを拡張できるようになっている
  - RFC8441
- しかしこれはHTTP/2用
- この仕様で新しいパラメータを追加したりはしない
- 代わりに`http3_transport_support`が、`CONNECT`の拡張を示すために使われる

### 4.3. Creating a New Session

- Http3TransportはHTTP/3上で展開されるので、`https`のURIスキームを使う
- まずはHTTPのCONNECTリクエストをクライアントが送る
  - `:protocol`は`webtransport`
  - `:scheme`は`https`
  - `:authority`と`:path`も必須
  - 新しいSIDを選んで`:sessionid`に載せる
  - Originヘッダも必須
- それを受け取ったサーバーは、WebTransportのサーバーがあるか確認する
  - ないなら`404`
  - あるなら`200`
- `200`を返す前に、`SID`のコンフリクトを確認する必要がある
  - Originの確認も必須
- セッションが確立されるタイミング
  - クライアント: `200`のレスポンスを受け取ったら
  - サーバー: `200`を返したら
- セッションが確立されるまで、一切のデータは送受信しない
  - Http3Transportは、いわゆる0-RTTをサポートしない

### 4.4. Limiting the Number of Simultaneous Sessions

- HTTPによってセッション確立が成されるので、フロー制御もHTTPと同様
- この仕様では特別なフロー制御を提案したり、セッション確立用のリクエストを個別に扱ったりもしない
- よしなにフロー制御すればよい
  - `HTTP_REQUEST_REJECTED`というエラーコードが用意されてるので使える
  - `429`を返すことでも表現できる

## 5. WebTransport Features

- Http3Transportは、WebTransportとしての特性をすべてサポートする
- `SID`は異なるHttp3Transportのセッションを識別するために使われる

### 5.1. Unidirectional streams

- どちらのエンドポイントからでも、単方向のストリームを作れる
- HTTP/3の`Control Stream`のtypeは`0x54`
- `SID`につづいて`Stream Body`がくる

### 5.2. Client-Initiated Bidirectional Streams

- `WEBTRANSPORT_STREAM`: `0x41`フレームによって双方向ストリームを作れる
- こちらも`SID`につづいて`Stream Body`がくる

### 5.3. Server-Initiated Bidirectional Streams

- HTTP/3としては、サーバー起因の双方向ストリームを規定していない
  - `http3_transport_support`で判断
- `Stream Type`はなく、`SID`と`Stream Body`から成る

### 5.4. Datagrams

- DATAGRAMフレームによってデータグラムを送信できる
  - `http3_transport_support`でネゴシエーション
  - draft-schinazi-quic-h3-datagram-02
- `Stream Type`はなく、`SID`と`Datagram Body`から成る
  - `Datagram Body`は`Flow Identifier`からはじまる
- QUICではDATAGRAMフレームを1パケットに収めて送る
  - なのでMTUをアプリケーションが知る必要がある

## 6. Session Termination

- Http3Transportは、どちらかのエンドポイントがストリームを閉じたら終了する

## 7. Transport Properties

- WebTransportとしての特性は4つあったが、それのサポート状況
  - Stream independence: サポート
  - Partial reliability: サポート
  - Pooling support: サポート
  - Connection mobility: 実装に依存する

## 8. Security Considerations

- WebTransportとして、QuicTransportのセキュリティ要件をすべて満たす
  - HTTP/3はQUICベースなので、QUIC本体の要件もある
- Http3Transportでは、明示的にQUICのパラメーターを使用する
  - 潜在的なプロトコル混乱攻撃を回避できる
  - Originヘッダも必須で、信頼できない接続を拒否できる
- HTTP/3と同様に、単一のコネクションで複数のオリジンに対する接続をプールする
  - なので輻輳制御やフリー制御のコンテキストを共有することになる
  - リソース枯渇攻撃のおそれがある

## 9. IANA Considerations

- 本文なし

### 9.1. Upgrade Token Registration

- HTTPの`Upgrade Token Registry`に、`webtransport`を追加

### 9.2. QUIC Transport Parameter Registration

- `http3_transport_support`パラメータの追加

### 9.3. Frame Type Registration

- HTTP/3の`Frame Type`に、`WEBTRANSPORT_STREAM`: `0x54`を追加

### 9.4. Stream Type Registration

- HTTP/3の`Stream Type`に、`WebTransport stream`: `0x41`を追加
