# Synchronized

## ユーザー入力が行われたとき
```javascript
  Synchronized.prototype.applyClient = function (client, operation) {
    client.sendOperation(client.revision, operation);
    return new AwaitingConfirm(operation);
  };
```
現在のrevisionと操作を送り、**AwaitingConfirm**に遷移

### サーバーから操作を受け取ったとき
```javascript
  Synchronized.prototype.applyServer = function (client, operation) {
    client.applyOperation(operation);
    return this;
  };
```
操作を適用して、Synchronizedのまま

### Ackを受け取ったとき
```javascript
  Synchronized.prototype.serverAck = function (client) {
    throw new Error("There is no pending operation.");
  };
```
エラーを投げる

### 選択範囲の変更
```javascript
Synchronized.prototype.transformSelection = function (x) { return x; };
```

そのまま返す
