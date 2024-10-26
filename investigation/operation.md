## Operation Transformation調査

### 登場人物
 - SimpleTextOperation
 - TextOperation
 - Selection

## TextOperation

 - ops : 操作を格納する
 - baseLength : 操作を適用する文字列の長さ
 - targetLength : 操作を適用した結果の文字列の長さ

### 関数

#### 種別の判定

##### isRetain
nだけカーソルを進める
n > 0

##### isInsert
現在のカーソルの位置に文字列を挿入する

######### isDelete
現在のカーソルの位置から、nだけ文字を削除する
n < 0

#### 操作の適用

操作の関数は自身を返り値とするため、処理をchainしやすい

##### retain
カーソルを進める
baseLength + n
targetLength + n

1.配列の最後の要素がRetainなら、last.n + n

2.配列の最後の要素がRetain意外なら、配列にpushする

##### insert
文字列挿入
targetLength + n

1.配列の最後の要素が**挿入**
配列の最後の要素に文字列を連結

2.配列の最後の要素が**削除**

 2-1. 配列のlength-2の要素が、**挿入**
 配列のlength-2の要素に文字列を連結

 2-2. 配列のlength-2の要素が、**挿入**以外
 削除の処理と挿入の処理をswapする

##### delete
文字列の削除
baseLength - n

1.配列の最後の要素が**削除**
length-1の削除数 + n

2.配列の最後の要素が**削除**以外
配列に操作をpush

##### isNoop
影響0の操作の判定

 - opsの配列の長さが0
 - カーソルを進める処理のみ

##### toString
ログ用、各要素の文字列にして連結

##### fromJson
JsonArrayをTextOperationに変換する

##### ⭐️apply
操作を適用される
str.lengthと operation.baseLengthが一致しないときは、
エラーをスローする

```javascript
    var newStr = [], j = 0;
    var strIndex = 0;
    var ops = this.ops;
```

1.**保持**の場合
```javascript
        newStr[j++] = str.slice(strIndex, strIndex + op);
        strIndex += op;
```
strIndexからnだけの文字列を配列に格納
strIndexを更新

2.**挿入**の場合
```javascript
newStr[j++] = op;
```
配列に文字列格納

3.**削除**の場合
```javascript
strIndex -= op;
```


##### ⭐️invert










