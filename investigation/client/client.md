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

### Client.applyClient (ユーザー入力が行われたとき呼ばれる)
```javascript
  // Call this method when the user changes the document.
  Client.prototype.applyClient = function (operation) {
    this.setState(this.state.applyClient(this, operation));
  };
```

### Client.applyServer (サーバーからの操作を受け取った時に呼ばれる)
```javascript
  // Call this method with a new operation from the server
  Client.prototype.applyServer = function (operation) {
    this.revision++;
    this.setState(this.state.applyServer(this, operation));
  };
```

### サーバーからAckを受け取ったときは、revisionを+1
```javascript
  Client.prototype.serverAck = function () {
    this.revision++;
    this.setState(this.state.serverAck(this));
  };
```

### サーバーとの通信が回復した時、再送信できるステータスなら再送信
```javascript
  Client.prototype.serverReconnect = function () {
    if (typeof this.state.resend === 'function') { this.state.resend(this); }
  };
```

### サーバーのカーソル位置に、クライアントの操作を適用する
```javascript
  Client.prototype.transformSelection = function (selection) {
    return this.state.transformSelection(selection);
  };
```