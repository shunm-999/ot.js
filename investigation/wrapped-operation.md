# Wrapped Operation

WrappedOperation は、
基本的な操作（operation）とそのメタデータ（meta）を一緒に保持するためのクラスです。
meta は操作に付随する追加情報
（たとえば、操作を行ったユーザーや操作の内容に関連するデータなど）を格納できます。

このコードは、`WrappedOperation` クラスを使って操作 (`operation`) とそのメタデータ (`meta`) をラップし、操作に付随する情報を管理するための仕組みです。以下に、各部分の説明と、このクラスが何を達成しようとしているのかを解説します。

### 1. `WrappedOperation` クラス
`WrappedOperation` は、基本的な操作（`operation`）とそのメタデータ（`meta`）を一緒に保持するためのクラスです。`meta` は操作に付随する追加情報（たとえば、操作を行ったユーザーや操作の内容に関連するデータなど）を格納できます。

- **プロパティ**
    - `this.wrapped`: 実際の操作を保持。
    - `this.meta`: メタデータを保持。

- **メソッド**
    - `apply`: `operation` を実行するためのメソッド。引数を操作オブジェクトの `apply` メソッドにそのまま渡し、ドキュメントに適用します。
    - `invert`: 現在の操作を反転（逆操作）します。反転した操作とメタデータを含む新しい `WrappedOperation` インスタンスを返します。
        - メタデータも反転できる場合には、メタデータ内の `invert` メソッドを呼び出します。

### 2. メタデータの操作（`composeMeta` と `transformMeta`）
- **`copy` 関数**  
  `copy(source, target)` は、`source` のすべてのプロパティを `target` にコピーするための補助関数です。

- **`composeMeta(a, b)`**  
  2つのメタデータを合成します。`a` の `compose` メソッドが存在する場合、それを呼び出してメタデータを合成し、そうでない場合は `a` と `b` のプロパティをすべて持つ新しいメタデータを作成して返します。

- **`transformMeta(meta, operation)`**  
  メタデータと操作を受け取り、メタデータ内に `transform` メソッドがあれば、それを実行して操作の影響を反映します。これにより、操作が変換されても関連するメタデータが正確に更新されます。

### 3. `compose` メソッド
- `WrappedOperation.prototype.compose(other)` は、2つの操作を合成（合併）して1つの新しい `WrappedOperation` を作成します。
- `this.wrapped.compose(other.wrapped)` を使って操作を合成し、メタデータは `composeMeta(this.meta, other.meta)` を使用して合成されます。

### 4. `transform` 関数
`WrappedOperation.transform(a, b)` は、2つの操作 `a` と `b` を互いに変換して順序に影響されないようにするための静的メソッドです。

- **手順**:
    1. `a.wrapped` と `b.wrapped` をそれぞれ `a.wrapped.constructor.transform` によって変換して、衝突しない形にします。
    2. 結果として得られる2つの新しい操作 `pair[0]` と `pair[1]` を、新しい `WrappedOperation` としてラップし、それぞれのメタデータも `transformMeta` を使って変換します。

### まとめ
- `WrappedOperation` は、操作とそのメタデータを一緒に管理し、メタデータを維持したまま操作を反転、合成、変換できるように設計されています。
- これにより、リアルタイム共同編集システムなどで、操作とその関連情報を管理しつつ、正確なドキュメントの同期を実現することが可能です。

### 環境に応じたエクスポート
- `ot.WrappedOperation` として、ブラウザやNode.jsで使用できるように設計されています。