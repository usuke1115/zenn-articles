---
title: "再帰関数の実装ガイド"
emoji: "🔁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# 対象読者
- 再帰関数が何かなんとなくわかる
- 階乗・フィボナッチ数などありきたりな再帰関数の実装は理解できる
- 実践的なものになると途端に実装できなくなる

# 編集前記
わけあって `Erlang` という関数型言語を学んでいます。

一般に関数型言語では、 `for` 文や `while` 文というものは存在しません。

その代わりに再帰関数を使って `for` 文や `while` 文で書ける処理を実装します。

```python:sample.py
arr = [1, 2, 3]
for i in range(n):
    print(arr[i])
```

上のような簡単な `Python` コードも、関数型言語だと `for` 文が使えないため、再帰関数で
実装するしかありません。 

以降は関数型言語から学んだ再帰関数の実装方法を共有します。

# 再帰関数の実装
```python:sample.py
arr = [1, 2, 3]
for i in range(n):
    print(arr[i])
```

上のコードを再帰関数で書くと
```python:recursive_for.py
def func_for(_list):
    if len(_list) == 0:
        print("PROCESS DONE")
        return
    print(_list[0])
    func_for(_list[1:])

func_for([1, 2, 3])
# ---- 出力 -----
# 1
# 2
# 3
# PROCESS DONE
```
となります。
