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






