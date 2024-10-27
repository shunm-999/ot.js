# Server

## Serverクラス
document : String
operations : operationの配列

### receiveOperation

revision : バージョン
operation : 操作

#### リビジョン確認
```javascript
    if (revision < 0 || this.operations.length < revision) {
    throw new Error("operation revision not in history");
}
```
クライアントから送られてきた操作のリビジョンが、
サーバーが保持しているリビジョンより進んでいる場合は、エラーを投げる

#### 並行操作の抽出
```javascript
    var concurrentOperations = this.operations.slice(revision);
```
クライアントが知らないサーバー上の最新の操作をすべて取り出す

#### 操作の変換
```javascript
    var transform = operation.constructor.transform;
    for (var i = 0; i < concurrentOperations.length; i++) {
      operation = transform(operation, concurrentOperations[i])[0];
    }
```

取り出したサーバーの最新の操作を、受け取った操作に適用する

#### ドキュメントへの操作適用
```javascript
    // ... and apply that on the document.
    this.document = operation.apply(this.document);
    // Store operation in history.
    this.operations.push(operation);
```

transformした操作を、ドキュメントに適用
operationsの配列にpush