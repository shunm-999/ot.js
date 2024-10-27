# Selection
テキストエディタ内の選択状態を管理するための Selection と Range クラスの定義です。

SelectionはRangeの配列を持つ

## Rangeクラス

```javascript
  function Range (anchor, head) {
    this.anchor = anchor;
    this.head = head;
  }
```

anchor: 選択範囲の開始位置を示すインデックス。
head: 選択範囲のカーソルを示すインデックス。

anchor と head が等しい場合、これはカーソル位置として扱われます。

### transform

```javascript
  Range.prototype.transform = function (other) {
    function transformIndex (index) {
      var newIndex = index;
      var ops = other.ops;
      for (var i = 0, l = other.ops.length; i < l; i++) {
        if (TextOperation.isRetain(ops[i])) {
          index -= ops[i];
        } else if (TextOperation.isInsert(ops[i])) {
          newIndex += ops[i].length;
        } else {
          newIndex -= Math.min(index, -ops[i]);
          index += ops[i];
        }
        if (index < 0) { break; }
      }
      return newIndex;
    }

    var newAnchor = transformIndex(this.anchor);
    if (this.anchor === this.head) {
      return new Range(newAnchor, newAnchor);
    }
    return new Range(newAnchor, transformIndex(this.head));
  };
```

##### opは**保持**の場合
保持分、indexを減らす

##### opは**挿入**の場合
挿入数分、newIndexを増やす

##### opは**削除**の場合
indexと、削除数で、少ないほうをnewIndexから引く
indexを、削除数分減らす


transformIndexを、anchorと、headに適用する

#### createCursor
```javascript
  Selection.createCursor = function (position) {
    return new Selection([new Range(position, position)]);
  };
```

カーソルから、Selectionを作成

#### somethingSelected

```javascript
  Selection.prototype.somethingSelected = function () {
    for (var i = 0; i < this.ranges.length; i++) {
      if (!this.ranges[i].isEmpty()) { return true; }
    }
    return false;
  };
```

何かを選択している

#### Selection.transform

```javascript
  Selection.prototype.transform = function (other) {
    for (var i = 0, newRanges = []; i < this.ranges.length; i++) {
      newRanges[i] = this.ranges[i].transform(other);
    }
    return new Selection(newRanges);
  };
```

rangeに操作を順番に適用

