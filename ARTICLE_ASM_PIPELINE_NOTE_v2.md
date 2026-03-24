---
author:       bunjin
platform:     note
published:    2025-03-24
title:        AIにアセンブラを学習させるデータセットをゼロから作った全記録——設計・実装・デバッグ・品質改善まで
description:  GitHub APIでMITライセンスのC・Rustリポジトリを収集しGCC/rustcでコンパイル、objdumpでアセンブラを抽出するパイプラインをゼロから構築。--crate-type lib問題・マングリング除外バグ・GPL混入・DWARF解析・多言語集約まで全工程を記録する。
keywords:     アセンブラ, データセット, HuggingFace, C言語, Rust, 逆コンパイル, DWARF, SHA256, GCC, 最適化, Claude Code, AI学習データ, x86-64, MITライセンス, 多言語集約, objdump, pyelftools
site:         https://gstate.dev
tags:         アセンブラ, データセット, HuggingFace, AI, 機械学習, C言語, Rust, 逆コンパイル, セキュリティ, オープンソース, ClaudeCode, x86-64, 低レイヤー
---

# AIにアセンブラを学習させるデータセットをゼロから作った全記録

**GitHub APIでコードを自動収集し、GCCとrustcで4段階コンパイル、objdumpでアセンブラを抽出、DWARFで型情報まで取得——「アセンブラを理解するAI」を作るためのデータセットをゼロから構築した全工程を記録する。設計の失敗・デバッグの迷走・品質改善の試行錯誤まで、包み隠さず書く。**

---

## なぜ作るのか

PythonやJavaScriptのコードはAIが大量に学習してきた。GitHub上のコードを集めれば良質なデータセットがすぐ手に入る。

しかしアセンブラは違う。HuggingFace上でアセンブラ関連のデータセットを探してみると、その少なさに驚く。世界中の研究者が使っているGhidraやIDA Proといった逆コンパイルツールは、アセンブラをC言語に変換する能力を持っているが、それを「AIに学習させる」ためのデータが圧倒的に不足している。

リバースエンジニアリング・マルウェア解析・脆弱性研究の現場では「アセンブラを読んで意味を理解するAI」への需要が急速に高まっている。マルウェア解析者は毎日何千件もの不審なバイナリを確認しなければならない。もしAIがアセンブラを見てC言語相当のコードに変換できれば、解析時間は劇的に短縮できる。

データがないから作る——それがこのプロジェクトの出発点だ。

このプロジェクトは [G.state（gstate.dev）](https://gstate.dev/) で進めており、Claude Codeを全面的に活用してパイプラインを構築した。

**▶ プロジェクトの詳細・スクリプト・続編 → [gstate.dev](https://gstate.dev/)**

---

## 作るものの設計：なぜ「言語中立」にするのか

最初に陥りがちな設計ミスがある。「C言語のデータセットを作る」と決めてこう設計してしまうことだ。

```json
{
  "c_code":      "int add(int a, int b) { return a + b; }",
  "asm_code":    "lea eax, [rdi+rsi]\n ret",
  "optimization": "-O3",
  "arch":        "x86-64"
}
```

後からRustも追加しようとすると、こうなる。

```json
{
  "rust_code":   "fn add(a: i32, b: i32) -> i32 { a + b }",
  "asm_code":    "lea eax, [rdi+rsi]\n ret",
  "optimization": "opt-level=3",
  "arch":        "x86-64"
}
```

問題は明確だ。同じアセンブラ（`lea eax, [rdi+rsi] / ret`）なのに、C版とRust版が**別々のデータとして存在する**。AIは「CとRustは同じアセンブラを生成することがある」という知識を学べない。将来さらに言語を追加するたびに全データを作り直す必要も出てくる。

だから最初から**言語中立なスキーマ**で設計する。

```json
{
  "asm_code":  "lea eax, [rdi+rsi]\n ret",
  "arch":      "x86-64",
  "opt_level": 3,
  "asm_hash":  "9a4bfb8c1...",
  "func_meta": {
    "name":               "add",
    "insn_count":         2,
    "optimized_change":   true,
    "calling_convention": "System V AMD64",
    "return_type":        "int",
    "return_size_bytes":  4,
    "params": [
      {"name": "a", "type_name": "int", "size_bytes": 4, "is_pointer": false},
      {"name": "b", "type_name": "int", "size_bytes": 4, "is_pointer": false}
    ],
    "dwarf_parsed": true
  },
  "sources": [
    {
      "lang": "c",    "compiler": "gcc",   "compiler_ver": "13.2",
      "code": "int add(int a, int b){ return a+b; }",
      "opt_flag": "-O3", "code_match": "exact"
    },
    {
      "lang": "rust", "compiler": "rustc", "compiler_ver": "1.75",
      "code": "fn add(a: i32, b: i32) -> i32 { a + b }",
      "opt_flag": "3",  "code_match": "short_name"
    }
  ],
  "normalized_func_name": "add"
}
```

アセンブラを主キーとして、同じアセンブラを生成するC・Rustのコードを `sources` リストに集約する。後から言語を追加しても既存データの構造は変わらない。

### メタ情報として残しておくと強いもの

**size_bytes（型のバイトサイズ）：** `int` が4バイト、`long` が8バイト。これを記録しておくと「なぜ `mov eax`（32bit）が使われるのか」をAIが理解しやすくなる。`mov eax` が使われるのは引数が4バイトだからだ。

**calling_convention（呼び出し規約）：** LinuxとWindowsでは引数の渡し方が違う（System V AMD64 vs Microsoft x64）。記録しておかないと、同じCコードでも環境によってアセンブラが違う理由が後からわからなくなる。

**optimized_change（最適化変化フラグ）：** `-O0` と `-O3` で同じアセンブラが出るかどうか。`false` の場合は「最適化の余地がないシンプルな関数」として難易度ラベルに使える。

**normalized_func_name（正規化関数名）：** `bubbleSort`（C）と `bubble_sort`（Rust）を `bubble_sort` に統一して多言語集約の照合キーにする。

---

## パイプラインの全体像

```
Phase 1: GitHub API
  ↓ MITライセンスリポジトリを検索・収集
  ↓ GPL系（GPL-2.0/3.0/LGPL/AGPL）は自動除外
  ↓ BLOCKED_REPOS（TheAlgorithms/C等）も除外

Phase 2: コンパイル & ディスアセンブル
  ↓ C言語: GCC -O0/-O1/-O2/-O3 + -g（DWARF用）
  ↓ Rust: rustc --crate-type lib opt-level=0〜3 -C debuginfo=2
  ↓ objdump -d -M intel --no-show-raw-insn
  ↓ 関数単位に分割（5〜500命令でフィルタ）

Phase 2.5: 多言語集約
  ↓ 関数名を正規化（bubbleSort → bubble_sort）
  ↓ (normalized_func_name, opt_level) が同じなら sources に統合

Phase 3: DWARF解析
  ↓ pyelftools でELFのDW_TAG_subprogram を解析
  ↓ 引数の型名・size_bytes を取得
  ↓ pyelftools 未インストール時は objdump --dwarf=info でフォールバック

Phase 4: optimized_change フラグ付与
  ↓ (source_repo, normalized_func_name, lang) をキーに
  ↓ O0ハッシュ と O3ハッシュ を比較
  ↓ 重複除去キーは (asm_hash, opt_level) の複合キー

Phase 5: JSON / Parquet出力
  ↓ JSON（人間が読みやすい）
  ↓ Parquet snappy圧縮（HuggingFace向け）
```

---

## 構築の記録：詰まったポイントと解決策

ここからが本題だ。スムーズにはいかなかった。ゼロから作ると必ずハマる箇所がある。実際に起きた問題と解決策を順番に記録する。

### 問題① Rustレコードが0件——`--crate-type lib` の欠落

最初にC・Rust両方で動かしてみると、こんな結果になった。

```
✅ データセット生成完了
   総レコード数 : 303
   [c     ] ソースあり: 301 件
   [rust  ] ソースあり: 0 件    ← なぜ？
```

Rustのコンパイルを単体でテストしてみた。

```
rustc returncode: 1
stderr: error[E0601]: `main` function not found in crate `src`
```

原因は単純だった。RustはデフォルトでBinaryクレート（実行ファイル）を生成しようとするため、`fn main()` が必須になる。ライブラリ形式でコンパイルするには `--crate-type lib` が必要だった。

**修正前：**
```bash
rustc -C opt-level=0 --emit obj src.rs
→ error: main function not found
```

**修正後：**
```bash
rustc -C opt-level=0 --crate-type lib --emit obj src.rs
→ ✅ コンパイル成功
```

一行追加するだけで解決した。しかしこれは序章に過ぎなかった。

---

### 問題② Rustの関数が全件フィルタで除外される

`--crate-type lib` を追加して再実行してもRustは0件のままだった。次はデバッグ用スクリプトで実際のobjdump出力を確認した。

```bash
returncode: 0
検出関数数: 286
  _ZN100_$LT$alloc..boxed..Box$LT$dyn...$GT$4from17hd51b770aec3f70eeE: 11命令
  _ZN102_$LT$core..iter..adapters..map..Map$LT$I$C$F$GT$...4fold17h89ae5eee13a14664E: 9命令
```

コンパイルは成功していた。関数も286件検出されていた。しかし**すべての関数名が `_ZN...` 形式のマングリングシンボル**だった。

スクリプトの `parse_functions_from_objdump` 関数を見ると、こうなっていた。

```python
# 問題のあったコード
if any(name.startswith(p) for p in ("_ZN", "__", ".L", "GCC_")):
    current = None  # _ZN で始まるRustの全関数がここで除外される
```

`_ZN` はRustのマングリング形式そのものだ。C言語でも内部シンボルとして使われることがあるため除外リストに入れていたが、Rustの全関数がここで消えていた。

**修正：lang引数を追加してRustの場合はデマングルして保持する**

```python
def parse_functions_from_objdump(objdump_out: str, lang: str = "c") -> dict[str, str]:
    for line in objdump_out.splitlines():
        m = FUNC_HEADER.match(line)
        if m:
            raw_name = m.group(1)
            if lang == "rust":
                if raw_name.startswith("_ZN"):
                    # マングリングをデマングルして使用
                    name = _demangle_rust_simple(raw_name)
                elif any(raw_name.startswith(p) for p in ("__", ".L", "GCC_")):
                    current = None
                    continue
                else:
                    name = raw_name
            else:
                # C/Python: 従来どおり内部シンボルを除外
                if any(raw_name.startswith(p) for p in ("_ZN", "__", ".L", "GCC_")):
                    current = None
                    continue
                name = raw_name
```

簡易デマングルは `rustfilt` を優先し、未インストール時は正規表現で変換する。

```python
def _demangle_rust_simple(name: str) -> str:
    # rustfilt が使えれば優先
    try:
        r = subprocess.run(["rustfilt"], input=name, capture_output=True, text=True)
        if r.returncode == 0 and r.stdout.strip() != name:
            return r.stdout.strip()
    except FileNotFoundError:
        pass
    # 正規表現で簡易変換
    # _ZN4core3cmp3max17h...E → core::cmp::max
    m = re.match(r"^_ZN(.+?)(?:17h[0-9a-f]+E|E)$", name)
    if not m:
        return name
    # 長さプレフィクス付きセグメント列をパース
    ...
```

これでRustのレコード数が **0件 → 897件** に急増した。

---

### 問題③ Rustのソースコード対応率が0%

件数は増えたが、今度は別の問題が発覚した。

```
【Rust code_match 内訳】
  none  : 897件（100%）← 全件ソースコードが空欄
```

デマングル後の関数名を見ると、こうなっていた。

```
_<alloc..boxed..Box<dyn core..error..Error> as core..convert::From<E>>::from
_<core..iter..adapters..map::Map<I,F> as core..iter::traits::iterator::Iterator>::fold
```

これはRustのトレイト実装関数の名前だ。元のRustソースで `fn from(` や `fn fold(` は存在するが、`extract_functions_rust` が `fn` パターンで正規表現マッチしても、デマングル後の長いトレイト名とは一致しない。

**3段階フォールバックで解決した。**

```python
# ① func_name でそのまま一致（exact）
code = src_functions.get(func_name, "")
code_match = "exact" if code else "none"

if not code and lang == "rust":
    # ② 末尾セグメントで再検索（short_name）
    # "core::cmp::max" → "max" で再検索
    short_name = func_name.split("::")[-1].split("<")[0].strip()
    code = src_functions.get(short_name, "")
    if code:
        code_match = "short_name"

    if not code:
        # ③ ファイル全体をコンテキストとして使用（full_file）
        # トレイト実装等で fn名が一致しない場合のフォールバック
        code = src_functions.get("_FULL_FILE_", "")
        if code:
            code_match = "full_file"
```

また `extract_functions_rust` も強化した。`pub fn` だけでなく `async fn` / `unsafe fn` / `impl` ブロック内の `fn` も抽出し、さらにファイル全体を `_FULL_FILE_` キーで保存するようにした。

```python
def extract_functions_rust(source: str) -> dict[str, str]:
    functions: dict[str, str] = {}
    fn_pattern = re.compile(
        r"^\s*(?:pub(?:\s*\([^)]*\))?\s+)?(?:async\s+)?(?:unsafe\s+)?fn\s+(\w+)\s*[<(]"
    )
    # ... fn抽出処理 ...

    # フォールバック用にファイル全体も保存
    if len(source.splitlines()) <= MAX_SRC_LINES * 3:
        functions["_FULL_FILE_"] = source

    return functions
```

結果、Rustのソースコード対応率は **0% → 98.8%** に改善した。

| 種別 | 件数 | 意味 |
|---|---|---|
| exact | — | 関数名が完全一致 |
| short_name | 167件 | 末尾セグメントで一致 |
| full_file | 348件 | ファイル全体をコンテキストとして使用 |
| none | 6件 | 取得不可（1.2%） |

---

### 問題④ GPL-3.0リポジトリの混入

多言語集約を実現するため、`TheAlgorithms/C`（C言語版）と `TheAlgorithms/Rust`（Rust版）をペアで収集する設計にした。同じアルゴリズムを両言語で実装しているので、`bubble_sort` などが多言語集約されるはずだった。

しかしAPIでライセンスを確認すると衝撃の結果が出た。

```
TheAlgorithms/C:    license=GPL-3.0   ← GPL！
TheAlgorithms/Rust: license=MIT       ← OK
```

`TheAlgorithms/C` はMITに見えて実際は **GPL-3.0** だった。名前に `TheAlgorithms` とあるし、教育目的のリポジトリなのでMITと思い込んでいた。

GPLのコードをデータセットに含めてHuggingFaceで公開すると、データセット自体もGPLで公開する必要が生じる可能性がある。これはまずい。

**対処：GPLフィルタを2箇所に追加**

```python
_BLOCKED_LICENSES = frozenset({
    "GPL-2.0", "GPL-3.0", "LGPL-2.0", "LGPL-2.1",
    "LGPL-3.0", "AGPL-3.0",
})
_BLOCKED_REPOS = frozenset({"TheAlgorithms/C"})

def _fetch_repo_info(full_name: str) -> Optional[dict]:
    if full_name in _BLOCKED_REPOS:
        log.warning("除外済みリポジトリ: %s（GPL確認済み）", full_name)
        return None
    ...
    lic_id = item.get("license", {}).get("spdx_id", "")
    if lic_id in _BLOCKED_LICENSES:
        log.warning("GPL除外: %s (%s)", full_name, lic_id)
        return None
```

`_fetch_repo_info`（paired_repos収集時）と `search_repos`（API検索時）の両方にフィルタを追加した。今後はどんなリポジトリが検索にヒットしてもGPL系は自動でスキップされる。

**代替リポジトリの選定：**

| リポジトリ | ライセンス | 特徴 |
|---|---|---|
| `fragglet/c-algorithms` | ISC（MIT互換） | list/hash/sort/search が揃っている |
| `swenson/sort` | MIT | bubble/quick/merge/heap など12種以上 |
| `afiskon/c-algorithms` | MIT | sort/search/graph/tree |

---

### 問題⑤ optimized_change が常に不明（null）

最適化によってアセンブラが変化するかどうかを記録する `optimized_change` フラグが、ほぼ全件 `null` になっていた。

原因を調査した結果、重複除去のキーが問題だとわかった。

```python
# 問題のあったコード
records: dict[str, AsmRecord] = {}  # asm_hash のみをキーに
```

`-O0` と `-O3` が同じアセンブラを生成する関数（最適化の余地がない単純な関数）があるとする。このとき `asm_hash` が同じなので、1件目が登録された後は2件目がスキップされる。結果、`-O0` と `-O3` の両方が揃わないため比較ができず `optimized_change` が `null` のままになる。

**修正：(asm_hash, opt_level) の複合キーに変更**

```python
# 修正後
records: dict[tuple, AsmRecord] = {}  # (asm_hash, opt_level) の複合キー

rec_key = (h, opt_level_int)
if rec_key in records:
    # 同一アセンブラ・同一最適化レベルが既存 → sources に追記
    ...
else:
    records[rec_key] = AsmRecord(...)
```

これにより `-O0` と `-O3` が同じアセンブラでも別レコードとして保持され、`optimized_change=False`（最適化で変化なし）のレコードも正確に記録できるようになった。

---

### 問題⑥ normalize_func_name のキャメルケース変換バグ

多言語集約のキーとなる `normalize_func_name` で、`bubbleSort → bubble_sort` の変換が正しく動いていなかった。

```python
# バグのあったコード
base = name.split("::")[-1].split("<")[0].strip().lower()  # 先に小文字化してしまう
base = re.sub(r"([a-z])([A-Z])", r"\1_\2", base).lower()   # 大文字が消えているので変換が効かない
```

`lower()` を先に実行してしまうと、後の正規表現 `([a-z])([A-Z])` が大文字を検出できなくなる。

```python
# 修正後：小文字化前にキャメルケース変換
base = name.split("::")[-1].split("<")[0].strip()  # 大文字を保持したまま取得
base = re.sub(r"([a-z])([A-Z])", r"\1_\2", base)   # キャメルケース変換
base = re.sub(r"([A-Z]+)([A-Z][a-z])", r"\1_\2", base)  # HTTPSResponse 等にも対応
base = base.lower()
```

全10テストケースが合格した。

```
✅ bubbleSort    → bubble_sort
✅ binarySearch  → binary_search
✅ mergeSort     → merge_sort
✅ quickSort     → quick_sort
✅ fib_recursive → fibonacci（エイリアス解決）
✅ nth_fibonacci → fibonacci（エイリアス解決）
```

---

## DWARF解析でsize_bytesを取得する

型情報がなければアセンブラの理解は浅い。`int add(int a, int b)` という関数がある。アセンブラでは `mov eax, edi` という命令が使われるかもしれない。なぜ `rdi`（64bit）ではなく `edi`（32bit）が使われるのか——それは `int` が4バイト（32bit）だからだ。

この情報をDWARFデバッグ情報から取得する。

### コンパイル時の設定

```bash
# C言語
gcc -O3 -g -march=x86-64 -c src.c -o src.o
#       ↑ これがDWARF情報を付与するフラグ

# Rust
rustc -C opt-level=3 -C debuginfo=2 --crate-type lib src.rs -o src.o
#                        ↑ debuginfo=2 が DWARF情報を付与する
```

### 解析の仕組み

```
ELFオブジェクトファイル (.o)
  └─ .debug_info セクション
       └─ DW_TAG_subprogram（関数）
            ├─ DW_AT_name: "add"
            ├─ DW_AT_type: → DW_TAG_base_type
            │                  ├─ DW_AT_name: "int"
            │                  └─ DW_AT_byte_size: 4
            └─ DW_TAG_formal_parameter（引数）
                 ├─ DW_AT_name: "a"
                 └─ DW_AT_type: → DW_TAG_base_type (int, 4)
```

`pyelftools` でこの構造を辿ることで、引数の型名とバイトサイズを取得できる。ポインタ（`char *`）は常に8バイト（x86-64）、`int` は4バイト、`long` は8バイト、`struct` は実際のサイズが取得できる。

### 2段階フォールバック

`pyelftools` が未インストールの場合でも動作するよう、`objdump --dwarf=info` のテキスト出力をパースするフォールバックを実装した。

```python
def extract_dwarf_func_info(obj_path: Path, func_name: str) -> Optional[FuncMeta]:
    # pyelftools 優先
    try:
        return _extract_dwarf_pyelftools(obj_path, func_name)
    except ImportError:
        pass
    # フォールバック: objdump テキストパース
    try:
        return _extract_dwarf_objdump(obj_path, func_name)
    except Exception:
        pass
    return None  # 失敗してもスクリプトは続行
```

### 取得できるメタ情報の例

```json
"func_meta": {
  "name": "ActivitySelection",
  "return_type": null,
  "return_size_bytes": null,
  "params": [
    {
      "name": "a",
      "type_name": "struct Activity *",
      "size_bytes": 8,
      "is_pointer": true
    },
    {
      "name": "n",
      "type_name": "int",
      "size_bytes": 4,
      "is_pointer": false
    }
  ],
  "dwarf_parsed": true
}
```

---

## 多言語集約の設計

「CとRustで同じアルゴリズムを書いたら同じアセンブラになる」——この知識をAIに学習させるには、同じ関数を別言語で実装したコードを1レコードに集約する必要がある。

### Phase 2.5：正規化関数名による集約

```python
FUNC_NAME_ALIASES = {
    "bubble_sort":   ["bubble_sort", "bubbleSort", "bubble", "bsort"],
    "quick_sort":    ["quick_sort",  "quickSort",  "quicksort", "qsort"],
    "binary_search": ["binary_search", "binarySearch", "binary_search_iterative"],
    "fibonacci":     ["fibonacci", "fib", "fib_recursive", "nth_fibonacci"],
    # ... 44種類のアルゴリズムを定義
}

def normalize_func_name(name: str) -> str:
    base = name.split("::")[-1].split("<")[0].strip()
    base = re.sub(r"([a-z])([A-Z])", r"\1_\2", base)
    base = base.lower()
    return _ALIAS_REVERSE.get(base, base)
```

`(normalized_func_name, opt_level)` が同じレコードを `sources` に統合する。

### PAIRED_REPOS：同一アルゴリズムを両言語で実装しているリポジトリペア

```python
PAIRED_REPOS = {
    "fragglet/c-algorithms":      {"rust": "TheAlgorithms/Rust"},
    "swenson/sort":               {"rust": "EbTech/rust-algorithms"},
    "afiskon/c-algorithms":       {"rust": "alexfertel/rust-algorithms"},
    "TheAlgorithms/Rust":         {"c":    "fragglet/c-algorithms"},
    "EbTech/rust-algorithms":     {"c":    "swenson/sort"},
    "alexfertel/rust-algorithms": {"c":    "afiskon/c-algorithms"},
}
```

これらのリポジトリを通常のAPI検索より前に**最優先で収集**することで、関数名が一致しやすいデータが集まりやすくなる。

---

## 現在の品質指標

最新版（修正全適用後）のデータセットで確認できた数値だ。

| 指標 | 値 | 評価 |
|---|---|---|
| 総レコード数 | 811件 | 🟡 増加中 |
| C言語ソースコード対応率 | 97.7% | ✅ 良好 |
| Rustソースコード対応率 | **98.8%** | ✅ 良好 |
| DWARF解析成功率 | **75.5%** | ✅ 良好 |
| DWARFのparams付き | 69.7% | ✅ 良好 |
| 全ソースコード空欄 | **1.8%** | ✅ ほぼ解消 |
| 最適化で変化あり | 64.1% | ✅ 正常 |
| Rustマングリング残存 | **0件** | ✅ 完全解消 |
| 多言語集約 | 4件 | 🔧 改善中 |

主要な品質指標は実用レベルに達した。多言語集約はMITリポジトリの差し替えで増加予定だ。

### Rustのcode_match 内訳

```
full_file   : 348件（ファイル全体をコンテキストとして保持）
short_name  : 167件（末尾セグメントで関数を特定）
none        :   6件（取得不可・1.2%）
→ コードあり率: 98.8%
```

`full_file` の348件はトレイト実装関数だ。`_<impl Ord for usize>::cmp` のような関数は `fn` 名との直接一致が取れないため、ファイル全体をコンテキストとして保持している。「このアセンブラはこのファイルから生成された」という対応関係は維持されている。

---

## セットアップと実行方法

### 必要な環境

```bash
# Python パッケージ
pip install requests pyarrow pandas tqdm pyelftools

# コンパイラ（Linux）
sudo apt install gcc binutils

# Rust（MIT LicenseなのでRust公式インストーラーを推奨）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env

# rustfilt（任意・未インストールでも動作する）
cargo install rustfilt
```

### 実行

```bash
export GITHUB_TOKEN=ghp_xxxxxxxxxxxx

# C + Rust（推奨）
python3 asm_dataset_pipeline_multilang.py \
    --langs c rust \
    --max-repos 50 \
    --max-files 200

# C言語のみ（軽量テスト）
python3 asm_dataset_pipeline_multilang.py \
    --langs c \
    --max-repos 10
```

### externally-managed-environment エラーが出た場合

Debian/Ubuntu系Linuxでよく起きるエラーだ。仮想環境を使うのが安全だ。

```bash
python3 -m venv ~/.venv/asm_dataset
source ~/.venv/asm_dataset/bin/activate
pip install requests pyarrow pandas tqdm pyelftools
```

---

## 失敗から学んだこと

このプロジェクトで実際に起きたミスをまとめると、こうなる。

**ライセンスは名前で判断するな。APIで確認しろ。** `TheAlgorithms/C` はGPL-3.0だった。有名リポジトリでも思い込みは禁物だ。

**重複除去のキーは慎重に設計しろ。** `asm_hash` 単体でキーにすると、最適化レベルをまたいだ比較が壊れる。`(asm_hash, opt_level)` の複合キーが正解だった。

**文字列処理の順序に注意しろ。** `lower()` してから正規表現でキャメルケース変換しても動かない。変換してから小文字化が正しい順序だ。

**言語ごとの「普通」が違う。** Rustは `main` が必要、マングリングされる、トレイト実装が複雑——C言語の常識をそのまま持ち込んではいけない。

**フォールバックを必ず用意しろ。** `pyelftools` がなくても動く、`rustfilt` がなくても動く、DWARFが解析できなくても続行する。一点が止まったら全体が止まるパイプラインは運用できない。

---

## まとめ

このプロジェクトで作ったもの、解決した問題、現在の品質をひと言でまとめると次のようになる。

- GitHub API → GCCコンパイル → objdump の完全自動パイプラインを構築した
- `--crate-type lib` の欠落・マングリング全除外・GPL混入という3つの致命的バグをすべて修正した
- DWARFによる型情報（`size_bytes`）取得でデータの学習価値が上がった
- 3段階フォールバックでRustのソースコード対応率を0%から98.8%に改善した
- 重複除去を `(asm_hash, opt_level)` 複合キーにして最適化比較を正確に記録できるようにした
- `normalize_func_name` のキャメルケース変換バグを修正し、多言語集約の基盤が完成した
- GPLリポジトリの自動除外フィルタを追加し、ライセンス的に安全なデータセットにした

次のステップはHuggingFaceへの公開とデータカード（README）の整備、そして多言語集約の件数増加だ。

---

**▶ スクリプト全文・HuggingFace公開状況・続編 → [gstate.dev](https://gstate.dev/)**

G.stateでは、AIとレガシー技術をつなぐプロジェクトを継続的に公開している。アセンブラデータセットの進捗・コード・HuggingFaceへの公開状況はすべてサイトで追える。

---

<!--
==========================================================
SEO / AIO / GEO 対応メタ情報
note投稿時はこのコメントブロックごと削除してください
==========================================================

【ページタイトル】
AIにアセンブラを学習させるデータセットをゼロから作った全記録 | G.state

【メタディスクリプション（150字以内）】
GitHub APIでMITリポジトリを収集しGCC/rustcでコンパイル、objdumpでアセンブラを抽出。--crate-type lib問題・マングリング除外・GPL混入・DWARF解析・多言語集約まで全デバッグ記録をG.stateが公開。

【主要キーワード（SEO）】
アセンブラ データセット 作り方
逆コンパイル AI 学習データ
HuggingFace アセンブラ 公開
C言語 Rust アセンブラ 対訳
DWARF 型情報 size_bytes pyelftools
GCC 最適化 -O0 -O3 objdump
SHA256 重複除去 複合キー
GPLライセンス 除外 TheAlgorithms
rustc crate-type lib マングリング
Claude Code AI パイプライン 自動化
extern managed environment Python venv

【AI検索（AIO）想定Q&A】
Q: アセンブラのデータセットはどうやって作るか？
A: GitHub APIでMITライセンスのCリポジトリを収集し、GCCで-O0〜-O3の4段階でコンパイル、objdumpでアセンブラを抽出してJSON/Parquet形式で保存する。重複除去は(asm_hash, opt_level)の複合キーで行う。

Q: rustcでライブラリのオブジェクトファイルを生成するには？
A: --crate-type libオプションが必要。これを付けないとmain関数が必須になりコンパイルエラーになる。

Q: Rustのマングリングシンボル(_ZN...)を関数名として使うには？
A: rustfiltコマンドにパイプするか、正規表現で_ZN+長さプレフィクス付きセグメント列をパースして::区切りの人間可読形式に変換する。

Q: DWARFからsize_bytesを取得するには？
A: GCCに-g、rustcに-C debuginfo=2を付けてコンパイルし、pyelftoolsでDW_TAG_formal_parameterのDW_AT_byte_sizeを読む。pyelftools未インストール時はobjdump --dwarf=infoで代替できる。

Q: GPLリポジトリをデータセットから除外するには？
A: GitHub APIのlicense.spdx_idがGPL-2.0/GPL-3.0等に一致する場合はスキップする。TheAlgorithms/CはGPL-3.0なので注意。

Q: optimized_changeが常にNullになる原因は？
A: 重複除去キーがasm_hash単体だと、-O0と-O3が同じアセンブラを生成する関数が1件に統合されて比較対象が消える。(asm_hash, opt_level)の複合キーにする。

【GEO（生成エンジン最適化）設計ポイント】
・各問題を「問題番号→原因→修正コード→結果」の構造で提示
・技術用語（crate-type lib / マングリング / DWARF / spdx_id）を本文内で定義
・数値データ（98.8% / 75.5% / 1.8%）で品質を定量化
・まとめを箇条書きで整理
・gstate.devへの導線を冒頭・末尾の両方に設置

【推奨ハッシュタグ（note）】
#アセンブラ #データセット #AI #機械学習 #HuggingFace
#C言語 #Rust #逆コンパイル #セキュリティ #ClaudeCode
#オープンソース #低レイヤー #x86 #プログラミング #DWARF

==========================================================
-->
