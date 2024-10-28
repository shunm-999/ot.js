# Undo Manager
操作の取り消し（undo）およびやり直し（redo）を管理する
操作の履歴を管理し、直前の操作を逆向きに実行して「取り消す」ことや、
「やり直す」ためのメソッドを提供します。

## 状態
現在の状態を示す3つの定数が定義されています。

**NORMAL_STATE**: 通常の状態
**UNDOING_STATE**: 取り消し動作中
**REDOING_STATE**: やり直し動作中

## Constructor

```javascript
  function UndoManager (maxItems) {
    this.maxItems  = maxItems || 50;
    this.state = NORMAL_STATE;
    this.dontCompose = false;
    this.undoStack = [];
    this.redoStack = [];
  }
```
maxItems: 履歴として保持する最大操作数（デフォルトは50）。
state: 現在の状態。
dontCompose: 操作を合成するかを判定するためのフラグ。
undoStack: 取り消し操作の履歴を保持するスタック。
redoStack: やり直し操作の履歴を保持するスタック。

## add

操作を undoStack または redoStack に追加します。

```javascript
  UndoManager.prototype.add = function (operation, compose) {
    if (this.state === UNDOING_STATE) {
      this.redoStack.push(operation);
      this.dontCompose = true;
    } else if (this.state === REDOING_STATE) {
      this.undoStack.push(operation);
      this.dontCompose = true;
    } else {
      var undoStack = this.undoStack;
      if (!this.dontCompose && compose && undoStack.length > 0) {
        undoStack.push(operation.compose(undoStack.pop()));
      } else {
        undoStack.push(operation);
        if (undoStack.length > this.maxItems) { undoStack.shift(); }
      }
      this.dontCompose = false;
      this.redoStack = [];
    }
  };
```

### UNDOING_STATEのとき
redoStackの操作をpush
dontComposeをtrueに

### REDOING_STATEのとき
undoStackに操作をpush
dontComposeをtrueに

### NORMAL_STATEのとき
1. dontComposeがfalse、引数のcomposeがtrue、undoStackに要素があるとき

undoStackの最後の要素と、操作を合成してpush

2. 1以外のとき

undoStackにpush
undoStackがmaxItemsより多くなったら、先頭の要素を削除
dontComposeをfalse
redoStackをクリア

### transformStack
```javascript
  function transformStack (stack, operation) {
    var newStack = [];
    var Operation = operation.constructor;
    for (var i = stack.length - 1; i >= 0; i--) {
      var pair = Operation.transform(stack[i], operation);
      if (typeof pair[0].isNoop !== 'function' || !pair[0].isNoop()) {
        newStack.push(pair[0]);
      }
      operation = pair[1];
    }
    return newStack.reverse();
  }
```

stackの末尾から、transformを適用させて、新しい配列に格納していく
順番が逆なっているので、最後にreverseを呼んでいる

### transform

undoStackと、redoStackに、transformStackを適用させる


### performUndo
```javascript
  UndoManager.prototype.performUndo = function (fn) {
    this.state = UNDOING_STATE;
    if (this.undoStack.length === 0) { throw new Error("undo not possible"); }
    fn(this.undoStack.pop());
    this.state = NORMAL_STATE;
};
```
stateを**UNDOING_STATE**に変更
undoStackの最後の要素を、ラムダ式に渡す
最後に、stateを**NORMAL_STATE**に変更


### performRedo
```javascript
  UndoManager.prototype.performRedo = function (fn) {
    this.state = REDOING_STATE;
    if (this.redoStack.length === 0) { throw new Error("redo not possible"); }
    fn(this.redoStack.pop());
    this.state = NORMAL_STATE;
  };
```
performUndoの反対


