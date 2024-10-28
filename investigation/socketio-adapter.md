# Socket Io Adapter

## Constructor

```javascript

  function SocketIOAdapter (socket) {
    this.socket = socket;

    var self = this;
    socket
      .on('client_left', function (clientId) {
        self.trigger('client_left', clientId);
      })
      .on('set_name', function (clientId, name) {
        self.trigger('set_name', clientId, name);
      })
      .on('ack', function () { self.trigger('ack'); })
      .on('operation', function (clientId, operation, selection) {
        self.trigger('operation', operation);
        self.trigger('selection', clientId, selection);
      })
      .on('selection', function (clientId, selection) {
        self.trigger('selection', clientId, selection);
      })
      .on('reconnect', function () {
        self.trigger('reconnect');
      });
  }

```
サーバーからWebSocket通信で受信した値を、Callbackに渡す

## sendOperation
```javascript
  SocketIOAdapter.prototype.sendOperation = function (revision, operation, selection) {
    this.socket.emit('operation', revision, operation, selection);
  };
```
socketに操作を送信

## sendSelection
```javascript
  SocketIOAdapter.prototype.sendSelection = function (selection) {
    this.socket.emit('selection', selection);
  };
```
選択範囲を送信

## registerCallbacks
```javascript
  SocketIOAdapter.prototype.registerCallbacks = function (cb) {
    this.callbacks = cb;
  };
```
コールバックを登録

## trigger
eventをキーに、コールバックを呼ぶ