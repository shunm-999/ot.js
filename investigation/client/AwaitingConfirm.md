# AwaitingConfirm

AwaitingConfirm は、クライアントがサーバーに操作を送信し、その応答を待っている状態を表します。

この状態には次のような特徴があります：
保留中の操作（outstanding）を保持: ユーザーが編集操作を行うと、
クライアントはその操作をサーバーに送信しますが、
サーバーからの確認応答（ACK）を受け取るまで、outstanding プロパティにその操作を保存しておきます。
サーバーからの応答待ち: この状態ではサーバーからの応答を待っており、
サーバーが確認応答を返すと Synchronized 状態に戻ります。

## Constructor
```javascript
  function AwaitingConfirm (outstanding) {
    // Save the pending operation
    this.outstanding = outstanding;
  }
```

応答待ちの操作を保持

## ユーザー入力が行われたとき
```javascript
  AwaitingConfirm.prototype.applyClient = function (client, operation) {
    // When the user makes an edit, don't send the operation immediately,
    // instead switch to 'AwaitingWithBuffer' state
    return new AwaitingWithBuffer(this.outstanding, operation);
  };
```

AwaitingWithBufferに遷移

### サーバーから操作を受け取ったとき
```javascript
  AwaitingConfirm.prototype.applyServer = function (client, operation) {
    // This is another client's operation. Visualization:
    //
    //                   /\
    // this.outstanding /  \ operation
    //                 /    \
    //                 \    /
    //  pair[1]         \  / pair[0] (new outstanding)
    //  (can be applied  \/
    //  to the client's
    //  current document)
    var pair = operation.constructor.transform(this.outstanding, operation);
    client.applyOperation(pair[1]);
    return new AwaitingConfirm(pair[0]);
  };

```
応答待ちの操作に、サーバーからきた操作を適用して、保持する
サーバーから来た操作に応答待ちの操作を適用して、ドキュメントに適用

AwaitingConfirmのまま

### Ackを受け取ったとき
```javascript
  AwaitingConfirm.prototype.serverAck = function (client) {
    // The client's operation has been acknowledged
    // => switch to synchronized state
    return synchronized_;
  };
```

Synchronizedに遷移

### 選択範囲の変更
```javascript
  AwaitingConfirm.prototype.transformSelection = function (selection) {
    return selection.transform(this.outstanding);
  };
```
サーバーから受け取った選択範囲に、応答待ちの操作を適用する

