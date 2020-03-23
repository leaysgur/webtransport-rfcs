> Read [original](https://tools.ietf.org/html/draft-ietf-quic-transport-25) / [markdown](../markdown/draft-ietf-quic-transport-25.md)

---

# QUIC: A UDP-Based Multiplexed and Secure Transport

## Abstract

- QUICトランスポートのコアについて
- ロス推定と帯域制限、TLSによる鍵交換についてもあわせて読むこと
  - draft-ietf-quic-tls
  - draft-ietf-quic-recovery

## 1.  Introduction

- QUICが提供するもの
  - ストリーム多重化
  - ストリーム・コネクションレベルでのフロー制御
  - 低レイテンシーな接続確立
  - NATの再接続耐性
  - 認証と暗号化されたヘッダ、ペイロード
- UDPの上に構築されるので、旧来のOSなどでも動く

### 1.1.  Document Structure

- この文書の構成
  - Section 2 ~ 4: QUICストリーム
  - Section 5 ~ 11: QUICコネクション
  - Section 12 ~ 14: QUICパケット、フレーム
  - Section 15 ~ 20: エンコーディングの詳細

### 1.2.  Terms and Definitions

- いつもの
- QUIC: 略語ではなく名称
- エンドポイント: QUICコネクションに参加する主体
  - ClientとServerの2つのタイプがある
  - QUICパケットをやりとりする
- Client: QUICコネクションを開始するエンドポイント
- Server: QUICコネクションを受け付けるエンドポイント
- Address: IPバージョン、アドレス、UDPプロトコル、UDPポート番号から成るタプル
- コネクション ID: 各エンドポイントでQUICコネクションを識別するためのID
- ストリーム: QUICコネクション内で流れる単方向または双方向のバイト列
  - 1コネクション内で、Nストリームを扱う

### 1.3.  Notational Conventions

- パケットやフレームをあらわす図の読み方

## 2.  Streams

- ストリームは単方向、双方向どちらもある
- 無限長のメッセージを抽象化したものとも言える
  - データを送信することで作られる
  - 1つの`STREAM`フレームで開始から終了までできる
  - QUICコネクションと同じライフタイムを生き続けることもある
- どちらかのエンドポイントで作られる
- 他のストリームとは独立しており、順序も保証されない
- ストリームの数、それぞれのデータ量などは制限することができる

### 2.1.  Stream Types and Identifiers

- Client/Serverどちらかが初期化する
- 方向は単方向か双方向
- ストリームはIDをもつ
  - 62bit整数
  - コネクション内のすべてのストリーム間で一意
  - 使い回してはいけない
- ストリームIDのLSBの2bitを見れば、初期化したエンドポイントと方向性がわかる
  - `0x0`: Client発の双方向ストリーム
  - `0x1`: Server発の双方向ストリーム
  - `0x2`: Client発の単方向ストリーム
  - `0x3`: Server発の単方向ストリーム

### 2.2.  Sending and Receiving Data

- `STREAM`フレームでデータを送信する
  - ストリームIDとオフセットというフィールドで順序を保証する
- エンドポイントは順不同で受け取ったデータを保持し、並び替える
  - フロー制御制限が決められており、その限界まで貯める
- QUIC自体は、順不同に送信することを許可していない
  - ただし実装はそれを選択できる
- 同じオフセットでデータを複数回受信できる
  - 既に受信していたものは破棄できる
  - その場合、同様のデータを送信する必要がある
  - 同じオフセットで異なるデータを受信したら、`PROTOCOL_VIOLATION`エラーとして処理する
- エンドポイントは対向のフロー制御制限を確認せずに送信してはならない

### 2.3.  Stream Prioritization

- ストリームごとの優先度
- QUIC自体はそういった情報を交換する仕組みはもたない
  - アプリケーション側が勝手にやる
- QUICの実装は、アプリケーションが優先度を指定できるようにするべき
- 同様に、相対的な優先度も示す必要がある

### 2.4.  Required Operations on Streams

- QUICストリームが満たすべき機能について
- データの書き込み
  - フロー制御が成されたことを理解する
- ストリームの終了
  - `STREAM`フレームに`FIN`bitをセットする
- ストリームのリセット
  - `RESET_STREAM`フレーム
- データの読み出し
- 読み出しの中止と、終了の要求
  - `STOP_SENDING`フレーム
- ストリームの状態の通知
  - ストリームの開始、終了
  - データが読み書きできるようになったタイミングなども

## 3.  Stream States

- 送信と受信で、2つのステートマシンが必要になる
- 単方向ストリームなら1つだが、双方向ストリームなら2つの組み合わせになる
  - 双方向ストリームの場合、両方のエンドポイントで2つずつの状態ができるので、より複雑な手順になる

### 3.1.  Sending Stream States

```
       o
       | Create Stream (Sending)
       | Peer Creates Bidirectional Stream
       v
   +-------+
   | Ready | Send RESET_STREAM
   |       |-----------------------.
   +-------+                       |
       |                           |
       | Send STREAM /             |
       |      STREAM_DATA_BLOCKED  |
       |                           |
       | Peer Creates              |
       |      Bidirectional Stream |
       v                           |
   +-------+                       |
   | Send  | Send RESET_STREAM     |
   |       |---------------------->|
   +-------+                       |
       |                           |
       | Send STREAM + FIN         |
       v                           v
   +-------+                   +-------+
   | Data  | Send RESET_STREAM | Reset |
   | Sent  |------------------>| Sent  |
   +-------+                   +-------+
       |                           |
       | Recv All ACKs             | Recv ACK
       v                           v
   +-------+                   +-------+
   | Data  |                   | Reset |
   | Recvd |                   | Recvd |
   +-------+                   +-------+
```

- まずアプリケーションによって送信用のストリームが作られる
  - その時点では`Ready`
  - 送信するデータがバッファに貯められているかも
- 最初の`STREAM`フレームか、`STERAM_DATA_BLOCKED`フレームを送信すると、`Send`に状態遷移する
  - この時点までストリームIDの割当を行わない実装もありえる
- `Send`状態では、フロー制御制限を尊重しつつデータを`STREAM`フレームで送信する
  - その際、`MAX_STREAM_DATA`フレームを生成する
  - `STERAM_DATA_BLOCKED`は、フロー制御によって送信できなかったときに生成される
- `FIN`bitつきで`STREAM`フレームを送ると、`Data Sent`状態に遷移する
  - ここからはもう再送しかできない
  - フロー制御制限を気にしなくていいし、`STREAM_DATA_BLOCKED`も生成しない
  - `MAX_STREAM_DATA`は受信するかもしれないが、もう無視できる
- すべてのデータ送信が確認されたら、最終的な`Data Recvd`状態に遷移する
- `Ready`, `Send`, `Data Sent`状態のときに、送信中断することもできる
  - エンドポイントから`STOP_SENDING`フレームが届いた場合も同様
  - そのときは、`RESET_STREAM`フレームを送って、`Reset Sent`状態になる
- `Ready`から`Reset Sent`に遷移することも可能
- `RESET_STREAM`フレームが確認されたら、`Reset Recvd`状態になる

### 3.2.  Receiving Stream States

```
       o
       | Recv STREAM / STREAM_DATA_BLOCKED / RESET_STREAM
       | Create Bidirectional Stream (Sending)
       | Recv MAX_STREAM_DATA / STOP_SENDING (Bidirectional)
       | Create Higher-Numbered Stream
       v
   +-------+
   | Recv  | Recv RESET_STREAM
   |       |-----------------------.
   +-------+                       |
       |                           |
       | Recv STREAM + FIN         |
       v                           |
   +-------+                       |
   | Size  | Recv RESET_STREAM     |
   | Known |---------------------->|
   +-------+                       |
       |                           |
       | Recv All Data             |
       v                           v
   +-------+ Recv RESET_STREAM +-------+
   | Data  |--- (optional) --->| Reset |
   | Recvd |  Recv All Data    | Recvd |
   +-------+<-- (optional) ----+-------+
       |                           |
       | App Read All Data         | App Read RST
       v                           v
   +-------+                   +-------+
   | Data  |                   | Reset |
   | Read  |                   | Read  |
   +-------+                   +-------+
```

- 受信側は明示的なきっかけがアプリケーションから得られないので、`Ready`のような状態はない
  - 代わりに、送信側からはわからない「アプリケーションへのデータの配送」を司る状態遷移をする
- 送信側から`STREAM`, `STREAM_DATA_BLOCKED`, `RESET_STREAM`フレームを受け取るところからはじまる
  - 双方向ストリームの場合は、送信用の`MAX_STREAM_DATA`, `STOP_SENDING`フレームでも
- そうして作られた最初の状態は`Recv`
  - 双方向ストリームの場合は、対応する送信側を用意して`Ready`になった時点で`Recv`になる
- `MAX_STREAM_DATA`, `STOP_SENDING`を受け取ることで双方向ストリームを作成した場合
  - `MAX_STREAM_DATA`フレームは、フロー制御のクレジットを発行したことを意味する
  - `STOP_SENDING`フレームは、データを受け取る意思がないことを意味する
- 必ずストリームIDが小さい順でストリームを作成することで、両エンドポイントで作成順序が一貫する
- `Recv`状態のときに、`STREAM`と`STREAM_DATA_BLOCKED`フレームを受信できる
- 受信したデータはバッファに貯められ並び替えられてからアプリケーションに渡される
- アプリケーション側でデータが消費されたら、`MAX_STREAM_DATA`フレームを送ることで新たなデータを送信させる
- `FIN`bitつきの`STREAM`フレームを受信したら、ストリームの最終的なサイズがわかる
  - そうすると`Size Known`状態に遷移する
  - `MAX_STREAM_DATA`フレームを送る必要がなくなって、再送されたデータのみ受信する
- すべてのデータが受信されると、`Data Recvd`状態に遷移する
  - `Size Known`状態に遷移すると同時に遷移することもある
  - 以降、`STREAM`, `STREAM_DATA_BLOCKED`フレームは無視できる
- アプリケーション側への配送も確認できたら、`Data Recvd`から最終的な`Data Read`状態に遷移する
- `Recv`, `Size Known`状態で`RESET_STREAM`フレームを受け取ると、`Reset Recvd`状態に遷移する
- データは`RESET_STREAM`フレームと前後して受信されるかもしれない
  - その場合にどうするかは実装に委ねられる
- `RESET_STREAM`を受信したことは、送信側が正しいデータの配送を保証できなくなったとき
  - なので受信側も、アプリケーションがデータを消費する前に破棄してしまってもよい
- タイミングによって、正しくデータが受信できた場合は、その`RESET_STREAM`を無視するかもしれない
  - その場合はもちろん`Data Recvd`状態に留まる
- 正しく`RESET_STREAM`フレームを処理するならば、最終的な`Reset Read`状態に遷移する

### 3.3.  Permitted Frame Types

- 送信側が使えるフレームは3種類だけ
  - `STREAM`, `STREAM_DATA_BLOCKED`, `RESET_STREAM`
- 終着状態である`Data Recvd`と`Reset Recvd`状態では、いかなるフレームも送信してはいけない
- 加えて、`Reset Sent`状態においては、`STREAM`, `STREAM_DATA_BLOCKED`を送信してはいけない
- 受信側は、`MAX_STREAM_DATA`と`STOP_SENDING`を送信する
- `MAX_STREAM_DATA`は`Recv`状態のときだけ送信できる
- `STOP_SENDING`フレームは、いかなる状態でも送信できる
  - ただし`RESET_STREAM`を受け取っていない、`Reset Recvd`, `Reset Read`状態のときを除く
- どちら側も、パケットの遅延などによっては例外的にフレームを受け取る可能性はある

### 3.4.  Bidirectional Stream States

- 双方向ストリームの場合は、送信と受信で2つの状態の組み合わせになる
- どちらも正常な状態の場合に`open`、どちらかが終着状態になったら`closed`と表すことができる
- HTTP/2で定義される状態との対応を取ることもできる
  - 上の2つに加えて、3つの状態がある
  - `idle`: 送信か受信どちらかが作られていない、送信側が`Ready`で受信側が`Recv`
  - `half-closed(remote)`: 受信側が終着状態
  - `half-closed(local)`: 送信側が終着状態
- このマッピングはあくまで一例である

### 3.5.  Solicited State Transitions

- アプリケーションがデータ受信を中断する場合について
- ストリームが`Recv`か`Size Known`状態なら、`STOP_SENDING`フレームを送って対向に知らせる
  - その間にも`STREAM`フレームは届くかもしれない
  - それらはフロー制御の観点からは引き続き数えられる
- `STOP_SENDING`フレームは、受信側に`RESET_STREAM`フレームの送信を要求する
  - `Ready`か`Send`状態なら、`RESET_STREAM`フレームを送信しなければならない
  - `Data Sent`状態なら、再送する代わりに`RESET_STREAM`フレームを送るべき
- エンドポイントは`STOP_SENDING`フレームからエラーコードを`RESET_STREAM`フレームへコピーするかもしれない
  - アプリケーションのエラーコードを使うかもしれない
- `STOP_SENDING`を送ってきたエンドポイントは、その後に受け取った`RESET_STREAM`のエラーコードを無視するかもしれない
- 既に`Data Sent`状態で`STOP_SENDING`を受け取ったら
  - 再送をやめる場合でも`RESET_STREAM`を先に送る必要がある
- `STOP_SENDING`フレームはリセットされていないエンドポイントへ送るべきもの
  - `Recv`か`Size Known`状態のときに有用なフレーム
- `STOP_SENDING`フレームを含むパケットがロストした場合は、それを再送したいかもしれない
  - そこで`RESET_STREAM`を受信した場合は、`Recv`, `Size Known`状態ではなくなる
  - そうなると`STOP_SENDING`を送る必要はなくなる
- 双方向ストリームの両端を終了したい場合
  - `RESET_STREAM`を送信し、`STOP_SENDING`を送信することでそれを促すことができる

## 4.  Flow Control

- QUICストリームは、送受信の両側でフロー制御される
- 受信側は送信側が送信できるデータ量をコントロールできるようになっている
- 同じように、QUICストリームの数自体も制限することができる
- `CRYPTO`フレームは制御されない
  - そこはTLSが担当し、それ用のインターフェースを用意する

### 4.1.  Data Flow Control

- QUICでは、HTTP/2と同様のクレジットベースのフロー制御の仕組みを使う
  - 受信側が想定しているバイト数を伝える
- ストリームとコネクションの2つのレベルで制御される
  - 1つのストリームが、コネクション全体を専有しないように
  - 1つのコネクション内で送信されるすべての`STREAM`フレームの量が、受信側のバッファ容量を超えないように
- 受信側は、まずハンドシェイク時にすべてのストリームの初期クレジットを伝える
  - これくらいのバイト数なら受信できますという値
- その後は`MAX_STREAM_DATA`フレームを送ってクレジットを追加する
  - ストリームIDを指定して送られる
- コネクションレベルでは、`MAX_DATA`フレームを送信側に送ることでクレジットを追加する
  - すべてのストリームのオフセットを合算した値
- 1度大きな値にしたあと、それより小さい値のオフセットにすることはできない
  - 送信側も、フロー制御制限の値を大きくするもの以外は無視する
- 送信側がフロー制限に違反したら、`FLOW_CONTROL_ERROR`エラーでコネクションを閉じる必要がある
- 送信側でクレジットが尽きたら、`STREAM_DATA_BLOCKED`か`DATA_BLOCKED`フレームを送る
  - 一定時間待ってもクレジットが追加されない場合、タイムアウトしてコネクションを閉じるかも

### 4.2.  Flow Credit Increments

- いつ、どのように、クレジットを増やすか
- 頻繁に小さな値で変更することは望ましくない
  - オーバーヘッドになる
- ただしいきなり大きな値に変更することも懸念がある
  - それだけのバッファが必要になりリソースを消費する
- TCPの実装と同じで、RTTと受信効率から自動チューニングするとよい
  - フロー制御のために余分なパケットを送信しないようにするべき
  - まだ送りたいフレームがあるとか、ブロックされているときとか
- 送信側をブロックされている状態にしないことがベスト
  - 少なくとも2RTTで、`MAX_DATA`か`MAX_STREAM_DATA`フレームを送ってあげる必要がある
- 受信側も`STREAM_DATA_BLOCKED`や`DATA_BLOCKED`フレームを待ってはいけない
  - これを受けてからクレジットを追加しようとしない

### 4.3.  Handling Stream Cancellation

- エンドポイントはどちらも、最終的なクレジットの値に合意する必要がある
- `RESET_STREAM`フレームにより、以降そのストリームに送られてくるデータは無視される
  - これにオフセットが含まれていない場合は、各エンドポイントでバイト数に差がでるかもしれない
- なので`RESET_STREAM`フレームには、データの最終的なサイズが含まれている
  - 受信側はその値をフロー制御に利用する
- `RESET_STREAM`は一方向のストリームだけを終了する
  - 双方でストリームの場合、片側には影響しない

### 4.4.  Stream Final Size

- 最終的なサイズは、そのストリームで消費されたクレジットの総量
- 送信されてないならもちろん0
- リセットされたストリームなら`RESET_STREAM`フレームに示された値
- それ以外は、`FIN`bitがある`STREAM`フレームの長さ + そのオフセット
- エンドポイントは、`Size Known`か`Reset Recvd`状態になったときに最終的なサイズを知る
  - それ以上は送ってはいけないし、この値を超えるようなことをしてもいけない
- 最終的なサイズを変更しようとする`RESET_STREAM`や、`STREAM`フレームを受信した場合
  - `FINAL_SIZE_ERROR`エラーを返す
  - そのために、閉じられたストリームの最終的なサイズは保存しておく必要がある

### 4.5.  Controlling Concurrency

- エンドポイントは、受信用のストリームの数を制限する
  - `initial_max_streams_xxx * 4 + initial_stream_id_for_type`より小さいストリームIDのみ
  - 初期値はハンドシェイク時に設定され、その後は`MAX_STREAMS`フレームで通知される
  - 単方向ストリームと双方向ストリームは別々の制限
- `max_streams`パラメータや、`MAX_STREAMS`フレームで、`2^60`より大きな値を受け取った場合
  - `STREAM_LIMIT_ERROR`エラーでコネクションを閉じる
- 受け取ったフレームのストリームIDがこれを超えていた場合も、`STREAM_LIMIT_ERROR`エラーで閉じる
- `MAX_STREAMS`フレームも、増やすことしかできない
  - 小さくしようとしても無視される
- 制限によりストリームを作成できないエンドポイントは、`STREAMS_BLOCKED`フレームを送信する
  - ストリームのフロー制御と同じように、このフレームを待ってから増やすことはしてはいけない

## 5.  Connections

- QUICコネクションの確立とは
  - 暗号コンテキストとバージョンのネゴシエーション
  - トランスポートのハンドシェイク
- 一度それが確立すると、異なるIP・ポートになってもコネクションを継続できる
- そしてどちらかのエンドポイントによって閉じられる

### 5.1.  Connection ID

- コネクションは、一意に識別するIDをもつ
  - エンドポイントが選んでつける
- UDPやIPの層のアドレス変更をカバーするために必要
  - このIDを頼りにパケットを送信する
- もちろん重複してはならないし、同じものを使いまわしてもいけない
  - なにかしら類推できる情報も含まない
- QUICパケットのLongヘッダとShortヘッダに、SCIDとDCIDとして載せられる
- Shortヘッダには、DCIDのみ含まれる
  - IDのフォーマットや長さはエンドポイントが知っておく必要がある
  - ロードバランサーなどもそれを知っておく必要がある
- Version Negotiationパケットは、このIDをエコーする
- 長さ0のIDを使うこともできる
  - 送り先を指定する必要がない場合
  - ただ多重化もできないし、再利用もできないので、基本的には使わない
- エンドポイントは`NEW_CONNECTION_ID`フレームでIDを確認できる

#### 5.1.1.  Issuing Connection IDs

- コネクションIDは、重複を防ぐためにもシーケンス番号を含む
- ハンドシェイク時、最初のIDがLongヘッダのSCIDフィールドに載ってくる
- 初期値は`0`
  - `preferred_address`パラメータが使われる場合、初期値は`1`
- 新たなIDは`NEW_CONNECTION_ID`フレームで通知される
  - 1ずつインクリメントする
- InitialパケットやRetryパケットでは、サーバーで保持されていない限りランダムなIDになる
- エンドポイントが一度コネクションIDを払い出したら
  - そのID向けのパケットを受け入れなければならない
  - `RETIRE_CONNECTION_ID`フレームで無効化されるまで
- `active_connection_id_limit`パラメータ
  - 将来のために、受信したコネクションIDを保存しておく上限数
  - これをこえたら`CONNECTION_ID_LIMIT_ERROR`エラーで閉じる
- IDが枯渇するリスクを避けるため、払い出せるコネクションIDの数や頻度を制限するかも

#### 5.1.2.  Consuming and Retiring Connection IDs

- コネクションIDは接続中にいつでも変更できる
  - `RETIRE_CONNECTION_ID`フレームによってそのIDを使わないことを通知
  - `NEW_CONNECTION_ID`フレームで、その代わりになる新たなIDを通知
- コネクションIDは、特定のローカルアドレスに紐づく
  - アドレスを変更する場合、その移行前アドレスで使われていたIDをすべて使用中止する必要がある
- `NEW_CONNECTION_ID`フレーム
  - `Retire Prior To`フィールドにて新たなIDを指定する
  - それを受け取ったら、`RETIRE_CONNECTION_ID`フレームで返答する
  - そして新たなIDを使用開始する
- 送信した`NEW_CONNECTION_ID`フレームが確認されて3PTO（ProveTimeOut）経過すると、そのIDを破棄する
  - それまでは使用中止しようとしたIDに対するパケットは受け取る必要がある
  - それ以降は、Stateless Resetにつながる

### 5.2.  Matching Packets to Connections

- 飛んできたパケットをどう見分けるか
  - 特定のコネクションに紐づくもの
  - 受け取るのがサーバーなら、新規のコネクションかもしれない
- DCIDを含むなら、それは既存のコネクションのもの
  - DCIDの長さが0の場合は、先述の長さ0のルールに従う
- Stateless Resetのためにパケットを送ることもある
- 不正なパケットは捨てられる
  - 既存のコネクションのプロトコルバージョンと違うとか
  - 暗号化を解くのに失敗したりしたとか
- 暗号化される前のInitial, Retry, Version Negotiationパケットも同様
  - その場合はコネクションエラー

#### 5.2.1.  Client Packet Handling

- サーバーからクライアントへ送られるパケットには、DCIDが必ず含まれる
- ネットワークの遅延なので、暗号キーより先に届いてしまったパケットもあるかも
  - 実装によっては捨てられるかもしれないし、貯められるかもしれない
- サポートしてないバージョンのパケットは、必ず捨てられる

#### 5.2.2.  Server Packet Handling

- クライアントからサーバーに送られるパケット
  - サポートしてないバージョンならVersion Negotiation
- サポートしていないバージョンのパケットは、デコードしようとしてはいけない
  - 暗号キーがあって、復号化できる条件が揃っていたとしても
- 初期パケットに問題がないなら、ハンドシェイクを続行する
  - この時点で、クライアントが選んだバージョンで決定される
- サーバーがクライアントを受け入れられない場合
  - `CONNECTION_CLOSE`フレームをエラーコード`SERVER_BUSY`で返す
- 0-RTTパケットを受け取った場合
  - Initialパケットが追って届くことを期待して、バッファに貯めるかも
- サーバーが返答する前にクライアントがハンドシェイクパケットを送信してきたら無視する

### 5.3.  Life of a QUIC Connection

- QUICコネクションの一生
- そもそもは、アプリケーションが相互にデータを送り合うためのもの
- ハンドシェイクでは、TLSを使って共有シークレットと各種接続パラメータを決める
- クライアントは、0-RTTを使ってハンドシェイクが終わる前にデータを送信できる
  - その反面、リプレイ攻撃などの保護がない
- サーバーも、クライアントからのハンドシェイク完了メッセージを受け取る前にデータを送信できる
  - これらの機能はレイテンシを下げるためのオプション
- コネクションを終了させる方法はいくつかあり、後述する

### 5.4.  Required Operations on Connections

- この文書では、特定のAPIについて記述しないが、以下の機能を実装する必要がある
- クライアント
  - コネクションの開始
  - 0-RTTの有効化と、それがサーバーによって受け入れられたかを通知する
- サーバー
  - コネクションの受け入れとハンドシェイクの開始
  - Early Dataがサポートされるならその処理
- どちらも
  - 接続パラメータによるストリーム数の制限
  - フロー制御やストリームの数などリソースの利用料の管理
  - ハンドシェイクが終わったか、進行中かなどのステータスの通知
  - `PING`フレームなど、タイムアウトを防ぐための仕組み
  - 即座にコネクションを終了する

## 6.  Version Negotiation

- Version Negotiationで、クライアント・サーバーはバージョンを同意する
- サーバーはコネクション開始のパケットへの返答として、Version Negotiationパケットを返す
- クライアントが送信する最初のパケットのサイズが重要になってくる
  - TODO: どういうこと？

### 6.1.  Sending Version Negotiation Packets

- クライアントが指定したバージョンが、サーバーでサポートしてなかったとき
  - Version Negotiationパケットでレスポンスする
- このときに、サーバーがサポートするバージョンのリストを返す
  - Version Negotiationパケットに対して、Version Negotiationパケットを返してはいけない
- この間、サーバー側の状態は更新されない
  - クライアントもパケロスしたりすると、同じものを再送する
  - 何度も同じレスポンスが返ってくるはず
- サーバーによっては、レスポンスする回数を限定しているかも
  - その場合、0-RTTを利用する場合などはつながらないままになる

### 6.2.  Handling Version Negotiation Packets

- クライアントがVersion Negotiationパケットを受け取ったら
  - 現在おこなっている接続はやめる必要がある
- このバージョンを合意する仕組みは、将来のQUICバージョンで変更されるかもしれない
  - バージョンのダウングレード攻撃なども懸念される
  - まだ仕様としても検討の余地がある部分

#### 6.2.1.  Version Negotiation Between Draft Versions

- ドラフト実装としては、ダウングレード攻撃を対策しなくてもよい
- なので、サポートするバージョンがあるならそれでつなぎなおしてよい
  - ただし、DCIDはランダムに新規発行する必要がある
- この内容は、ドラフト実装でのみ使用すること

### 6.3.  Using Reserved Versions

- QUICのバージョンは将来的に増える
  - なのでクライアントは、Version Negotiationを正しくハンドルする必要がある
- バージョンには、将来のために予約されているものがある
  - `0x?a?a?a?a`
- これを使うことで、強制的にVersion Negotiationの挙動を確認できる
  - クライアントから送ることも、サーバーから返すこともできる

## 7.  Cryptographic and Transport Handshake

- QUICでは接続確立のレイテンシを下げるために、暗号と接続のハンドシェイクを一挙に行う
  - 暗号化は`CRYPTO`フレームでハンドシェイクする
  - QUICのバージョン: `0x00000001`ではTLSを使うが、将来のバージョンでは違うかも
- 暗号のハンドシェイクでやりとりする内容
  - 鍵交換
  - サーバーは常に認証されるが、クライアントは任意
  - 1-RTTの鍵はForwardSecurityあり
- ECN(Explicit Congestion Notification)のサポートも検証できる
- `CRYPTO`フレームは複数のパケット番号で送ることができ、0はじまりのシーケンス番号で順序保証する
- アプリケーションプロトコルについてもネゴシエーションする

### 7.1.  Example Handshake Flows

- QUICにおけるTLSの扱いの詳細は、別の文書を参照
  - draft-ietf-quic-tls
- 暗号のハンドシェイクの前に、アドレスのバリデーションが行われる
- ハンドシェイクは、InitialパケットとHandshakeパケットを使う
- 1-RTTのハンドシェイクのフロー図

```
Client                                                  Server

Initial[0]: CRYPTO[CH] ->

                                 Initial[0]: CRYPTO[SH] ACK[0]
                       Handshake[0]: CRYPTO[EE, CERT, CV, FIN]
                                 <- 1-RTT[0]: STREAM[1, "..."]

Initial[1]: ACK[0]
Handshake[0]: CRYPTO[FIN], ACK[0]
1-RTT[0]: STREAM[0, "..."], ACK[0] ->

                            1-RTT[1]: STREAM[3, "..."], ACK[0]
                                       <- Handshake[1]: ACK[0]
```

- 0-RTTのハンドシェイクのフロー図

```
Client                                                  Server

Initial[0]: CRYPTO[CH]
0-RTT[0]: STREAM[0, "..."] ->

                                 Initial[0]: CRYPTO[SH] ACK[0]
                                  Handshake[0] CRYPTO[EE, FIN]
                          <- 1-RTT[0]: STREAM[1, "..."] ACK[0]

Initial[1]: ACK[0]
Handshake[0]: CRYPTO[FIN], ACK[0]
1-RTT[1]: STREAM[0, "..."] ACK[0] ->

                            1-RTT[1]: STREAM[3, "..."], ACK[1]
                                       <- Handshake[1]: ACK[0]
```

### 7.2.  Negotiating Connection IDs

- コネクションIDのネゴシエーションについて
  - SCIDとDCID
- サーバーからInitialパケットやHandshakeパケットをまだ受け取っていないクライアントの場合
  - DCIDには、予測できないランダムな値をいれる（8バイト)
  - このDCIDは、Initialパケットの保護鍵のために使われる
- クライアントはSCIDとその長さを送る
- 0-RTTの場合、クライアントは最初のInitialパケットで、DCIDとSCIDに同じ値を使う
  - サーバーからレスポンスがあったら、そのSCID値を新たにDCIDとして後続のパケットで使用する
  - つまりクライアントは接続確立時に、DCIDの値を変更する
- クライアントは、このDCIDの変更以外はしてはならない
- コネクションIDは、コネクションマイグレーションなどで変更になる可能性がある

### 7.3.  Transport Parameters

- 接続パラメータも暗号のハンドシェイクに含まれる
  - ハンドシェイクが終わると、そのパラメータが利用できる
- 無効なパラメータは、`TRANSPORT_PARAMETER_ERROR`エラーとして処理する
  - 重複するパラメータも同様
- Retryパケットを送信する場合は、`original_connection_id`パラメータが必要

#### 7.3.1.  Values of Transport Parameters for 0-RTT

- 0-RTTとパラメータについて
- サーバーからのパラメータを保持し、後続の0-RTTパケットに適用する
  - 明示的に除外されるものもある
  - サポートしていないパラメータは保持しなくてよい
- クライアントは次のパラメータは再利用しない
  - `original_connection_id`
  - `preferred_address`
  - `stateless_reset_token`
  - `ack_delay_exponent`
  - `active_connection_id_limit`
- 0-RTTを受け入れたサーバーはつぎのパラメータに値を設定しない
  - `initial_max_data`
  - `initial_max_stream_data_bidi_local`
  - `initial_max_stream_data_bidi_remote`
  - `initial_max_stream_data_uni`
  - `initial_max_streams_bidi`
  - `initial_max_streams_uni`
- 特定のパラメータに0を設定するか省略すると、0-RTTが有効になるが、それらは使えなくなる
  - 暗黙の値がサポートできない場合は、ハンドシェイクを中止するか0−RTTを拒否する
- 0-RTTでは、必ず保持しているパラメータのみを使用する必要がある
  - 1-RTTで受信した値や、新たにサーバーから受け取った値を使用してはいけない
  - それが発覚すると、サーバーは`PROTOCOL_VIOLATION`エラーとするかも

#### 7.3.2.  New Transport Parameters

- 新しいパラメータは、新しいプロトコルでネゴシエーションに使われるかもしれない
- なのでサポートしてないパラメータは、必ず無視する
  - この挙動を確認するために、いくつかのパラメータが既に予約されている

### 7.4.  Cryptographic Message Buffering

- 実装は、`CRYPTO`フレームが順不同で届いたとしても対応する
  - `CRYPTO`フレームを送信する時点ではフロー制御がないから
  - 少なくとも4096バイトはバッファする必要がある（もっと多くてもよい）
- このバッファに収まらなかった場合は、`CRYPTO_BUFFER_EXCEEDED`エラーになる
  - その場合`CRYPTO`フレームは捨てられることになるので、それがわかるようにしないといけない

## 8.  Address Validation

- トラフィック増幅攻撃を避けるために、アドレスを検証する
  - 送信元になりすましてサーバーに接続し、レスポンスで攻撃させる手法がある
- 接続確立の際だけでなく、マイグレーション時にも検証する

### 8.1.  Address Validation During Connection Establishment

- アドレスは接続確立時に検証される
  - サーバーからInitialパケットを受け取ったときなど
- クライアントからHandshakeパケットが届いたら、正しく検証されたと考えられる
- アドレスを検証する前に、サーバーは受信したバイト数の3倍以上のサイズのパケットを送信してはいけない
  - 増幅攻撃の規模を制限できる
  - 正常に処理できたパケットのサイズのみを数える
- クライアントは、Initialパケットを送信する際、1200バイトのペイロードにする
  - 足りないならパディングする
- サーバーからのHandshakeパケットがパケロスすると
  - ハンドシェイクがデッドロック状態になる可能性がある
  - クライアントはタイムアウト時にパケットを送信する
  - draft-ietf-quic-recovery
- サーバーもハンドシェイク開始前にクライアントのアドレスを検証したいかも
  - Initialパケットにそのためのトークンが含まれておりそれを使える
- トークンは、Retryパケットか、`NEW_TOKEN`フレームから得られる
- アドレス検証前の送信制限について
  - クライアント: 輻輳コントローラーにのみ制限される
  - サーバー: 輻輳コントローラー + 上述の送信制限がある

#### 8.1.1.  Token Construction

- サーバーからクライアントに提供されるトークンの生成について
  - `Retry`パケットか`NEW_TOKEN`フレームでクライアントに送信する
  - 同じフィールドで届けられるが、扱い方が異なる

#### 8.1.2.  Address Validation using Retry Packets

- クライアントからInitialパケットを受け取ると、サーバーはトークンを含むRetryパケットを送ることができる
  - それでクライアントはアドレスを検証できる
- クライアントはInitialパケットをそのトークン付きで繰り返す
  - これでサーバーも正しいクライアントであることがわかる
- クライアントから無効なトークンが送られてきたら
  - パケットを破棄して、タイムアウトしハンドシェイクを失敗させる
  - `INVALID_TOKEN`エラーでコネクションを閉じる

#### 8.1.3.  Address Validation for Future Connections

- アドレス検証は特に0-RTTにおいて重要である
  - そのレスポンスでクライアントにデータを送ってしまうから
- サーバーは`NEW_TOKEN`フレームで、将来的な接続時に使える新たなトークンを配布できる
  - サーバーがRetryパケットで新たなトークンを返さない限り、このトークンを使う
  - Retryパケットにあるトークンは、保持しておいて将来的に使うことはできない
- このトークンは期限付きであるべき
  - もちろんサーバーがそれを発行して期限切れかチェックする
- `NEW_TOKEN`フレームにのるトークンは、全てのクライアント間でもユニークである必要がある
  - パケロスの再送は除く
- このトークンを使うと、サーバーがクライアントを識別できるため、使用は任意
- サーバーはRetryパケットで発行したトークンの期限を短くするとよい
  - DDoS攻撃に対する備えとして
- `NEW_TOKEN`フレームで発行したトークンの期限は長くてもよい
  - ただし、1度だけ使用できるようにすることが望ましい

#### 8.1.4.  Address Validation Token Integrity

- トークンは推測できてはならない
  - 十分に長いランダムな値
- サーバーは、完全性を保証し改ざんされていないことを確認する必要がある
- トークンに決められた形式はない
  - クライアントのIP、ポート、タイムスタンプなどの情報を含むこともできる

### 8.2.  Path Validation

- パス検証はコネクションのマイグレーション時に行われる
  - リモートとローカルのアドレスで疎通を確認する
  - アドレスとはIP、ポートの2つからなるタプル
- `PATH_CHALLENGE`フレームと`PATH_RESPONSE`フレームで疎通を確認する
- 存在を定期的に確認するために行ってもよい
- NAT越えの用途としては使わない
- 他のフレームと合わせて送られるかもしれない
  - PMTUを知るついでだったり、`PATH_RESPONSE`のついでに`PATH_CHALLENGE`したり
- パス検証時に新たなコネクションIDを使いたい場合もあるかも
  - そのときは、`NEW_CONNECTION_ID`フレームと`PATH_CHALLENGE`フレームを一緒のパケットで送る

### 8.3.  Initiating Path Validation

- パス検証では、ランダムなペイロードで`PATH_CHALLENGE`フレームを送る
  - フレームごとに予測できない値を使う
- パケロスに備えて複数の`PATH_CHALLENGE`フレームを送信するかも
  - ただし1つのパケットに複数の`PATH_CHALLENGE`フレームを含めてはいけない
  - Initialパケットよりも頻繁に送ってもいけない

### 8.4.  Path Validation Responses

- `PATH_CHALLENGE`フレームを受け取ったら、そのペイロードをエコーする形で`PATH_RESPONSE`フレームを送る
- レスポンスは1度だけしか返してはいけない
  - レスポンスが欲しいなら、`PATH_CHALLENGE`をもう一度送る

### 8.5.  Successful Path Validation

- `PATH_RESPONSE`フレームが返ってきたら、その新しいアドレスの確証が持てる
- 異なるローカルアドレスでレスポンスを受け取るかもしれない
  - 単に間違いの可能性もあるし、フォワードされたものかもしれない

### 8.6.  Failed Path Validation

- パス検証は、それを放棄したときにのみ失敗する
- エンドポイントはタイマーでパス検証を放棄する
  - PTO（ProveTimeOut）か
  - 初期タイムアウト値（`2 * kInitialRtt`）の3倍くらいの時間が推奨
    - draft-ietf-quic-recovery
  - `validation_timeout = max(3*PTO, 6*kInitialRtt)`
- パス検証が失敗しても、接続が失敗するというわけではない
  - 他のパスが有効かもしれないから
- 使用可能なパスが存在しない場合は、新しいパスを待つか、コネクションを閉じる
- 新しいパスへのマイグレーションがはじまると、古いパスでのパス検証が放棄されるかもしれない

## 9.  Connection Migration

- コネクションIDによって、IPアドレスとポートが変更になっても、接続を継続できる
- ただしハンドシェイクが完了するまでは、マイグレーションできない
- `disable_active_migration`パラメータ
  - マイグレーションの利用可否をあらわす
  - これがある場合、マイグレーションしてはいけない
- これがあるのに異なるアドレスからパケットが送られてきたとわかったら
  - StatelessResetせずにそれを捨てるか
  - パス検証してマイグレーションを許容するか
- アドレス変更は、故意ではない場合もある
  - NATの再バインディングなど
- アドレス変更を検知したら、即座にパス検証をする必要がある
- マイグレーションは、クライアントから開始する
  - 後述する一部のケースを除く
  - サーバーからそのようなパケットを受信したら、必ず捨てること

### 9.1.  Probing a New Path

- マイグレーション前に、新しいパスが到達できるのかパス検証を行う
- 検証に失敗したら
  - その新しいアドレスは使えないとわかる
  - 他にパスがあるなら、コネクションが終了するわけではない
- 新しいアドレスから、新しいコネクションIDを使用する
  - `NEW_CONNECTION_ID`フレーム
- プロービングフレーム
  - `PATH_CHALLENGE`、`PATH_RESPONSE`、 `NEW_CONNECTION_ID`、`PADDING`フレーム
  - これによって新たなパス、アドレスが有効であると証明できる
- プロービングフレームのみのパケットを、プロービングパケットという

### 9.2.  Initiating Connection Migration

- 新しいアドレスからノンプロービングパケットを送ることで、マイグレーションを開始できる
- 新しいパスでは現在の送信速度をサポートしない可能性があるので、輻輳コントローラーをリセットする
- 同様にECN(Explicit Congestion Notification)がない場合もあるので、それも検証する
  - RFC3168
- 新しいパスで送信したデータが確認されたら、新アドレスが有効だと証明される

### 9.3.  Responding to Connection Migration

- ノンプロービングフレームを新しいアドレスから受け取ったら、マイグレーションを開始したと考えられる
  - いくつかのパケットを返送して、パス検証をする
- ノンプロービングパケットがいくつか受け取った場合は、パケット番号が最大のもののみ対応する
  - そうしたら小さい番号のものは捨てる
  - NATの再バインディングの結果、これが起こる可能性がある
- 新しいクライアントのアドレスを確認したら、サーバーはアドレス検証する
- 特定のケースでは、これらの検証をスキップすることもある

#### 9.3.1.  Peer Address Spoofing

- 送信元アドレスを偽装することで、特定のエンドポイントを攻撃できる
- そのためにも、アドレス検証でアドレスの有効性を確認する必要がある
  - 確認が取れるまで、送信するデータ量を制限する必要がある
  - 輻輳ウィンドウの最小データ量しか送ってはいけない
  - draft-ietf-quic-recovery
- アドレス検証をスキップする場合は、この制限も必要ない

#### 9.3.2.  On-Path Address Spoofing

- パス内で元のパケットより先に偽装パケットを送信する攻撃
  - 元のパケットが重複してるとみなされて捨てられてしまう
- 新しいアドレスにマイグレーションしようとしてしまう
  - `PATH_CHALLENGE`フレームが検証できず失敗する
- 新しいアドレスの検証に失敗したら、最後に検証されたアドレスを使い続ける必要がある
  - それがない場合はコネクションを閉じる
  - 応答がなければStatelessResetするかも
- 別のパケット番号が大きいものを受信したら、またマイグレーションを開始する

#### 9.3.3.  Off-Path Packet Forwarding

- パス外からパケットのコピーを転送する攻撃
- コピーが先に到着したら、NATの再バインディングがあったと受け取られる
  - 本物が重複として捨てられてしまう
  - マイグレーションが行われてしまい、後続のパケットを覗かれてしまう
- 元のパスでノンプロービングパケットのパケット番号が増えると、元のパスに戻る
  - 攻撃が失敗したことになる
  - パケット交換をすることが、攻撃への緩和になる
- マイグレーションに際して、以前アクティブだったパスを`PATH_CHALLENGE`フレームで検証しておく必要がある
- 古いパスでパケットが受信されるなら、NATの再バインディングは起こってないとわかる

### 9.4.  Loss Detection and Congestion Control

- マイグレーション後は、移行前と同じような輻輳制御やRTT推定ができないかもしれない
- その確証がない限り、輻輳コントローラーやRTT推定はリセットする必要がある
  - ポート番号が変わっただけならほぼ同じだろう、など
  - ただ100％の確度ではないはずなので注意すること
- マイグレーション中には複数のパスが使われることがあるが、輻輳制御のコンテキストは1つで十分
  - draft-ietf-quic-recovery
- プロービングパケットの例外を送信することで、輻輳コントローラーが送信レートを下げないようにもできる
- `PATH_CHALLENGE`フレーム送信時にタイマーを設定し、`PATH_RESPONSE`フレーム受信時にキャンセルする
  - タイマーが先に動いたら、改めて`PATH_CHALLENGE`を送信し、タイマーの値を少し長くする

### 9.5.  Privacy Implications of Connection Migration

- 複数のネットワークで同じコネクションIDを使うと、ネットワーク監視者が活動を認識できてしまう
  - なので異なるローカルアドレスごとに、異なるコネクションIDを生成する
  - その他の情報と紐付かないように努める必要がある
  - DCIDはいつでも変更できる
- マイグレーションするか新しいパスをプロービングする際、新たなコネクションIDを使用する必要がある
  - ヘッダーは暗号化されているので、パケット番号をみて活動を認識することはできなくなる
- コネクションIDを変更せずに、パスが変更される可能性もある
  - NATの再バインディングなど
- なんらかの理由で、コネクションIDとUDPポート番号を変更したいかもしれない
  - ポート番号が変更になると、輻輳の状態がリセットされる可能性があるので、頻繁に行うべきではない
- コネクションIDが枯渇したエンドポイントは、新しいパスで送信されるすべてのパケットに、`NEW_CONNECTION_ID`フレームを含める

### 9.6.  Server's Preferred Address

- QUICではサーバー側がハンドシェイク完了後に、より望ましいアドレスに移行することを許している
  - まず代表アドレスで接続を受けて、その後で移行するといったケース
- 接続途中のマイグレーションは将来的に検討する
- クライアントは`preferred_address`パラメータなしに新しいアドレスのサーバーからパケットを受け取ったら、それを捨てるべき

#### 9.6.1.  Communicating a Preferred Address

- TLSハンドシェイクの際に、`preferred_address`パラメータを示す
  - IPv4とIPv6それぞれ出してくるかも
- ハンドシェイクが完了したら、クライアントはそのアドレスから選んでパス検証を行う
  - コネクションIDは、`preferred_address`パラメータによって渡される
- パス検証が成功したら、その後のパケットを新しいアドレスに向けて流す
  - 新しいコネクションIDで
- パス検証が失敗したら、既存のアドレスに対してパケットを流す（何も変わらない）

#### 9.6.2.  Responding to Connection Migration

- サーバーは、その優先アドレスにおいていつでもパケットを受け取る可能性がある
- そのパケットが`PATH_CHALLENGE`フレームを含んでいたら、`PATH_RESPONSE`フレームで応答する
- サーバーは、元のアドレスでノンプロービングフレームを送る必要がある
  - 優先アドレスでパケット番号が最大であるノンプロービングパケットを受け取って、そのパスを検証するまで
- 検証できたらノンプロービングパケットを優先アドレスから送ることができる
  - 元のアドレスからのパケットはもう捨ててもよいが、遅延したパケットだけは処理されるかもしれない

#### 9.6.3.  Interaction of Client Migration and Preferred Address

- サーバーの優先アドレスにマイグレーションする前に、クライアントもマイグレーションするかもしれない
  - その場合、クライアントは新旧両方のアドレスから、サーバーの新旧アドレスそれぞれに並行してパス検証を行う
- クライアントは、パス検証に成功したほうのサーバーアドレスを使う
- サーバーは、既知のクライアントアドレス以外からの接続に備える必要がある
  - 前述の攻撃と区別できないので
- サーバーがプロービングパケットを異なるアドレスから受信したら、パス検証を行ってよい
  - パス検証が完了するまでは、輻輳ウィンドウの最小データ量しか送ってはいけない
- クライアントは、サーバーの優先アドレスのファミリーにあわせてマイグレーションするべき

### 9.7.  Use of IPv6 Flow-Label and Migration

- IPv6をつかってデータを送る場合は、フローラベルを適用する必要がある
  - RFC6437
  - APIでそれを禁止されていない限り
- ラベルは、以前のラベルとの関連が最小限になるようにするべき
  - 値はランダムでそれらしいものを
  - 実装としてはハッシュ関数で計算するなど

## 10. Connection Termination

- QUICコネクションを終了するには3通りのやり方がある
  - IdleTimeout
  - ImmediateClose
  - StatelessReset
- パケットを送信できる検証済のパスがなくなったときにも終了するかもしれない

### 10.1.  Closing and Draining Connection States

- 3PTO（ProveTimeOut）経過すると、コネクションは閉じられる
- ImmediateCloseで状態遷移するときは、`CONNECTION_CLOSE`フレームを含まないパケットは送信してはいけない
  - 閉じたコネクションとしては、コネクションIDとQUICバージョンだけあれば十分
- 対向がコネクションを閉じようとしていることがわかったら、こちらもdrain状態に遷移する
  - こうなったらもうパケットを送信してはいけない
- 処理の途中でコネクションが破棄されると、それによって不要なStatelessResetが発生する可能性がある
  - それを避けるために、先にUDPソケットを閉じてしまうなど処理を省略してもよい
- 終了処理が終わったら、すべての接続状態を破棄する必要がある
  - そうすることで、新たな接続を待ち受けることができる
- StatelessResetが送信された場合は、drainなどの期間は適応されない
- この間、キーの更新処理はできない
- マイグレーションを受け付けることもできるが、送信するパケット数は制限される
  - もしくはパケットを破棄することを選んでもよい

### 10.2.  Idle Timeout

- IdleTimeoutはどちらかのピアが有効にしている場合に機能する
- `max_idle_timeout`と3PTO（ProveTimeOut）が経過したら、コネクションが閉じられる
- `max_idle_timeout`値は、どちらも指定できるが、より小さいほうが推奨される
  - 待ちきれない場合、ImmediateCloseするかも
- パケットを正常に受信できたら、タイマーをリセットする
  - ack-elictingパケットを送信したときにもリセットできる
  - draft-ietf-quic-recovery
- フロー制御の制限下にあるときは、アプリケーションデータが送信できない
  - それでタイムアウトしてしまわないように、何らかのパケットを送信する必要があるかも
- タイムアウト間近な場合、送信したパケットが捨てられるかもしれない
  - 対向がすでにdrainをはじめているかもしれないから
- PTO（ProveTimeOut）の時間内にタイムアウトする可能性がある場合
  - 重要なデータを送信する前に、接続が生きていることを確認することを推奨

### 10.3.  Immediate Close

- `CONNECTION_CLOSE`フレームを送ることで、ただちに接続を終了できる
  - 送信すると、closing状態になる
- closing期間も、受信パケットには応答してもよい
  - その場合は、`CONNECTION_CLOSE`フレームを生成する数を制限すべき
- closing期間にはいると、パケット保護キーを破棄する
  - なので無効なパケットかどうか識別できなくなる
  - よって、`CONNECTION_CLOSE`フレームを送信する頻度は減らす必要がある
- 管理すべき状態を最小限にするために、同一のパケットを送信してもよい
  - 本来はパケットごとに新しいパケット番号を払い出すもの
  - この最終パケットだからこそ許される
- `CONNECTION_CLOSE`フレームを受信すると、drain状態になる
  - その前に、`CONNECTION_CLOSE`フレームと`NO_ERROR`コードを含むパケットを1つだけ送信するかも
  - `CONNECTION_CLOSE`フレームを互いに送り合い続けることを回避する
- ImmediateCloseは、アプリケーションが接続終了を指示したときに使われる
  - 接続が終了したことを通知できる
- `CONNECTION_CLOSE`フレームを確実に処理してもらうために、パケット保護レベルは最高にする

### 10.4.  Stateless Reset

- StatelessResetは、コネクションの状態にアクセスできなくなった時の最後の手段
  - アクティブな接続と関連付けられないパケットを受信したときに送ってもよい
  - 状態がわかっているなら、`CONNECTION_CLOSE`フレームを送って終了すべき
- `NEW_CONNECTION_ID`フレームに、`Stateless Reset Token`フィールドがある
  - サーバーは、`stateless_reset_token`パラメータでも
  - このトークンは、`RETIRE_CONNECTION_ID`フレームで退役したときに無効化される
- StatelessResetパケットは、Shortヘッダを持つ通常のパケットと同様にできる
  - 予測不可能な値で埋められて、最後の16バイトにトークンが入る
  - 予測不可能な値の長さは、条件によって様々
- 今後のバージョンでは、Longヘッダを使ってもよいとなるかもしれない
  - ただLongヘッダは接続確立後にのみ使用される
  - 接続確立までは、Longヘッダをもつ未知のパケットを無視する
- Shortヘッダの場合、SCIDがわからないため、StatelessResetパケットにDCIDを設定できない
  - DCIDがランダムになるということは、`NEW_CONNECTION_ID`フレームを使用したように見える
- ランダムなコネクションIDには2つ問題がある
  - 誤ってルーティングされてしまい、パケットが届かないかもしれない
    - 別のStatelessResetを引き起こすかもしれない
  - ときどき異なるコネクションIDを使用するエンドポイントが誤解するかもしれない
- これらはあくまでQUICのバージョン1に特有の仕様である

#### 10.4.1.  Detecting a Stateless Reset

- UDPの最後の16バイトで、StatelessResetを検出する
  - 検出したら、あらかじめ生成したトークンと照合
  - 合致するものがあれば、そのアドレスがStatelessResetを行ったと判断できる
- 正常に処理できるパケットの場合、処理をスキップしてもよい
  - 正常に処理できる = 予測不可能な値ではない = StatelessResetパケットではない
- 使用していないコネクションIDに紐付いたトークンはチェックしてはいけない
- StatelessResetだと判断したら、それ以上はパケットを送信してはいけない

#### 10.4.2.  Calculating a Stateless Reset Token

- トークンは予測不可能な値である必要がある
  - コネクションごとにランダムなシークレットを生成しておくこともできる
  - ただし最適なアプローチではない
- たとえば`HMAC(static_key, connection_id)`するなどする
  - 出力結果を16バイトに切り捨てる
- このパターンは、コネクションIDに依存している
  - 常に同じ長さのコネクションIDが必要
  - 状態がない場合でも判断できるように、長さ0のコネクションIDは使わない
- トークンを公開すると、接続終了されてしまうので、値は1度しか使用できないようにする
  - staticなキーも、別の接続に使用してはいけない
  - たまたまコネクションIDが被ったり、攻撃者がルーティングを変更できる場合に困る
- トークンの値を以前のトークンすべてに重複してないかチェックする必要はない
  - ただ重複を`PROTOCOL_VIOLATION`エラーとしてもよい
- StatelessResetパケットには、暗号保護がないことに注意

#### 10.4.3.  Looping

- あるサーバーが別のサーバーにStatelessResetを送信した場合
  - そこでもStatelessResetが受信される
  - 無限に交換されてしまう可能性がある
- StatelessResetを引き起こしたパケットよりも、送信するStatelessResetパケットは、小さくなることを保証する必要がある
- サイズを41バイト以下にすると、コネクションIDの長さによっては、StatelessResetパケットであることが判断できてしまう
  - ネットワーク管理者などから

## 11. Error Handling

- エラーを検知したエンドポイントは、対向にそれを通知すべき
- トランスポートレベルとアプリケーションレベルのエラーがある
  - どちらも接続全体に影響を与える
  - アプリケーションレベルのエラーは、ストリームに分離もできる
- エラーコードは、エラーを通知するフレームに含まれるべき
  - エラーの条件が特定される場合は、エラーコードも特定のものになる
  - `PROTOCOL_VIOLATION`や`INTERNAL_ERROR`は、汎用的に使用できる
- StatelessResetは、`CONNECTION_CLOSE`や`RESET_STREAM`フレームと区別する
  - StatelessResetは最後の手段なので

### 11.1.  Connection Errors

- 明らかなエラーは、`CONNECTION_CLOSE`フレームを送る
  - 1つのストリームだけが影響される場合でも、コネクションを閉じてよい
- `CONNECTION_CLOSE`フレームにはアプリケーション固有のフィールドがある
  - トランスポートのエラーは、QUIC固有のフィールドにいれる
- 終了したコネクションで多くのパケットを受信したら、`CONNECTION_CLOSE`フレームを再送すべき
  - 再送回数や送信する時間を制限するとよい
- 再送しないことを選択する場合もある
  - その場合はStatelessResetするしかない

### 11.2.  Stream Errors

- アプリケーションレベルのエラーが起きて、単一のストリームにだけ影響を与える場合
  - `RESET_STREAM`フレームを送ってそのストリームだけを終わらせることもできる
  - 適切なエラーコードを含める
- `RESET_STREAM`フレームは、アプリケーション側による指示なしでは使用しない
  - 状態が回復できなくなることがある
- アプリケーションがそのAPIを呼び、リモート側は`STOP_SENDING`フレームを送る
  - それで`RESET_STREAM`が実行される
- これらの手順をアプリケーション側でまとめてあるべき

12. Packets and Frames

12.1.  Protected Packets

12.2.  Coalescing Packets

12.3.  Packet Numbers

12.4.  Frames and Frame Types

13. Packetization and Reliability

13.1.  Packet Processing

13.2.  Generating Acknowledgements

13.2.1.  Sending ACK Frames

13.2.2.  Managing ACK Ranges

13.2.3.  Receiver Tracking of ACK Frames

13.2.4.  Limiting ACK Ranges

13.2.5.  Measuring and Reporting Host Delay

13.2.6.  ACK Frames and Packet Protection

13.3.  Retransmission of Information

13.4.  Explicit Congestion Notification

13.4.1.  ECN Counts

13.4.2.  ECN Validation

14. Packet Size

14.1.  Path Maximum Transmission Unit (PMTU)

14.2.  ICMP Packet Too Big Messages

14.3.  Datagram Packetization Layer PMTU Discovery

14.3.1.  PMTU Probes Containing Source Connection ID

15. Versions

16. Variable-Length Integer Encoding

17. Packet Formats

17.1.  Packet Number Encoding and Decoding

17.2.  Long Header Packets

17.2.1.  Version Negotiation Packet

17.2.2.  Initial Packet

17.2.3.  0-RTT

17.2.4.  Handshake Packet

17.2.5.  Retry Packet

17.3.  Short Header Packets

17.3.1.  Latency Spin Bit

18. Transport Parameter Encoding

18.1.  Reserved Transport Parameters

18.2.  Transport Parameter Definitions

19. Frame Types and Formats

19.1.  PADDING Frame

19.2.  PING Frame

19.3.  ACK Frames

19.3.1.  ACK Ranges

19.3.2.  ECN Counts

19.4.  RESET_STREAM Frame

19.5.  STOP_SENDING Frame

19.6.  CRYPTO Frame

19.7.  NEW_TOKEN Frame

19.8.  STREAM Frames

19.9.  MAX_DATA Frame

19.10. MAX_STREAM_DATA Frame

19.11. MAX_STREAMS Frames

19.12. DATA_BLOCKED Frame

19.13. STREAM_DATA_BLOCKED Frame

19.14. STREAMS_BLOCKED Frames

19.15. NEW_CONNECTION_ID Frame

19.16. RETIRE_CONNECTION_ID Frame

19.17. PATH_CHALLENGE Frame

19.18. PATH_RESPONSE Frame

19.19. CONNECTION_CLOSE Frames

19.20. HANDSHAKE_DONE frame

19.21. Extension Frames

20. Transport Error Codes

20.1.  Application Protocol Error Codes

21. Security Considerations

21.1.  Handshake Denial of Service

21.2.  Amplification Attack

21.3.  Optimistic ACK Attack

21.4.  Slowloris Attacks

21.5.  Stream Fragmentation and Reassembly Attacks

21.6.  Stream Commitment Attack

21.7.  Peer Denial of Service

21.8.  Explicit Congestion Notification Attacks

21.9.  Stateless Reset Oracle

21.10. Version Downgrade

21.11. Targeted Attacks by Routing

21.12. Overview of Security Properties

21.12.1.  Handshake

21.12.2.  Protected Packets

21.12.3.  Connection Migration

22. IANA Considerations

22.1.  Registration Policies for QUIC Registries

22.1.1.  Provisional Registrations

22.1.2.  Selecting Codepoints

22.1.3.  Reclaiming Provisional Codepoints

22.1.4.  Permanent Registrations

22.2.  QUIC Transport Parameter Registry

22.3.  QUIC Frame Type Registry

22.4.  QUIC Transport Error Codes Registry

23. References

23.1.  Normative References

23.2.  Informative References

Appendix A.  Sample Packet Number Decoding Algorithm

Appendix B.  Sample ECN Validation Algorithm

Appendix C.  Change Log

C.1.  Since draft-ietf-quic-transport-24

C.2.  Since draft-ietf-quic-transport-23

C.3.  Since draft-ietf-quic-transport-22

C.4.  Since draft-ietf-quic-transport-21

C.5.  Since draft-ietf-quic-transport-20

C.6.  Since draft-ietf-quic-transport-19

C.7.  Since draft-ietf-quic-transport-18

C.8.  Since draft-ietf-quic-transport-17

C.9.  Since draft-ietf-quic-transport-16

C.10. Since draft-ietf-quic-transport-15

C.11. Since draft-ietf-quic-transport-14

C.12. Since draft-ietf-quic-transport-13

C.13. Since draft-ietf-quic-transport-12

C.14. Since draft-ietf-quic-transport-11

C.15. Since draft-ietf-quic-transport-10

C.16. Since draft-ietf-quic-transport-09

C.17. Since draft-ietf-quic-transport-08

C.18. Since draft-ietf-quic-transport-07

C.19. Since draft-ietf-quic-transport-06

C.20. Since draft-ietf-quic-transport-05

C.21. Since draft-ietf-quic-transport-04

C.22. Since draft-ietf-quic-transport-03

C.23. Since draft-ietf-quic-transport-02

C.24. Since draft-ietf-quic-transport-01

C.25. Since draft-ietf-quic-transport-00

C.26. Since draft-hamilton-quic-transport-protocol-01

Contributors

Authors' Addresses
