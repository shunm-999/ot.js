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

1. 配列の最後の要素がRetainなら、last.n + n

2. 配列の最後の要素がRetain意外なら、配列にpushする

##### insert
文字列挿入
targetLength + n

1. 配列の最後の要素が**挿入**
配列の最後の要素に文字列を連結

2. 配列の最後の要素が**削除**

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

1. **保持**の場合
```javascript
        newStr[j++] = str.slice(strIndex, strIndex + op);
        strIndex += op;
```
strIndexからnだけの文字列を配列に格納
strIndexを更新

2. **挿入**の場合
```javascript
newStr[j++] = op;
```
配列に文字列格納

3. **削除**の場合
```javascript
strIndex -= op;
```

⚠️ op < 0であることに注意

##### ⭐️invert
undoのための関数

1. **保持**の場合
```javascript
        inverse.retain(op);
        strIndex += op;
```

保持のまま
strIndexを進める

2. **挿入**の場合
```javascript
        inverse['delete'](op.length);
```

削除に変更

3. **削除**の場合

```javascript
        inverse.insert(str.slice(strIndex, strIndex - op));
        strIndex -= op;
```
挿入に変更 
⚠️ op < 0であることに注意

##### ⭐️compose
連続したOperationを一つにまとめる

```javascript
    if (operation1.targetLength !== operation2.baseLength) {
      throw new Error("The base length of the second operation has to be the target length of the first operation");
    }
```
Operationが連続に並んでいることをチェック

```javascript
      if (isDelete(op1)) {
        operation['delete'](op1);
        op1 = ops1[i1++];
        continue;
      }
      if (isInsert(op2)) {
        operation.insert(op2);
        op2 = ops2[i2++];
        continue;
      }
```

op1が**削除**の場合
op1を適用し、op1を一つ進める

op2が**挿入**の場合
op2を適用し、op2を一つ進める

```javascript
      if (isRetain(op1) && isRetain(op2)) {
        if (op1 > op2) {
          operation.retain(op2);
          op1 = op1 - op2;
          op2 = ops2[i2++];
        } else if (op1 === op2) {
          operation.retain(op1);
          op1 = ops1[i1++];
          op2 = ops2[i2++];
        } else {
          operation.retain(op1);
          op2 = op2 - op1;
          op1 = ops1[i1++];
        }
      } 
```

###### op1が**保持**かつ、op2が**保持**

1. op1 > op2
op2を適用して、
op1のカーソル増加分を減らす
op2を一つ進める

2. op1 == op2
op1を適用し、
op1、op2を一つ進める

3. op1 < op2
op1を適用して、
op2のカーソル増加分を減らす
op1を一つ進める

###### op1が挿入かつ、op2が削除

```javascript
      } else if (isInsert(op1) && isDelete(op2)) {
        if (op1.length > -op2) {
          op1 = op1.slice(-op2);
          op2 = ops2[i2++];
        } else if (op1.length === -op2) {
          op1 = ops1[i1++];
          op2 = ops2[i2++];
        } else {
          op2 = op2 + op1.length;
          op1 = ops1[i1++];
        }
      } 
```

1.op1の挿入数が、op2の削除数を上回る場合
op1の最初のop2数分を削除して、更新
op2は一つ進む

2. op1の挿入数と、op2の削除数が同数の場合
op1は一つ進む
op2は一つ進む

3. op1の挿入数が、op2の削除数を下回る場合
op2の削除数を、挿入数分だけ減らして更新
op2 < 0

op1は一つ進む

###### op1が挿入かつ、op2が保持

```javascript
      else if (isInsert(op1) && isRetain(op2)) {
        if (op1.length > op2) {
          operation.insert(op1.slice(0, op2));
          op1 = op1.slice(op2);
          op2 = ops2[i2++];
        } else if (op1.length === op2) {
          operation.insert(op1);
          op1 = ops1[i1++];
          op2 = ops2[i2++];
        } else {
          operation.insert(op1);
          op2 = op2 - op1.length;
          op1 = ops1[i1++];
        }
      }
```

1. op1 の長さが op2 より大きい場合
op2の長さ分だけ、op1の文字列を挿入
op1を、op1をop2分だけ切り捨てて更新
op2は一つ進む

2. op1 の長さが op2 と等しい場合
op1を挿入
op1は一つ進む
op2は一つ進む

3. op1 の長さが op2 より小さい場合
op1を挿入
op2はop1の長さ分切り詰める
op1は一つ進む

###### op1は保持、op2が削除の場合

```javascript
else if (isRetain(op1) && isDelete(op2)) {
        if (op1 > -op2) {
          operation['delete'](op2);
          op1 = op1 + op2;
          op2 = ops2[i2++];
        } else if (op1 === -op2) {
          operation['delete'](op2);
          op1 = ops1[i1++];
          op2 = ops2[i2++];
        } else {
          operation['delete'](op1);
          op2 = op2 + op1;
          op1 = ops1[i1++];
        }
    }
```

1. op1 の長さが op2 より大きい場合
op2の長さ分、削除を実行
op1は、op2の長さ分切り詰められる
op2は一つ進む

2. op1 の長さが op2 と等しい場合
op2の長さ分、削除を実行
op1は一つ進む
op2は一つ進む

3. op1 の長さが op2 より小さい場合
op1の長さ分、削除を実行
op2は長さはop1分、切り詰められる
op1は一つ進む

##### getSimpleOp

```javascript
  function getSimpleOp (operation, fn) {
    var ops = operation.ops;
    var isRetain = TextOperation.isRetain;
    switch (ops.length) {
    case 1:
      return ops[0];
    case 2:
      return isRetain(ops[0]) ? ops[1] : (isRetain(ops[1]) ? ops[0] : null);
    case 3:
      if (isRetain(ops[0]) && isRetain(ops[2])) { return ops[1]; }
    }
    return null;
  }
```

1. 配列に一つだけの操作が含まれる場合
その操作を返す

2. 配列に２つの操作が含まれる場合
どちらか片方が保持なら、もう一方を返す
両方とも保持でないなら、null

3. 配列に３つの操作が含まれる場合
最初と最後が保持なら、真ん中の操作を返す
それ以外はnull


###### getStartIndex

```javascript
  function getStartIndex (operation) {
    if (isRetain(operation.ops[0])) { return operation.ops[0]; }
    return 0;
  }
```
最初の操作が保持なら、保持数を返す
それ以外は0

###### shouldBeComposedWith

Composeするべきが判定する
Undoでひとつ前の操作を打ち消すため

```javascript
  TextOperation.prototype.shouldBeComposedWith = function (other) {
    if (this.isNoop() || other.isNoop()) { return true; }

    var startA = getStartIndex(this), startB = getStartIndex(other);
    var simpleA = getSimpleOp(this), simpleB = getSimpleOp(other);
    if (!simpleA || !simpleB) { return false; }

    if (isInsert(simpleA) && isInsert(simpleB)) {
      return startA + simpleA.length === startB;
    }

    if (isDelete(simpleA) && isDelete(simpleB)) {
      // there are two possibilities to delete: with backspace and with the
      // delete key.
      return (startB - simpleB === startA) || startA === startB;
    }

    return false;
  };
```

どちらか一方が、Simpleな操作でないならfalse

両方が挿入なら、Aの挿入後のIndexと、BのstartIndexが一致するかどうか

両方が削除なら、Bの削除後の位置とAのstartIndexが等しい、
または、AとBのstartIndexが等しい

###### shouldBeComposedWithInverted
```javascript
  TextOperation.prototype.shouldBeComposedWithInverted = function (other) {
    if (this.isNoop() || other.isNoop()) { return true; }

    var startA = getStartIndex(this), startB = getStartIndex(other);
    var simpleA = getSimpleOp(this), simpleB = getSimpleOp(other);
    if (!simpleA || !simpleB) { return false; }

    if (isInsert(simpleA) && isInsert(simpleB)) {
      return startA + simpleA.length === startB || startA === startB;
    }

    if (isDelete(simpleA) && isDelete(simpleB)) {
      return startB - simpleB === startA;
    }

    return false;
  };
```

どちらか一方が、Simpleな操作でないならfalse

両方が挿入なら、Aの挿入後のIndexと、BのstartIndexが一致するかどうか、
または、AとBのstartIndexが等しい

両方が削除なら、Bの削除後の位置とAのstartIndexが等しい


##### Transform
同時に発生したOperationを一つにまとめる

###### op1が挿入

```javascript
      if (isInsert(op1)) {
        operation1prime.insert(op1);
        operation2prime.retain(op1.length);
        op1 = ops1[i1++];
        continue;
      }
```
op1pにop1を挿入
op2pのカーソルをop1分進める
op1は一つ進む

###### op2が挿入

```javascript

      if (isInsert(op2)) {
        operation1prime.retain(op2.length);
        operation2prime.insert(op2);
        op2 = ops2[i2++];
        continue;
      }
```

op1pのカーソルをop2分進める
op2pにop2を挿入
op2は一つ進む


##### op1とop2が**保持**

```javascript
      var minl;
      if (isRetain(op1) && isRetain(op2)) {
        // Simple case: retain/retain
        if (op1 > op2) {
          minl = op2;
          op1 = op1 - op2;
          op2 = ops2[i2++];
        } else if (op1 === op2) {
          minl = op2;
          op1 = ops1[i1++];
          op2 = ops2[i2++];
        } else {
          minl = op1;
          op2 = op2 - op1;
          op1 = ops1[i1++];
        }
        operation1prime.retain(minl);
        operation2prime.retain(minl);
      }
```

###### op1 > op2
minl = op2
op1をop2分減らす
op2を一つ進める

###### op1 == op2
minl = op2
op1を一つ進める
op2を一つ進める

###### op1 < op2
minl = op1;
op2をop1分減らす
op1を一つ進める

operation1primeをminl分保持
operation2primeをminl分保持

##### op1とop2が**削除**

```javascript
      } else if (isDelete(op1) && isDelete(op2)) {
        // Both operations delete the same string at the same position. We don't
        // need to produce any operations, we just skip over the delete ops and
        // handle the case that one operation deletes more than the other.
        if (-op1 > -op2) {
          op1 = op1 - op2;
          op2 = ops2[i2++];
        } else if (op1 === op2) {
          op1 = ops1[i1++];
          op2 = ops2[i2++];
        } else {
          op2 = op2 - op1;
          op1 = ops1[i1++];
        }
      // next two cases: delete/retain and retain/delete
      }
```

###### op1の削除数 > op2の削除数
op1の削除数を、op2分減らす
op2を一つ進める

###### op1の削除数 = op2の削除数
op1を一つ進める
op2を一つ進める

###### op1の削除数 < op2の削除数
op2の削除数を、op1分減らす
op1を一つ進める


##### op1が**削除**、op2が**保持**
```javascript
      } else if (isDelete(op1) && isRetain(op2)) {
        if (-op1 > op2) {
          minl = op2;
          op1 = op1 + op2;
          op2 = ops2[i2++];
        } else if (-op1 === op2) {
          minl = op2;
          op1 = ops1[i1++];
          op2 = ops2[i2++];
        } else {
          minl = -op1;
          op2 = op2 + op1;
          op1 = ops1[i1++];
        }
        operation1prime['delete'](minl);
      }
```

###### op1の削除数 > op2の保持数
minl = op2の保持数
op1の削除数を、op2の保持数だけ減らす
op2をひとつ進める

###### op1の削除数 = op2の保持数
minl = op2の保持数
op1をひとつ進める
op2をひとつ進める

###### op1の削除数 < op2の保持数
minl = op1の削除数
op2の保持数を、op1の削除数だけ減らす
op1をひとつ進める


operation1primeをminl分削除


##### op1が**保持**、op2が削除

```javascript
      } else if (isRetain(op1) && isDelete(op2)) {
        if (op1 > -op2) {
          minl = -op2;
          op1 = op1 + op2;
          op2 = ops2[i2++];
        } else if (op1 === -op2) {
          minl = op1;
          op1 = ops1[i1++];
          op2 = ops2[i2++];
        } else {
          minl = op1;
          op2 = op2 + op1;
          op1 = ops1[i1++];
        }
        operation2prime['delete'](minl);
      } 
```

###### op1の保持数 > op2の削除数
minl = op2の削除数
op1の保持数を、op2の削除数だけ減らす
op2を一つ進める

###### op1の保持数 = op2の削除数
minl = op1の保持数
op1を一つ進める
op2を一つ進める

###### op1の保持数 < op2の削除数
minl = op1の保持数
op2の削除数をop1の保持数だけ減らす
op1を一つ進める

operation2primeをminl分削除

