# Editor Socket IO Server

## Constructor

```javascript
function EditorSocketIOServer (document, operations, docId, mayWrite) {
    EventEmitter.call(this);
    Server.call(this, document, operations);
    this.users = {};
    this.docId = docId;
    this.mayWrite = mayWrite || function (_, cb) { cb(true); };
}
```

document: 初期のドキュメント状態。
operations: ドキュメントに対して行われた操作の履歴。
docId: ドキュメントの一意のID。
mayWrite: クライアントが編集可能かを判定する関数（デフォルトで常に true を返す）。


```javascript
util.inherits(EditorSocketIOServer, Server);
extend(EditorSocketIOServer.prototype, EventEmitter.prototype);
```
EventEmitter と Server を継承しており、イベントの発行や操作の管理が可能です。

## extend
```javascript
function extend (target, source) {
  for (var key in source) {
    if (source.hasOwnProperty(key)) {
      target[key] = source[key];
    }
  }
}
```
EventEmitter と Server からメソッドを
EditorSocketIOServer に拡張するための補助関数。

## addClient

```javascript
EditorSocketIOServer.prototype.addClient = function (socket) {
    var self = this;
    socket
        .join(this.docId)
        .emit('doc', {
            str: this.document,
            revision: this.operations.length,
            clients: this.users
        })
        .on('operation', function (revision, operation, selection) {
            self.mayWrite(socket, function (mayWrite) {
                if (!mayWrite) {
                    console.log("User doesn't have the right to edit.");
                    return;
                }
                self.onOperation(socket, revision, operation, selection);
            });
        })
        .on('selection', function (obj) {
            self.mayWrite(socket, function (mayWrite) {
                if (!mayWrite) {
                    console.log("User doesn't have the right to edit.");
                    return;
                }
                self.updateSelection(socket, obj && Selection.fromJSON(obj));
            });
        })
        .on('disconnect', function () {
            console.log("Disconnect");
            socket.leave(self.docId);
            self.onDisconnect(socket);
            if (
                (socket.manager && socket.manager.sockets.clients(self.docId).length === 0) || // socket.io <= 0.9
                (socket.ns && Object.keys(socket.ns.connected).length === 0) // socket.io >= 1.0
            ) {
                self.emit('empty-room');
            }
        });
};
```

クライアントが接続した際に次の処理を行います：
 - ドキュメントの初期状態をクライアントに送信。
- クライアントの operation イベントをリッスンし、操作の許可をチェックしてから onOperation メソッドで操作を処理。
- クライアントが選択範囲を送信した場合、updateSelection で他のクライアントに選択範囲を通知。
- クライアントが切断された場合、onDisconnect を呼び出してクライアントを削除し、必要なら部屋が空になったことを通知。

## onOperation

```javascript
EditorSocketIOServer.prototype.onOperation = function (socket, revision, operation, selection) {
    var wrapped;
    try {
        wrapped = new WrappedOperation(
            TextOperation.fromJSON(operation),
            selection && Selection.fromJSON(selection)
        );
    } catch (exc) {
        console.error("Invalid operation received: " + exc);
        return;
    }

    try {
        var clientId = socket.id;
        var wrappedPrime = this.receiveOperation(revision, wrapped);
        console.log("new operation: " + wrapped);
        this.getClient(clientId).selection = wrappedPrime.meta;
        socket.emit('ack');
        socket.broadcast['in'](this.docId).emit(
            'operation', clientId,
            wrappedPrime.wrapped.toJSON(), wrappedPrime.meta
        );
    } catch (exc) {
        console.error(exc);
    }
};
```

WrappedOperationに、Operation、Selectionを格納する
receiveOperationを呼ぶ

selectionの配列を更新

受け取った操作をサーバーに適用し、
クライアントに ack を送り、他のクライアントにブロードキャストします。

## updateSelection
```javascript
EditorSocketIOServer.prototype.updateSelection = function (socket, selection) {
  var clientId = socket.id;
  if (selection) {
    this.getClient(clientId).selection = selection;
  } else {
    delete this.getClient(clientId).selection;
  }
  socket.broadcast['in'](this.docId).emit('selection', clientId, selection);
};
```

 - クライアントの選択範囲を更新し、他のクライアントにブロードキャストします。
 - selection が存在しない場合は選択範囲を削除し、 存在する場合は現在の選択範囲を users に保存。

## setName
```javascript
EditorSocketIOServer.prototype.setName = function (socket, name) {
  var clientId = socket.id;
  this.getClient(clientId).name = name;
  socket.broadcast['in'](this.docId).emit('set_name', clientId, name);
};
```

クライアントの名前を設定し、他のクライアントに名前をブロードキャストします。

## getClient
```javascript
EditorSocketIOServer.prototype.getClient = function (clientId) {
  return this.users[clientId] || (this.users[clientId] = {});
};
```

 - 指定したクライアントの情報を取得します。
 - クライアントが存在しない場合は新しいクライアントオブジェクトを生成し、users に追加してから返します。

## onDisconnect
```javascript
EditorSocketIOServer.prototype.onDisconnect = function (socket) {
  var clientId = socket.id;
  delete this.users[clientId];
  socket.broadcast['in'](this.docId).emit('client_left', clientId);
};
```
 - クライアントが切断された際に呼ばれ、切断されたクライアントを削除し、他のクライアントに client_left イベントを送信します。

