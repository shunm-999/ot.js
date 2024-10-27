# AwaitingWithBuffer

```javascript
  // In the 'AwaitingWithBuffer' state, the client is waiting for an operation
  // to be acknowledged by the server while buffering the edits the user makes
  function AwaitingWithBuffer (outstanding, buffer) {
    // Save the pending operation and the user's edits since then
    this.outstanding = outstanding;
    this.buffer = buffer;
  }
```

AwaitingWithBuffer は、
クライアントが保留中の操作に加えて、サーバーからの応答を待つ間に追加で編集が行われた状態です。
この状態には次の特徴があります：

outstanding: AwaitingConfirm と同様、サーバーに送信したがまだ応答がない操作を保持しています。
buffer: AwaitingConfirm 状態でさらにユーザーが編集を行った場合、
その新しい操作が buffer に追加されます。
この状態でクライアントがサーバーの応答を待ちながらユーザーがさらに編集操作を行っても、
クライアントは一貫した操作順序を維持できます。


## ユーザー入力が行われたとき
```javascript
  AwaitingWithBuffer.prototype.applyClient = function (client, operation) {
    // Compose the user's changes onto the buffer
    var newBuffer = this.buffer.compose(operation);
    return new AwaitingWithBuffer(this.outstanding, newBuffer);
  };
```
bufferと新しい操作をマージする

### サーバーから操作を受け取ったとき
```javascript
  AwaitingWithBuffer.prototype.applyServer = function (client, operation) {
    // Operation comes from another client
    //
    //                       /\
    //     this.outstanding /  \ operation
    //                     /    \
    //                    /\    /
    //       this.buffer /  \* / pair1[0] (new outstanding)
    //                  /    \/
    //                  \    /
    //          pair2[1] \  / pair2[0] (new buffer)
    // the transformed    \/
    // operation -- can
    // be applied to the
    // client's current
    // document
    //
    // * pair1[1]
    var transform = operation.constructor.transform;
    var pair1 = transform(this.outstanding, operation);
    var pair2 = transform(this.buffer, pair1[1]);
    client.applyOperation(pair2[1]);
    return new AwaitingWithBuffer(pair1[0], pair2[0]);
  };
```

保持している操作と、サーバーから受け取った操作を適用
バッファしている操作と、変更済みのサーバーから受け取った操作を適用
変更済みのサーバーから受け取った操作をドキュメントに適用

保持 : サーバーの操作を適用した、保持している操作
バッファ : サーバーの操作を適用した、バッファ

### Ackを受け取ったとき

```javascript
  AwaitingWithBuffer.prototype.serverAck = function (client) {
    // The pending operation has been acknowledged
    // => send buffer
    client.sendOperation(client.revision, this.buffer);
    return new AwaitingConfirm(this.buffer);
  };
```

バッファを送信して、AwaitingConfirmに遷移する

### 選択範囲の変更
```javascript
  AwaitingWithBuffer.prototype.transformSelection = function (selection) {
    return selection.transform(this.outstanding).transform(this.buffer);
  };
```

選択範囲に、保持している操作と、バッファを適用

### 再送信
```javascript
  AwaitingWithBuffer.prototype.resend = function (client) {
    // The confirm didn't come because the client was disconnected.
    // Now that it has reconnected, we resend the outstanding operation.
    client.sendOperation(client.revision, this.outstanding);
  };
```
保持している操作を再送信

