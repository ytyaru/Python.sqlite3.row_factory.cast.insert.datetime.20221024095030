insert時にboolとdatetime型をSQLite3の型にキャストする【Python】

　UTC標準時の`yyyy-MM-dd HH:mm:ss`形式に。

<!-- more -->

# ブツ

* [リポジトリ][]

[リポジトリ]:https://github.com/ytyaru/Python.sqlite3.row_factory.cast.insert.datetime.20221024095030
[DEMO]:https://ytyaru.github.io/Python.sqlite3.row_factory.cast.insert.datetime.20221024095030/

## 実行

```sh
NAME='Python.sqlite3.row_factory.cast.insert.datetime.20221024095030'
git clone https://github.com/ytyaru/$NAME
cd $NAME/src
./test-castpy.py
./test-ntlite.py
```

# コード抜粋

　[前回][]と大差ない。日付のキャストを実装したのと、キャストのテストコードを書いた。

[前回]:https://github.com/ytyaru/Python.sqlite3.row_factory.cast.insert.20221018161507/

## キャスト

　Pythonの`bool`型と`datetime`型をSQLの型に変換する。

Python|SQLite3
------|-------
`bool`|`INTEGER`の`0`/`1`
`datetime`|UTC標準時の`yyyy-MM-dd HH:mm:ss`。もし渡された日時がタイムゾーンのないネイティブな日付ならローカル日時として解釈する。

```python
class CastPy:
    @classmethod
    def to_sql(cls, v):
        if isinstance(v, bool): return 1 if v else 0
        elif isinstance(v, datetime): return f"{AwareDateTime.to_utc(AwareDateTime.if_native_to_local(v)):%Y-%m-%d %H:%M:%S}"
        else: return v
    @classmethod
    def to_sql_by_row(cls, row):
        if isinstance(row, tuple): return tuple([cls.to_sql(col) for col in row])
        else: return row
    @classmethod
    def to_sql_by_rows(cls, rows):
        if isinstance(rows, list): return [cls.to_sql_by_row(row) for row in rows]
        else: return rows
```

　`to_sql()`がキャスト用メソッド。それ以外は行や複数行の単位でそれを呼び出しているだけ。

## `insert`

　こっちは前回と同じ。insert文をなるだけ簡単に発行するようにした。その中でキャストも一緒にやる。

```python
class NtLite:
    def _cast_exec(self, sql, params): return self.exec(sql, CastPy.to_sql_by_row(params))
    def _cast_execm(self, sql, params): return self.execm(sql, CastPy.to_sql_by_rows(params))
    def _insert_sql(self, table_name, params): return f"insert into {table_name} values ({','.join('?' * len(params))})"
    def insert(self, table_name, params): return self._cast_exec(self._insert_sql(table_name, params), params)
    def inserts(self, table_name, params): return self._cast_execm(self._insert_sql(table_name, params), params)
```

　使うときは以下。

```python
NtLite().insert(テーブル名, 挿入する列のデータをタプルで渡す)
```

　これで以下のようなSQL文を作ってくれる。preperd statement。データをクォートで囲むか否かを気にせずに済む。`?`を列数分だけ作らずに済む。内部ではPythonの`sqlite3.execute(sql, params)`を呼び出している。

```sql
insert into テーブル名 values(?,?,?...挿入するレコードの列数だけ?をつける);
```

　もし複数行のデータをまとめて挿入したければ複数形の`inserts()`を使う。

```python
NtLite().inserts(テーブル名, [挿入する列のタプル, 挿入する列のタプル, ...])
```

　内部ではPythonの`sqlite3.executemany(sql, params)`を呼び出している。以下のようなSQL文を作る。

```sql
insert into テーブル名 values(?,?,?...), (?,?,?...), ...;
```

# 所感

　こんな感じで他にも`update`, `delete`, `select`, `create table`文もラップしたいのだが`insert`より難しそう。

* どの列を使うか
* `while`文などの条件式の演算子をどう表現するか（`col <= 1`）
* ほかにも`group by`, `order by`, 副問合せ, テーブル結合, SQL関数, 句(`like`, `glob`, `with`, `limit`, `offset`等)など、明らかにSQL文を直接書かないと表現できないものがたくさんある

　どこかで断念すると思うけど、勉強がてらやってみる予定。どんどん本筋から遠ざかってゆく。

