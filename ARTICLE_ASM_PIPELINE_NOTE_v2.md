---
author:       bunjin
platform:     note
published:    2025-03-24
title:        AIにアセンブラを学習させるデータセットを作った——Rust対応・DWARF解析・GPL除外まで全部やった話
description:  GitHub APIでMITリポジトリを収集し、GCCとrustcで4段階コンパイル、objdumpでアセンブラ抽出。DWARF解析によるsize_bytes取得・Rust対応・GPL自動除外・多言語集約の全工程をG.stateが記録する。
keywords:     アセンブラ, データセット, HuggingFace, C言語, Rust, 逆コンパイル, DWARF, GCC, Claude Code, AI学習データ, x86-64, MIT ライセンス, GPL除外, 多言語集約
site:         https://gstate.dev
tags:         アセンブラ, データセット, HuggingFace, AI, 機械学習, C言語, Rust, 逆コンパイル, セキュリティ, オープンソース, ClaudeCode, x86-64
---

# AIにアセンブラを学習させるデータセットを作った

**「アセンブラを理解するAI」を作るためのデータセットをゼロから構築した記録だ。GitHub APIでコードを自動収集し、GCCとrustcで4段階コンパイル、objdumpでアセンブラを取り出す。詰まった箇所と解決策を中心に書く。**

**▶ プロジェクトの詳細・続編 → [gstate.dev](https://gstate.dev/)**

---

## なぜ作るのか

PythonやJavaScriptのAI学習データはすでに世界中にある。しかしアセンブラの対訳データセットはHuggingFaceでも圧倒的に不足している。

マルウェア解析・脆弱性研究・リバースエンジニアリングの現場で「アセンブラを読んで意味を理解するAI」への需要は急速に高まっている。データがないから自分で作る——それだけの理由だ。

---

## 作るものの概要

C言語またはRustのソースコードと、それをコンパイルして得たアセンブラをペアにして保存する。

```json
{
  "asm_code":  "lea eax, [rdi+rsi]\n ret",
  "arch":      "x86-64",
  "opt_level": 3,
  "func_meta": {
    "name": "add",
    "return_type": "int",
    "return_size_bytes": 4,
    "params": [
      {"name": "a", "type_name": "int", "size_bytes": 4},
      {"name": "b", "type_name": "int", "size_bytes": 4}
    ],
    "dwarf_parsed": true
  },
  "sources": [
    {"lang": "c",    "code": "int add(int a, int b){ return a+b; }", "opt_flag": "-O3"},
    {"lang": "rust", "code": "fn add(a: i32, b: i32) -> i32 { a + b }", "opt_flag": "3"}
  ]
}
```

同じアセンブラを生成するCとRustのコードを1レコードに集約するのが最大の特徴だ。

---

## 詰まった5つの問題と解決策

### ① Rustが全件コンパイルエラーになる

`rustc` に `--crate-type lib` を付けていなかったのが原因だ。Rustはデフォルトで実行ファイルを生成しようとするため `main` 関数が必須になる。ライブラリとしてコンパイルするオプション1行で解決した。

```
# ❌  → error[E0601]: main function not found
rustc -C opt-level=0 --emit obj src.rs

# ✅  → コンパイル成功
rustc -C opt-level=0 --crate-type lib --emit obj src.rs
```

### ② Rustの関数が全件除外される

`objdump` の出力でRustの関数名は `_ZN4core3cmp...` のようにマングリングされる。スクリプトがこのパターンを「コンパイラ内部シンボル」として除外していたため、Rustのレコードが0件になっていた。

C言語の場合はこのパターンを除外して正しいが、Rust用にはデマングル処理を分岐する必要がある。`rustfilt` があればそれを使い、なければ正規表現で `_ZN4core3cmp3max17h...E → core::cmp::max` のように簡易変換する。

### ③ Rustのソースコード対応率が0%

デマングル後の関数名（例：`core::cmp::impls::_<impl Ord for usize>::cmp`）は元のRustソースの `fn` 名と一致しない。3段階フォールバックで対応した。

まず完全一致を試みる。ダメなら末尾セグメント（`cmp` など）で再検索する。それでもダメならファイル全体をコンテキストとして保存し `code_match: "full_file"` と記録する。この仕組みでRustのソースコード対応率は **0% → 98.8%** まで改善した。

### ④ GPLリポジトリがデータに混入する

`TheAlgorithms/C` という有名リポジトリはMITに見えて実際は **GPL-3.0** だった。GPLコードを含むデータセットを公開するとライセンス汚染のリスクがある。

GitHub APIのレスポンスでライセンスの識別子を動的チェックし、GPL系（`GPL-2.0`・`GPL-3.0`・`LGPL`・`AGPL`）を自動除外するフィルタを入れた。代替として `fragglet/c-algorithms`（ISC・MIT互換）・`swenson/sort`（MIT）・`afiskon/c-algorithms`（MIT）を使っている。

### ⑤ optimized_change が常に不明になる

`-O0` と `-O3` で同じアセンブラを生成する関数の場合、SHA256ハッシュが同一なので重複除去で1件に統合されてしまう。比較対象が消えるため `optimized_change` が永久に `null` になっていた。

重複除去のキーを `asm_hash` 単体から `(asm_hash, opt_level)` の複合キーに変更した。これにより `-O0` と `-O3` は常に別レコードとして保持され、比較が正しく行われる。

---

## DWARFでsize_bytesを取得する

アセンブラの命令選択は型のサイズに依存する。`int`（4バイト）と `long`（8バイト）では使われるレジスタが変わるため、サイズ情報はAIにとって重要なヒントになる。

GCCに `-g`、rustcに `-C debuginfo=2` を付けることでDWARFデバッグ情報を付与できる。生成されたオブジェクトファイルを `pyelftools` で解析すると、引数の型名とバイトサイズが取得できる。`pyelftools` が未インストールの場合は `objdump --dwarf=info` のテキスト出力を正規表現でパースするフォールバックがある。

```json
"params": [
  {"name": "a", "type_name": "struct Activity *", "size_bytes": 8, "is_pointer": true},
  {"name": "n", "type_name": "int",               "size_bytes": 4, "is_pointer": false}
]
```

---

## 現在の品質指標

| 指標 | 値 |
|---|---|
| 総レコード数 | 811件 |
| Rustソースコード対応率 | **98.8%** |
| DWARF解析成功率 | **75.5%** |
| 全ソースコード空欄 | **1.8%** |
| 最適化で変化あり | 64.1% |
| Rustマングリング残存 | **0件** |

主要な品質指標は実用レベルに達している。次はHuggingFaceへの公開とデータカードの整備を進める。

---

## 多言語集約の仕組み

同一アルゴリズムのC版とRust版が同じアセンブラを生成する場合、`sources` リストに両言語を追加して1レコードに統合する。

アセンブラが異なる場合も、関数名を正規化（`bubbleSort → bubble_sort`、`binarySearch → binary_search` など）して `normalized_func_name` で照合し、同じアルゴリズムの実装として集約する。これにより「C言語とRustで同じアルゴリズムを書いても、最適化すると同じアセンブラになる」という知識をAIが一度に学べる構造になっている。

現時点で `TheAlgorithms/Rust`（MIT）と `fragglet/c-algorithms`（ISC・MIT互換）をペアとして収集しており、`backtrack` などの関数で多言語集約が発生している。今後対象リポジトリを増やすことで集約件数を伸ばす予定だ。

---

## まとめ

- **Rust:** `--crate-type lib` を付けないと全件コンパイルエラー
- **マングリング:** `_ZN...` はフィルタで除外せずデマングルして使う
- **ソース対応:** 3段階フォールバック（完全一致 → 短縮名 → ファイル全体）で98.8%達成
- **GPL:** APIレスポンスで動的チェックして自動除外する（`TheAlgorithms/C` は GPL-3.0）
- **最適化比較:** 重複除去キーを `(asm_hash, opt_level)` の複合キーにする
- **size_bytes:** `-g` フラグでDWARF付与、pyelftoolsで型情報を解析する
- **多言語集約:** 正規化関数名で言語をまたいでレコードを統合する

---

**▶ スクリプト全文・HuggingFace公開状況 → [gstate.dev](https://gstate.dev/)**

G.stateでは、AIとレガシー技術をつなぐプロジェクトを継続的に公開している。

---

<!--
==========================================================
SEO / AIO / GEO 対応メタ情報
note投稿時はこのコメントブロックごと削除してください
==========================================================

【ページタイトル】
AIにアセンブラを学習させるデータセットを作った——Rust・DWARF・GPL除外まで | G.state

【メタディスクリプション（150字以内）】
GitHub APIでMITリポジトリを収集しGCCとrustcで4段階コンパイル、objdumpでアセンブラ抽出。DWARF解析によるsize_bytes取得・Rust対応・GPL自動除外・多言語集約の全工程をG.stateが記録。

【主要キーワード】
アセンブラ データセット 作り方
HuggingFace アセンブラ 公開
C言語 Rust アセンブラ 対訳
DWARF size_bytes 型情報
GCC 最適化 objdump x86-64
GPLライセンス 除外 MIT
rustc crate-type lib マングリング
SHA256 重複除去 複合キー

【AIO（AI検索）想定Q&A】
Q: アセンブラのAI学習データセットはどうやって作るか？
A: GitHub APIでMITライセンスのC/Rustリポジトリを収集し、GCC/rustcで-O0〜-O3の4段階コンパイル、objdumpでアセンブラを抽出してJSON/Parquet形式で保存する。

Q: rustcでオブジェクトファイルを生成する方法は？
A: --crate-type lib --emit objオプションを付ける。--crate-type libなしではmain関数が必要でエラーになる。

Q: Rustのマングリングシンボルをデマングルするには？
A: rustfiltコマンドにパイプするか、_ZN...形式を正規表現でパースして人間可読な形式に変換する。

Q: DWARFで引数のsize_bytesを取得するには？
A: GCCに-g、rustcに-C debuginfo=2を付けてコンパイルし、pyelftoolsでDW_TAG_formal_parameterのDW_AT_typeを辿って取得する。

Q: GPLリポジトリをデータセット収集から自動除外するには？
A: GitHub APIのlicense.spdx_idがGPL-2.0/GPL-3.0等に一致する場合にスキップする。TheAlgorithms/CはGPL-3.0なので要注意。

Q: optimized_changeが常にNullになる原因と対策は？
A: 重複除去キーがasm_hash単体だと-O0と-O3が同一ハッシュで統合される。(asm_hash, opt_level)の複合キーにすると比較対象が保持されて正確に付与できる。

【GEO対応ポイント】
・問題①〜⑤を「原因→解決策」の構造で明示（AIが引用しやすい）
・まとめをシンプルな箇条書きで整理（AI要約の対象になりやすい）
・gstate.devへの導線を冒頭と末尾の両方に設置

【推奨ハッシュタグ（note）】
#アセンブラ #データセット #AI #機械学習 #HuggingFace
#C言語 #Rust #逆コンパイル #セキュリティ #ClaudeCode
#オープンソース #低レイヤー #x86 #プログラミング

==========================================================
-->
