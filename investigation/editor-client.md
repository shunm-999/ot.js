# Editor Client

## SelfMeta クラス

自分自身の操作に関連するメタデータを管理します。
selectionBefore と selectionAfter によって、操作前と操作後の選択範囲を保持。
invert、compose、transform メソッドを持ち、操作を逆転、合成、変換する機能を提供します。


```javascript
  SelfMeta.prototype.invert = function () {
    return new SelfMeta(this.selectionAfter, this.selectionBefore);
  };

  SelfMeta.prototype.compose = function (other) {
    return new SelfMeta(this.selectionBefore, other.selectionAfter);
  };

  SelfMeta.prototype.transform = function (operation) {
    return new SelfMeta(
      this.selectionBefore.transform(operation),
      this.selectionAfter.transform(operation)
    );
  };

```

## OtherMeta クラス

他のクライアントの操作に関連するメタデータを管理します。
clientId と selection を保持し、fromJSON メソッドでJSONデータからの初期化をサポート。
transform メソッドにより、他のクライアントの操作に基づいてメタデータを更新します。

```javascript
  function OtherMeta (clientId, selection) {
    this.clientId  = clientId;
    this.selection = selection;
  }

  OtherMeta.fromJSON = function (obj) {
    return new OtherMeta(
      obj.clientId,
      obj.selection && Selection.fromJSON(obj.selection)
    );
  };

  OtherMeta.prototype.transform = function (operation) {
    return new OtherMeta(
      this.clientId,
      this.selection && this.selection.transform(operation)
    );
  };

```


### OtherClient クラス

#### Constructor
```javascript
  function OtherClient (id, listEl, editorAdapter, name, selection) {
    this.id = id;
    this.listEl = listEl;
    this.editorAdapter = editorAdapter;
    this.name = name;

    this.li = document.createElement('li');
    if (name) {
      this.li.textContent = name;
      this.listEl.appendChild(this.li);
    }

    this.setColor(name ? hueFromName(name) : Math.random());
    if (selection) { this.updateSelection(selection); }
  }
```
他のクライアントの情報を管理します。
プロパティ:
id: クライアントID。
name: クライアント名。
selection: 現在の選択範囲。

listElには、現在のクライアントのlistElementが入ってくる

### setColor / setName
色と名前の指定


### updateSelection 

```javascript
  OtherClient.prototype.updateSelection = function (selection) {
    this.removeSelection();
    this.selection = selection;
    this.mark = this.editorAdapter.setOtherSelection(
      selection,
      selection.position === selection.selectionEnd ? this.color : this.lightColor,
      this.id
    );
  };

```
選択範囲の更新

markには以下のobjectが入る
```javascript
{
      clear: function () {
        for (var i = 0; i < selectionObjects.length; i++) {
          selectionObjects[i].clear();
        }
      }
};
```

### remove

```javascript
  OtherClient.prototype.remove = function () {
    if (this.li) { removeElement(this.li); }
    this.removeSelection();
  };
```
他のクライアントが離脱した時に呼ばれる

### removeSelection
```javascript
  OtherClient.prototype.removeSelection = function () {
    if (this.mark) {
      this.mark.clear();
      this.mark = null;
    }
  };
```
他のクライアントの選択範囲を消す

## EditorClient

EditorClient クラス

このクラスはエディターのメインクライアントを表し、サーバーとの通信と共同編集の状態管理を行います。

主なプロパティ:

 - serverAdapter: サーバーへの接続を管理。
- editorAdapter: エディターインターフェースを操作するためのアダプタ。
- undoManager: 取り消し/やり直し操作の管理。
- clients: 他のクライアントの情報を格納するオブジェクト。
- clientListEl: 他のクライアントのリストを表示するためのHTML要素。

主なメソッド:
- addClient: 他のクライアントを追加し、表示を更新します。
- initializeClients: クライアントの初期化を行い、全クライアントの情報を設定。
- onClientLeft: クライアントが離脱した際に表示から削除。
- applyUnredo: 取り消し/やり直し操作を実行し、選択範囲と操作を更新。
- undo および redo: 取り消し、やり直しを管理。
- onChange: テキストに変更があった際の処理を行い、選択範囲と操作を undoManager に記録。
- onSelectionChange: 選択範囲が変更された際に、新しい選択範囲をサーバーに送信。
- applyOperation: サーバーから送られてきた操作をエディターに適用します。

```javascript
    this.editorAdapter.registerCallbacks({
      change: function (operation, inverse) { self.onChange(operation, inverse); },
      selectionChange: function () { self.onSelectionChange(); },
      blur: function () { self.onBlur(); }
    });
```
change : 操作が発生した時に呼ばれる
selectionChange : 選択範囲が変更された時に呼ばれる
onblur : フォーカスが離れた時に呼ばれる

```javascript
    this.editorAdapter.registerUndo(function () { self.undo(); });
    this.editorAdapter.registerRedo(function () { self.redo(); });
```
UndoとRedo

```javascript
    this.serverAdapter.registerCallbacks({
      client_left: function (clientId) { self.onClientLeft(clientId); },
      set_name: function (clientId, name) { self.getClientObject(clientId).setName(name); },
      ack: function () { self.serverAck(); },
      operation: function (operation) {
        self.applyServer(TextOperation.fromJSON(operation));
      },
      selection: function (clientId, selection) {
        if (selection) {
          self.getClientObject(clientId).updateSelection(
            self.transformSelection(Selection.fromJSON(selection))
          );
        } else {
          self.getClientObject(clientId).removeSelection();
        }
      },
      clients: function (clients) {
        var clientId;
        for (clientId in self.clients) {
          if (self.clients.hasOwnProperty(clientId) && !clients.hasOwnProperty(clientId)) {
            self.onClientLeft(clientId);
          }
        }

        for (clientId in clients) {
          if (clients.hasOwnProperty(clientId)) {
            var clientObject = self.getClientObject(clientId);

            if (clients[clientId].name) {
              clientObject.setName(clients[clientId].name);
            }

            var selection = clients[clientId].selection;
            if (selection) {
              self.clients[clientId].updateSelection(
                self.transformSelection(Selection.fromJSON(selection))
              );
            } else {
              self.clients[clientId].removeSelection();
            }
          }
        }
      },
      reconnect: function () { self.serverReconnect(); }
    });
  }
```
client_left: 他のクライアント離脱
set_name: 他のクライアントの名前が決定された時に呼ばれる
ack: サーバーからAckが帰ってきた時に呼ばれる
operation : サーバーから操作を受信した時に呼ばれる
selection : 他のクライアントの選択範囲が変化したときによばれる
clients: 他のクライアントの配列が更新されたときによばれる
reconnect : 再接続時に呼ばれる


### addClient
```javascript
  EditorClient.prototype.addClient = function (clientId, clientObj) {
    this.clients[clientId] = new OtherClient(
      clientId,
      this.clientListEl,
      this.editorAdapter,
      clientObj.name || clientId,
      clientObj.selection ? Selection.fromJSON(clientObj.selection) : null
    );
  };
```
他のクライアントの追加

### initializeClients
クライアントの初期化

### getClientObject
クライアントの取得、なければ追加

### onClientLeft
クライアントが離脱した時によばれる

### initializeClientList
クライアントリストの初期化

### applyUnredo
Undo、Redoの中身の処理
実施された操作と反対の処理を、保存しておく

### undo
Undoの実施

### redo
redoの実施

### onChange

```javascript
  EditorClient.prototype.onChange = function (textOperation, inverse) {
    var selectionBefore = this.selection;
    this.updateSelection();
    var meta = new SelfMeta(selectionBefore, this.selection);
    var operation = new WrappedOperation(textOperation, meta);

    var compose = this.undoManager.undoStack.length > 0 &&
      inverse.shouldBeComposedWithInverted(last(this.undoManager.undoStack).wrapped);
    var inverseMeta = new SelfMeta(this.selection, selectionBefore);
    this.undoManager.add(new WrappedOperation(inverse, inverseMeta), compose);
    this.applyClient(textOperation);
  };

```

パラメータ
textOperation: ユーザーによって実行されたテキスト操作を表します。これがエディターで実行された変更内容です。
inverse: textOperation の逆操作で、取り消し操作に使用されます。

##### 選択範囲の保存と更新
操作前の選択範囲を selectionBefore に保存します。
updateSelection メソッドを呼び出して、操作後の現在の選択範囲に this.selection を更新します。

##### 操作のラップ（WrappedOperation インスタンスの生成）
WrappedOperationで、メタデータ格納

##### 操作の合成判定
compose フラグを設定して、今回の操作が undoStack の最後の操作と合成できるかを判定します。
判定条件は次の通りです：
undoStack に1つ以上の操作があること。
inverse.shouldBeComposedWithInverted メソッドが true を返すこと。
合成が可能であれば、前回の操作と新しい操作が1つにまとめられます。

##### 逆操作の追加
inverse（逆操作）に関するメタデータを生成し、inverseMeta に保持します。
逆操作とそのメタデータを WrappedOperation でまとめ、undoManager に追加します。
compose が true の場合、前回の操作と合成されます。

##### クライアント側への操作の適用
クライアント側で textOperation を適用し、エディターに反映させます。
applyClient メソッドを呼び出して、サーバーや他のクライアントへの通知に備えます。

