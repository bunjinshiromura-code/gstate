---
title:        "C・Rust対応アセンブラ対訳データセット構築の全工程——設計・6つのバグ修正・DWARF解析・多言語集約まで"
author:       bunjin
published:    true
date:         2025-03-24
updated_at:   2025-03-24
private:      false
tags:
  - Assembly
  - Rust
  - C
  - MachineLearning
  - Security
  - HuggingFace
  - GCC
  - DWARF
  - ReverseEngineering
  - AI
---

:::note info
この記事は **G.state**（[gstate.dev](https://gstate.dev/)）が進める「C言語 ↔ アセンブラ対訳データセット」構築プロジェクトの実装記録です。スクリプト全文・HuggingFace公開状況は [gstate.dev](https://gstate.dev/) で追えます。
:::

# C・Rust対応アセンブラ対訳データセット構築の全工程

GitHub API → コンパイル → ディスアセンブル → DWARF解析 → 多言語集約 → JSON/Parquet出力の完全自動パイプラインを構築した実装記録。6つの致命的バグとその解決策を中心にまとめる。

---

## 背景：なぜアセンブラのデータセットが必要か

逆コンパイルツール（Ghidra / IDA Pro）はアセンブラをC言語相当に変換するが、その精度をAIで改善するためのデータセットが世界的に不足している。HuggingFace上でアセンブラ関連のデータセットを探してみると、その少なさに気づく。

マルウェア解析・脆弱性研究の現場では「アセンブラを理解するAI」への需要が高い。データがないから作る。

---

## データ設計：言語中立スキーマ

将来の多言語拡張を見越し、**アセンブラを主キー**として複数言語のソースを集約する設計にした。

### NG設計：言語固有スキーマ

```json
// C専用で設計してしまった場合
{"c_code": "int add(int a, int b){...}", "asm_code": "lea eax, [rdi+rsi]\n ret"}

// 後からRustを追加すると別テーブルになり、同一アセンブラを学習できない
{"rust_code": "fn add(a: i32, b: i32) -> i32 {...}", "asm_code": "lea eax, [rdi+rsi]\n ret"}
```

### OK設計：言語中立スキーマ

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
      {"name": "a", "type_name": "int",  "size_bytes": 4, "is_pointer": false},
      {"name": "b", "type_name": "int",  "size_bytes": 4, "is_pointer": false}
    ],
    "dwarf_parsed":       true
  },
  "sources": [
    {
      "lang": "c",    "compiler": "gcc",   "compiler_ver": "13.2",
      "code": "int add(int a, int b){ return a+b; }",
      "opt_flag": "-O3", "code_match": "exact",
      "func_name": "add", "source_repo": "fragglet/c-algorithms", ...
    },
    {
      "lang": "rust", "compiler": "rustc", "compiler_ver": "1.75",
      "code": "fn add(a: i32, b: i32) -> i32 { a + b }",
      "opt_flag": "3",  "code_match": "short_name",
      "func_name": "add", "source_repo": "TheAlgorithms/Rust", ...
    }
  ],
  "normalized_func_name": "add"
}
```

| フィールド | 役割 |
|---|---|
| `asm_hash` | SHA256による重複除去キー（`opt_level`との複合キー） |
| `func_meta.size_bytes` | DWARFから取得した引数のバイトサイズ |
| `func_meta.optimized_change` | O0とO3でアセンブラが変わるかフラグ |
| `code_match` | ソースコードの取得方法（exact/short_name/full_file/none） |
| `normalized_func_name` | 多言語集約の照合キー（`bubbleSort → bubble_sort`） |

---

## パイプライン全体像

```
GitHub API
  ↓ MITライセンスリポジトリ収集
  ↓ GPL系（BLOCKED_LICENSES）を自動除外
  ↓ PAIRED_REPOS（同一アルゴリズムのC/Rustペア）を優先収集

ソースファイルダウンロード
  ↓ C:    GCC  -O0/-O1/-O2/-O3  -g（DWARF付与）
  ↓ Rust: rustc opt-level=0〜3  --crate-type lib  -C debuginfo=2

アセンブラ抽出
  ↓ objdump -d -M intel --no-show-raw-insn
  ↓ 関数単位に分割（5〜500命令でフィルタ）
  ↓ Rust: _ZN...マングリングをデマングル

重複除去
  ↓ (asm_hash, opt_level) の複合キー
  ↓ ※asm_hashのみだとO0=O3の比較が壊れる

DWARF解析
  ↓ pyelftools で DW_TAG_formal_parameter を解析
  ↓ 引数の型名・size_bytes を取得
  ↓ フォールバック: objdump --dwarf=info テキストパース

Phase 2.5: 多言語集約
  ↓ normalize_func_name で関数名を正規化
  ↓ (normalized_func_name, opt_level) が同じなら sources に統合

optimized_change フラグ付与
  ↓ (source_repo, normalized_func_name, lang) をキーに
  ↓ O0ハッシュ と O3ハッシュ を比較

JSON / Parquet（snappy圧縮）出力
```

---

## Bug #1: `--crate-type lib` の欠落（Rustレコード0件）

### 症状

```
✅ データセット生成完了
   [c    ] ソースあり: 301 件
   [rust ] ソースあり:   0 件  ← 全件0
```

### 原因

Rustはデフォルトで `bin`（実行ファイル）クレートを生成しようとするため、`fn main()` が必須。

```bash
# ❌ main がないとコンパイルエラー
rustc -C opt-level=0 --emit obj src.rs
# error[E0601]: `main` function not found in crate `src`
```

### 修正

```bash
# ✅ --crate-type lib を追加
rustc -C opt-level=0 --crate-type lib --emit obj src.rs
```

```python
# compile_rust 関数の修正箇所
r = subprocess.run([
    "rustc",
    f"-C", f"opt-level={opt_level}",
    f"-C", "target-cpu=x86-64",
    f"-C", "panic=abort",
    f"-C", "debuginfo=2",
    "--emit", "obj",
    "--edition", "2021",
    "--crate-type", "lib",   # ← この1行が必要
    "-A", "warnings",
    str(src), "-o", str(obj),
])
```

### 結果

Rustレコード: **0件 → 897件**

---

## Bug #2: Rustの全関数がフィルタで除外される

### 症状

コンパイルは成功（returncode=0）、関数も286件検出。しかしデータセットに入らない。

```
検出関数数: 286
  _ZN100_$LT$alloc..boxed..Box...$GT$4from17hd51b770aec3f70eeE: 11命令
  _ZN102_$LT$core..iter..adapters...4fold17h89ae5eee13a14664E:   9命令
```

### 原因

`parse_functions_from_objdump` の除外フィルタ：

```python
# ❌ _ZN はRustの全関数シンボルに付くプレフィクス
if any(name.startswith(p) for p in ("_ZN", "__", ".L", "GCC_")):
    current = None  # Rustの全関数がここで除外される！
```

### 修正：lang 引数を追加してRustは除外せずデマングル

```python
def _demangle_rust_simple(name: str) -> str:
    """rustfilt 優先 → 正規表現フォールバック"""
    # rustfilt が使えれば優先
    try:
        r = subprocess.run(
            ["rustfilt"], input=name, capture_output=True, text=True, timeout=5
        )
        if r.returncode == 0 and r.stdout.strip() != name:
            return r.stdout.strip()
    except FileNotFoundError:
        pass

    # 簡易パース: _ZN + セグメント列 + ハッシュ + E
    # _ZN4core3cmp3max17h...E → core::cmp::max
    m = re.match(r"^_ZN(.+?)(?:17h[0-9a-f]+E|E)$", name)
    if not m:
        return name
    inner = m.group(1)
    segments = []
    i = 0
    while i < len(inner):
        length_str = ""
        while i < len(inner) and inner[i].isdigit():
            length_str += inner[i]; i += 1
        if not length_str: break
        length = int(length_str)
        seg = inner[i:i+length]
        # Rustの特殊エスケープを展開
        for enc, dec in [("$LT$","<"),("$GT$",">"),("$u20$"," "),("$C$",","),("$RF$","&")]:
            seg = seg.replace(enc, dec)
        segments.append(seg)
        i += length
    return "::".join(segments)[:80] if segments else name


def parse_functions_from_objdump(
    objdump_out: str,
    lang: str = "c",
) -> dict[str, str]:
    functions: dict[str, list[str]] = {}
    current: Optional[str] = None

    for line in objdump_out.splitlines():
        m = FUNC_HEADER.match(line)
        if m:
            raw_name = m.group(1)

            if lang == "rust":
                if raw_name.startswith("_ZN"):
                    name = _demangle_rust_simple(raw_name)  # デマングル
                elif any(raw_name.startswith(p) for p in ("__", ".L", "GCC_")):
                    current = None; continue
                else:
                    name = raw_name
                current = name
                functions.setdefault(current, [])
            else:
                # C / Python: 内部シンボルを除外
                if any(raw_name.startswith(p) for p in ("_ZN", "__", ".L", "GCC_")):
                    current = None; continue
                current = raw_name
                functions[current] = []
            continue

        if current and INSN_LINE.match(line):
            functions[current].append(line.strip())

    return {
        name: "\n".join(insns)
        for name, insns in functions.items()
        if MIN_INSN_COUNT <= len(insns) <= MAX_INSN_COUNT
    }
```

### 結果

Rustマングリング残存: **全件 → 0件**

---

## Bug #3: Rustのソースコード対応率0%

### 症状

```
Rust code_match 内訳:
  none: 897件（100%）← 全件ソースコード空欄
```

### 原因

デマングル後の関数名が `fn` の関数名と一致しない。

```
# デマングル後の名前
_<alloc::boxed::Box<dyn core::error::Error> as core::convert::From<E>>::from

# Rustソース内の実際の fn 名
fn from(e: E) -> Self { ... }
```

トレイト実装関数は `fn from(` とは書かれているが、デマングル後のシンボル名は型修飾が付いた長い名前になる。単純な正規表現マッチでは取れない。

### 修正①: `extract_functions_rust` を強化

```python
def extract_functions_rust(source: str) -> dict[str, str]:
    """
    - pub fn / async fn / unsafe fn / impl ブロック内 fn を抽出
    - _FULL_FILE_ キーでファイル全体も保存（フォールバック用）
    """
    functions: dict[str, str] = {}
    lines = source.splitlines()

    # pub/async/unsafe fn を全て対象に
    fn_pattern = re.compile(
        r"^\s*(?:pub(?:\s*\([^)]*\))?\s+)?(?:async\s+)?(?:unsafe\s+)?fn\s+(\w+)\s*[<(]"
    )

    i = 0
    while i < len(lines):
        m = fn_pattern.match(lines[i])
        if m:
            func_name = m.group(1)
            depth, func_lines, j, started = 0, [], i, False
            while j < len(lines):
                depth += lines[j].count("{") - lines[j].count("}")
                func_lines.append(lines[j])
                if "{" in lines[j]: started = True
                if started and depth <= 0: break
                j += 1
            if len(func_lines) <= MAX_SRC_LINES:
                functions[func_name] = "\n".join(func_lines)
            i = j + 1
        else:
            i += 1

    # フォールバック用にファイル全体も保存
    if len(source.splitlines()) <= MAX_SRC_LINES * 3:
        functions["_FULL_FILE_"] = source

    return functions
```

### 修正②: 3段階フォールバックでソースコードを取得

```python
# ─── ソースコードの取得（Rust特有の対応を含む）───
code       = src_functions.get(func_name, "")
code_match = "exact" if code else "none"

if not code and cfg.lang == "rust":
    # ① 末尾セグメントで再検索
    # "core::cmp::max" → "max" で再検索
    short_name = func_name.split("::")[-1].split("<")[0].strip()
    code = src_functions.get(short_name, "")
    if code:
        code_match = "short_name"

    if not code:
        # ② ファイル全体をコンテキストとして使用
        # トレイト実装等で fn 名が一致しない場合
        code = src_functions.get("_FULL_FILE_", "")
        if code:
            code_match = "full_file"
```

### 結果

| code_match | 件数 |
|---|---|
| exact | — |
| short_name | 167件 |
| full_file | 348件 |
| none | 6件（1.2%） |

Rustソースコード対応率: **0% → 98.8%**

---

## Bug #4: GPL-3.0リポジトリの混入

### 症状

多言語集約のため `TheAlgorithms/C`（C言語版）を追加しようとAPIでライセンスを確認した。

```
TheAlgorithms/C:    license=GPL-3.0   ← GPL！
TheAlgorithms/Rust: license=MIT       ← OK
```

### 問題

データセットにGPLコードを含めると、データセット自体もGPLでの公開が必要になる可能性がある。

### 修正: 動的ライセンスチェックを2箇所に追加

```python
_BLOCKED_LICENSES = frozenset({
    "GPL-2.0", "GPL-3.0", "LGPL-2.0", "LGPL-2.1",
    "LGPL-3.0", "AGPL-3.0",
})
_BLOCKED_REPOS = frozenset({"TheAlgorithms/C"})


def _fetch_repo_info(full_name: str) -> Optional[dict]:
    """paired_repos 収集時にGPLを除外する"""
    if full_name in _BLOCKED_REPOS:
        log.warning("除外済みリポジトリ: %s（GPL確認済み）", full_name)
        return None

    resp = requests.get(f"{GITHUB_API}/repos/{full_name}", headers=_gh_headers())
    if resp.status_code != 200:
        return None

    item   = resp.json()
    lic_id = (item.get("license") or {}).get("spdx_id", "")

    if lic_id in _BLOCKED_LICENSES:
        log.warning("GPL除外: %s (%s)", full_name, lic_id)
        return None

    return {"full_name": item["full_name"], "stars": item.get("stargazers_count", 0),
            "license": lic_id, "paired": True}


def search_repos(cfg: LangConfig, max_repos: int) -> list[dict]:
    # ...
    for item in items:
        fn     = item["full_name"]
        lic_id = (item.get("license") or {}).get("spdx_id", "")

        # ブロックリストチェック
        if fn in _BLOCKED_REPOS or lic_id in _BLOCKED_LICENSES:
            log.debug("GPL除外: %s (%s)", fn, lic_id)
            continue
        # ...
```

### 代替リポジトリ（MIT確認済み）

| リポジトリ | ライセンス | 特徴 |
|---|---|---|
| `fragglet/c-algorithms` | ISC（MIT互換） | list/hash/sort/search が揃っている |
| `swenson/sort` | MIT | bubble/quick/merge/heap など12種以上 |
| `afiskon/c-algorithms` | MIT | sort/search/graph/tree |

---

## Bug #5: `optimized_change` が常に `null`

### 症状

```
最適化変化フラグ:
  変化あり (true) : 344 件
  変化なし (false):   0 件  ← あるはずなのに0件
  不明    (null)  : 854 件  ← 71%が不明
```

### 原因

重複除去キーが `asm_hash` 単体だった。

```python
# ❌ 問題のあったコード
records: dict[str, AsmRecord] = {}  # asm_hash のみ

# O0 と O3 が同じアセンブラを生成する関数（= optimized_change=False になるべき）
# → asm_hash が同じなので O0 が登録後、O3 はスキップされる
# → O0 と O3 のペアが揃わず optimized_change を比較できない
```

### 修正: `(asm_hash, opt_level)` の複合キー

```python
# ✅ 複合キーに変更
records: dict[tuple, AsmRecord] = {}  # (asm_hash, opt_level)

rec_key = (h, opt_level_int)  # O0 と O3 は opt_level が違うので別キー

if rec_key in records:
    # 同一アセンブラ・同一最適化レベルが既存 → sources に追記
    existing_langs = {s.lang for s in records[rec_key].sources}
    if cfg.lang not in existing_langs:
        records[rec_key].sources.append(src_entry)
else:
    records[rec_key] = AsmRecord(
        asm_code   = asm_code,
        opt_level  = opt_level_int,
        asm_hash   = h,
        ...
    )
```

`build_dataset` の `all_records` 辞書も同様に複合キーに変更した。

```python
all_records: dict[tuple, AsmRecord] = {}

for rec in lang_records:
    rec_key = (rec.asm_hash, rec.opt_level)
    if rec_key in all_records:
        # sources を統合
        ...
    else:
        all_records[rec_key] = rec
```

---

## Bug #6: `normalize_func_name` のキャメルケース変換バグ

### 症状

```python
normalize_func_name("bubbleSort")  # → "bubblesort" (間違い)
normalize_func_name("binarySearch")  # → "binarysearch" (間違い)
```

### 原因

```python
# ❌ 先に lower() してから正規表現でキャメルケース変換しようとしていた
base = name.split("::")[-1].split("<")[0].strip().lower()  # 先に小文字化
base = re.sub(r"([a-z])([A-Z])", r"\1_\2", base).lower()   # 大文字が消えて効かない
```

### 修正

```python
# ✅ 大文字を保持したまま変換してから小文字化
def normalize_func_name(name: str) -> str:
    base = name.split("::")[-1].split("<")[0].strip()  # 大文字保持
    base = re.sub(r"([a-z])([A-Z])", r"\1_\2", base)   # bubbleSort → bubble_Sort
    base = re.sub(r"([A-Z]+)([A-Z][a-z])", r"\1_\2", base)  # HTTPSResponse → HTTPS_Response
    base = base.lower()                                 # 最後に小文字化
    return _ALIAS_REVERSE.get(base, base)
```

### 全テストケース合格

```
✅ bubbleSort          → bubble_sort
✅ bubble_sort         → bubble_sort
✅ binarySearch        → binary_search
✅ binary_search_iterative → binary_search（エイリアス解決）
✅ fib_recursive       → fibonacci（エイリアス解決）
✅ nth_fibonacci       → fibonacci（エイリアス解決）
✅ mergeSort           → merge_sort
✅ quickSort           → quick_sort
✅ core::backtracking::backtrack → backtrack
✅ unknown_func        → unknown_func（変換なし）
```

---

## DWARF解析による型情報取得の実装

### なぜ必要か

```nasm
; int add(int a, int b) の -O3 出力
add:
    lea eax, [rdi+rsi]   ; なぜ eax（32bit）で rax（64bit）でないのか？
    ret
```

引数が `int`（4バイト = 32bit）だから `eax` が使われる。この「なぜ」をAIに学習させるには型情報が必要だ。

### コンパイルフラグ

```bash
# C言語: -g でDWARF情報を付与
gcc -O3 -g -march=x86-64 -std=c11 -c src.c -o src.o

# Rust: -C debuginfo=2 でDWARF情報を付与
rustc -C opt-level=3 -C debuginfo=2 --crate-type lib src.rs -o src.o
```

### pyelftools による解析

```python
from elftools.elf.elffile import ELFFile

def _extract_dwarf_pyelftools(obj_path: Path, func_name: str) -> Optional[FuncMeta]:
    with open(obj_path, "rb") as f:
        elf   = ELFFile(f)
        if not elf.has_dwarf_info():
            return None
        dwarf = elf.get_dwarf_info()

        for CU in dwarf.iter_CUs():
            for DIE in CU.iter_DIEs():
                if DIE.tag != "DW_TAG_subprogram":
                    continue
                # 関数名を確認
                name_attr = DIE.attributes.get("DW_AT_name")
                if not name_attr:
                    continue
                dname = name_attr.value
                if isinstance(dname, bytes):
                    dname = dname.decode("utf-8", errors="replace")
                if dname != func_name:
                    continue

                # 戻り値型を解決
                ret_type, ret_size = _resolve_type_from_die(DIE, CU, dwarf)

                # 引数（DW_TAG_formal_parameter）を収集
                params = []
                for child in DIE.iter_children():
                    if child.tag != "DW_TAG_formal_parameter":
                        continue
                    p_name_attr = child.attributes.get("DW_AT_name")
                    p_name = None
                    if p_name_attr:
                        pn = p_name_attr.value
                        p_name = pn.decode("utf-8", errors="replace") if isinstance(pn, bytes) else pn

                    p_type, p_size = _resolve_type_from_die(child, CU, dwarf)
                    is_ptr = p_type is not None and ("*" in p_type or "&" in p_type)

                    params.append(ParamMeta(
                        name=p_name, type_name=p_type,
                        size_bytes=p_size, is_pointer=is_ptr,
                    ))

                return FuncMeta(
                    name=func_name, insn_count=0,
                    return_type=ret_type, return_size_bytes=ret_size,
                    params=params, dwarf_parsed=True,
                )
    return None
```

### 型解決の再帰処理

```python
def _read_type_die(die, CU, dwarf_info, depth: int = 0):
    """typedef / const / volatile / ポインタを再帰的に辿って型名とサイズを返す"""
    if depth > 5:
        return None, None

    tag = die.tag

    if tag == "DW_TAG_base_type":
        # int, char, float などの基本型
        name = die.attributes.get("DW_AT_name")
        size = die.attributes.get("DW_AT_byte_size")
        return (name.value.decode() if name else None,
                size.value if size else None)

    if tag in ("DW_TAG_pointer_type", "DW_TAG_reference_type"):
        # ポインタ・参照: x86-64 は常に8バイト
        inner_name, _ = _follow_type(die, CU, dwarf_info, depth + 1)
        return f"{inner_name} *" if inner_name else "void *", 8

    if tag in ("DW_TAG_typedef", "DW_TAG_const_type", "DW_TAG_volatile_type"):
        # typedef / const / volatile: 元の型を辿る
        typedef_name = die.attributes.get("DW_AT_name")
        inner_name, inner_size = _follow_type(die, CU, dwarf_info, depth + 1)
        final_name = (typedef_name.value.decode() if typedef_name else None) or inner_name
        return final_name, inner_size or _BUILTIN_SIZES.get(final_name or "")

    # ... 配列・構造体・共用体も対応
```

### フォールバック: objdump テキストパース

```python
def _extract_dwarf_objdump(obj_path: Path, func_name: str) -> Optional[FuncMeta]:
    """pyelftools 未インストール時のフォールバック"""
    r = subprocess.run(
        ["objdump", "--dwarf=info", str(obj_path)],
        capture_output=True, text=True, timeout=30
    )
    if r.returncode != 0:
        return None

    lines = r.stdout.splitlines()
    # DW_TAG_subprogram → DW_AT_name → 関数名照合
    # DW_TAG_formal_parameter → DW_AT_name, DW_AT_type → 型解決
    # ...（正規表現でテキストをパース）
```

### 取得結果の例

```json
"func_meta": {
  "name": "ActivitySelection",
  "return_type": null,
  "return_size_bytes": null,
  "params": [
    {"name": "a", "type_name": "struct Activity *", "size_bytes": 8, "is_pointer": true},
    {"name": "n", "type_name": "int",               "size_bytes": 4, "is_pointer": false}
  ],
  "dwarf_parsed": true
}
```

---

## 多言語集約の実装

### PAIRED_REPOS: 同一アルゴリズムをC・Rust両方で実装しているリポジトリペア

```python
PAIRED_REPOS: dict[str, dict[str, str]] = {
    # MIT/ISC 確認済みのペア
    "fragglet/c-algorithms":      {"rust": "TheAlgorithms/Rust"},
    "swenson/sort":               {"rust": "EbTech/rust-algorithms"},
    "afiskon/c-algorithms":       {"rust": "alexfertel/rust-algorithms"},
    "TheAlgorithms/Rust":         {"c":    "fragglet/c-algorithms"},
    "EbTech/rust-algorithms":     {"c":    "swenson/sort"},
    "alexfertel/rust-algorithms": {"c":    "afiskon/c-algorithms"},
}
```

### FUNC_NAME_ALIASES: 44種類のアルゴリズム名正規化テーブル

```python
FUNC_NAME_ALIASES: dict[str, list[str]] = {
    # ソートアルゴリズム
    "bubble_sort":    ["bubble_sort", "bubbleSort", "bubble", "bsort"],
    "quick_sort":     ["quick_sort",  "quickSort",  "quicksort", "qsort",
                       "quick_sort_recursive", "partition"],
    "merge_sort":     ["merge_sort",  "mergeSort",  "merge",  "mergesort"],
    "heap_sort":      ["heap_sort",   "heapSort",   "heapsort"],
    "insertion_sort": ["insertion_sort", "insertionSort", "insert_sort"],
    "selection_sort": ["selection_sort", "selectionSort", "select_sort"],
    # 探索アルゴリズム
    "binary_search":  ["binary_search", "binarySearch",
                       "binary_search_iterative", "binary_search_recursive"],
    "linear_search":  ["linear_search", "linearSearch", "sequential_search"],
    # 数学
    "factorial":      ["factorial", "fact", "factorial_recursive",
                       "factorial_iterative"],
    "fibonacci":      ["fibonacci", "fib", "fib_recursive",
                       "fibonacci_iterative", "nth_fibonacci"],
    "gcd":            ["gcd", "greatest_common_divisor", "euclidean_gcd"],
    # グラフ
    "bfs":            ["bfs", "breadth_first_search", "breadthFirstSearch"],
    "dfs":            ["dfs", "depth_first_search",   "depthFirstSearch"],
    "dijkstra":       ["dijkstra", "dijkstras",        "dijkstra_shortest_path"],
    # バックトラッキング
    "backtrack":      ["backtrack", "backtracking",    "solve"],
    # ... 合計44種類
}
```

### Phase 2.5: `_merge_by_normalized_name`

```python
def _merge_by_normalized_name(records: list[AsmRecord]) -> list[AsmRecord]:
    """
    (normalized_func_name, opt_level) が同じレコードを多言語集約する。
    asm_hash が異なっても（= アセンブラが違っても）同一アルゴリズムとして集約。
    """
    norm_map: dict[tuple, int] = {}   # (nfn, opt_level) → result インデックス
    result: list[AsmRecord] = []

    for rec in records:
        nfn = rec.normalized_func_name
        if not nfn:
            result.append(rec); continue

        key = (nfn, rec.opt_level)

        if key in norm_map:
            primary = result[norm_map[key]]
            primary_langs = {s.lang for s in primary.sources}
            new_sources = [s for s in rec.sources if s.lang not in primary_langs]
            if new_sources:
                primary.sources.extend(new_sources)
        else:
            norm_map[key] = len(result)
            result.append(rec)

    multi_count = sum(1 for r in result if len({s.lang for s in r.sources}) > 1)
    log.info("正規化名集約: %d件 → %d件（多言語集約: %d件）",
             len(records), len(result), multi_count)
    return result
```

---

## 現在の品質指標

| 指標 | 値 | 評価 |
|---|---|---|
| 総レコード数 | 811件 | — |
| C言語ソースコード対応率 | 97.7% | ✅ |
| Rustソースコード対応率 | **98.8%** | ✅ |
| DWARF解析成功率 | **75.5%** | ✅ |
| DWARFのparams付き | 69.7% | ✅ |
| 全ソースコード空欄 | **1.8%** | ✅ |
| 最適化変化あり(true) | 64.1% | ✅ |
| Rustマングリング残存 | **0件** | ✅ |
| 多言語集約 | 4件 | 🔧 改善中 |

---

## セットアップと実行

```bash
# 仮想環境（externally-managed-environment エラーの回避）
python3 -m venv ~/.venv/asm_dataset
source ~/.venv/asm_dataset/bin/activate

# 依存パッケージ
pip install requests pyarrow pandas tqdm pyelftools

# Rust（公式インストーラー推奨）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env
cargo install rustfilt  # 任意・未インストールでもフォールバック動作

# C コンパイラ
sudo apt install gcc binutils  # Linux
brew install gcc binutils      # macOS

# 実行
export GITHUB_TOKEN=ghp_xxxxxxxxxxxx
python3 asm_dataset_pipeline_multilang.py \
    --langs c rust \
    --max-repos 50 \
    --max-files 200
```

---

## まとめ

| バグ | 原因 | 解決策 |
|---|---|---|
| Rustレコード0件 | `--crate-type lib` 未指定 | オプション追加 |
| Rust全関数除外 | `_ZN...` を内部シンボルとして除外 | デマングルして保持 |
| Rustソース0% | トレイト名と `fn` 名の不一致 | 3段階フォールバック |
| GPL混入 | `TheAlgorithms/C` がGPL-3.0 | 動的ライセンスチェック |
| optimized_change常にNull | 重複除去キーが `asm_hash` 単体 | `(asm_hash, opt_level)` 複合キー |
| キャメルケース変換失敗 | 先に `lower()` してから変換 | 変換してから `lower()` |

---

**▶ スクリプト全文・続編 → [gstate.dev](https://gstate.dev/)**

---

<!--
==========================================================
Qiita SEO / AIO / GEO 対応メタ情報
投稿時はこのコメントブロックごと削除してください
==========================================================

【Qiita検索最適化キーワード】
rustc crate-type lib エラー / Rust マングリング デマングル _ZN
objdump DWARF pyelftools 解析 / GCC -g debuginfo=2
アセンブラ データセット 作り方 / HuggingFace Parquet
GPL ライセンス 除外 spdx_id / SHA256 複合キー 重複除去
normalize キャメルケース スネークケース 変換
多言語集約 normalized_func_name / PAIRED_REPOS

【AIO（AI検索）想定Q&A】
Q: rustcでオブジェクトファイルをライブラリとして生成するコマンドは？
A: rustc --crate-type lib --emit obj -C opt-level=3 src.rs -o src.o
   --crate-type libがないとmain関数が必要でエラーになる。

Q: Rustのマングリングシンボル(_ZN...)をデマングルするには？
A: rustfiltコマンドにパイプするか、正規表現で_ZN+長さプレフィクス付きセグメント列をパースして::区切りに変換する。フォールバックとして後者を実装しておくとrustfilt未インストールでも動作する。

Q: DWARFから引数のsize_bytesを取得するPythonコードは？
A: GCCに-g、rustcに-C debuginfo=2を付けてコンパイルし、pyelftoolsのELFFile→get_dwarf_info()→iter_CUs()→iter_DIEs()でDW_TAG_subprogramを探し、DW_TAG_formal_parameterのDW_AT_typeを辿ってDW_AT_byte_sizeを読む。

Q: GitHubのGPLリポジトリをAPI経由で検出するには？
A: GitHub APIのsearch/repositoriesまたはrepos/{owner}/{repo}のレスポンスのlicense.spdx_idがGPL-2.0/GPL-3.0/LGPL等に一致する場合にスキップする。TheAlgorithms/CはGPL-3.0なので要注意。

Q: optimized_changeが常にnullになる場合の対処法は？
A: 重複除去キーをasm_hash単体から(asm_hash, opt_level)の複合キーに変更する。asm_hashのみだと-O0と-O3が同じアセンブラを生成する関数が1件に統合されてペア比較ができなくなる。

Q: Pythonでキャメルケースをスネークケースに変換する正しい順序は？
A: 先にlower()してから正規表現でキャメルケース変換しようとしても大文字が消えて機能しない。正しくは「変換してからlower()」の順序。

【GEO対応ポイント】
・Bug #1〜#6を「症状→原因→修正コード→結果」の構造で提示
・コードブロックに言語指定（python/bash/json/nasm）を明示
・まとめ表で6バグと解決策を一覧化（AIが引用しやすい）
・:::note infoブロックでgstate.devへの導線を記事冒頭に設置
・数値で品質を定量化（98.8% / 75.5% / 1.8% / 0件）

==========================================================
-->
