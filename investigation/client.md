# Client

## Constructor

```javascript
  function Client (revision) {
    this.revision = revision; // the next expected revision number
    this.state = synchronized_; // start state
  }
```

revision : バージョン
state : 

```javascript
function Synchronized () {}
function AwaitingConfirm (outstanding) {
    // Save the pending operation
    this.outstanding = outstanding;
}
function AwaitingWithBuffer (outstanding, buffer) {
    // Save the pending operation and the user's edits since then
    this.outstanding = outstanding;
    this.buffer = buffer;
}
```

このコードは、リアルタイム共同編集システムで使われる**クライアント側の状態管理クラス**を定義しています。それぞれのクラスは、サーバーとの同期状態に応じた異なるクライアントの状態を表現します。以下に、それぞれのクラスについて詳しく解説します。

### クラスの概要
1. `Synchronized`
2. `AwaitingConfirm`
3. `AwaitingWithBuffer`

これらのクラスは、クライアントの状態遷移に対応し、
クライアントがサーバーとのやり取りを管理するための状態管理パターンを提供しています。
リアルタイム編集ツールにおいて、クライアントはこのような状態を通じて操作の同期を行い、
編集内容の一貫性を保ちます。

### 1. `Synchronized`
```javascript
function Synchronized () {}
```
`Synchronized` は、クライアントが完全に同期されている状態を表します。
この状態では、クライアントはサーバーと最新の状態が一致しており、
特に保留中の操作やバッファはありません。
この状態から、ユーザーが編集操作を行うと、`AwaitingConfirm` 状態に移行することになります。

### 2. `AwaitingConfirm`
```javascript
function AwaitingConfirm (outstanding) {
    // Save the pending operation
    this.outstanding = outstanding;
}
```
`AwaitingConfirm` は、クライアントがサーバーに操作を送信し、
その応答を待っている状態を表します。
この状態には次のような特徴があります：

- **保留中の操作（`outstanding`）を保持**: 
ユーザーが編集操作を行うと、クライアントはその操作をサーバーに送信しますが、
サーバーからの確認応答（ACK）を受け取るまで、`outstanding` プロパティにその操作を保存しておきます。

- **サーバーからの応答待ち**: この状態ではサーバーからの応答を待っており、
サーバーが確認応答を返すと `Synchronized` 状態に戻ります。

### 3. `AwaitingWithBuffer`
```javascript
function AwaitingWithBuffer (outstanding, buffer) {
    // Save the pending operation and the user's edits since then
    this.outstanding = outstanding;
    this.buffer = buffer;
}
```
`AwaitingWithBuffer` は、クライアントが保留中の操作に加えて、
サーバーからの応答を待つ間に追加で編集が行われた状態です。この状態には次の特徴があります：

- **`outstanding`**: `AwaitingConfirm` と同様、サーバーに送信したがまだ応答がない操作を保持しています。
- **`buffer`**: `AwaitingConfirm` 状態でさらにユーザーが編集を行った場合、
その新しい操作が `buffer` に追加されます。 
この状態でクライアントがサーバーの応答を待ちながらユーザーがさらに編集操作を行っても、
クライアントは一貫した操作順序を維持できます。

### まとめ
- **`Synchronized`**: クライアントとサーバーが完全に同期している状態。
- **`AwaitingConfirm`**: クライアントがサーバーに操作を送信し、応答待ちで追加操作がない状態。
- **`AwaitingWithBuffer`**: 保留中の操作がサーバーに送信され、さらに追加の操作がバッファされている状態。

この仕組みによって、クライアントはリアルタイムでユーザーの操作を管理し、
サーバーとの同期の遅延があっても、編集の一貫性を維持することができます。