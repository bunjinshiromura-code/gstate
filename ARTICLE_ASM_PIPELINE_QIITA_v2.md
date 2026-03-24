---
title:        "C・Rust対応アセンブラ対訳データセット構築記——DWARF解析・GPL除外・多言語集約の全実装"
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

# C・Rust対応アセンブラ対訳データセット構築記

GitHub API → コンパイル → ディスアセンブル → DWARF解析 → JSON/Parquet出力のパイプラインを構築した実装記録。5つのバグと解決策を中心にまとめる。

---

## データスキーマ設計

将来の多言語拡張を見越し、アセンブラを主キーとして複数言語のソースを集約する言語中立設計にした。

```json
{
  "asm_code":  "lea eax, [rdi+rsi]\n ret",
  "arch":      "x86-64",
  "opt_level": 3,
  "asm_hash":  "9a4bfb8c1...",
  "func_meta": {
    "name": "add",
    "insn_count": 2,
    "optimized_change": true,
    "calling_convention": "System V AMD64",
    "return_type": "int",
    "return_size_bytes": 4,
    "params": [
      {"name": "a", "type_name": "int", "size_bytes": 4, "is_pointer": false},
      {"name": "b", "type_name": "int", "size_bytes": 4, "is_pointer": false}
    ],
    "dwarf_parsed": true
  },
  "sources": [
    {"lang": "c",    "code": "int add(int a, int b){ return a+b; }",
     "opt_flag": "-O3", "code_match": "exact"},
    {"lang": "rust", "code": "fn add(a: i32, b: i32) -> i32 { a + b }",
     "opt_flag": "3",  "code_match": "short_name"}
  ],
  "normalized_func_name": "add"
}
```

`sources` リストに複数言語を持てる構造にすることで、「CとRustが同じアセンブラを生成する」という知識をAIが一度に学べる。

---

## パイプライン全体像

```
GitHub API（MITライセンスリポジトリ収集・GPL自動除外）
  ↓
コンパイル
  C:    GCC    -O0 / -O1 / -O2 / -O3    + -g（DWARF付与）
  Rust: rustc  opt-level=0/1/2/3        + --crate-type lib + -C debuginfo=2
  ↓
objdump -d -M intel --no-show-raw-insn
  ↓
関数単位で分割（5〜500命令でフィルタ）
  ↓
SHA256(asm_code) + opt_level の複合キーで重複除去
  ↓
DWARF解析（pyelftools → objdump --dwarf=info テキストパース）
  ↓
normalized_func_name による多言語集約（bubbleSort → bubble_sort）
  ↓
optimized_change フラグ付与（O0 vs O3 のハッシュ比較）
  ↓
JSON / Parquet（snappy圧縮）出力
```

---

## 詰まった5つのバグと解決策

### バグ① Rustレコードが0件

**原因：** `rustc` に `--crate-type lib` を付けていなかった。

```bash
# ❌ error[E0601]: `main` function not found in crate `src`
rustc -C opt-level=0 --emit obj src.rs -o src.o

# ✅ main不要でコンパイル成功
rustc -C opt-level=0 --crate-type lib --emit obj src.rs -o src.o
```

Rustはデフォルトで `bin`（実行ファイル）を生成しようとするため `fn main()` が必須になる。`--crate-type lib` を付けることでライブラリとしてコンパイルされ、`pub fn` だけでコンパイルが通る。

### バグ② Rustの関数が全件除外される

**原因：** `parse_functions_from_objdump` で `_ZN...` 始まりのシンボルを内部シンボルとして除外していた。Rustのマングリング形式がこのパターンにすべて一致するため、Rustの全関数が除外されていた。

```python
# ❌ Rustの全関数がここで消える
if any(name.startswith(p) for p in ("_ZN", "__", ".L", "GCC_")):
    current = None
```

```python
# ✅ lang="rust" のときはデマングルして保持する
if lang == "rust":
    if raw_name.startswith("_ZN"):
        name = _demangle_rust_simple(raw_name)
    elif any(raw_name.startswith(p) for p in ("__", ".L", "GCC_")):
        current = None
        continue
    else:
        name = raw_name
else:
    # C / Python: 内部シンボルを除外
    if any(raw_name.startswith(p) for p in ("_ZN", "__", ".L", "GCC_")):
        current = None
        continue
```

`_demangle_rust_simple` は `rustfilt` を優先し、未インストール時は正規表現で長さプレフィクス付きセグメントを展開する。

```python
def _demangle_rust_simple(name: str) -> str:
    # rustfilt 優先
    try:
        r = subprocess.run(["rustfilt"], input=name,
                           capture_output=True, text=True, timeout=5)
        if r.returncode == 0 and r.stdout.strip() != name:
            return r.stdout.strip()
    except FileNotFoundError:
        pass
    # 簡易パース: _ZN + セグメント列 + ハッシュ + E
    m = re.match(r"^_ZN(.+?)(?:17h[0-9a-f]+E|E)$", name)
    if not m:
        return name
    # 長さプレフィクスでセグメントを分解
    inner, segments, i = m.group(1), [], 0
    while i < len(inner):
        length_str = ""
        while i < len(inner) and inner[i].isdigit():
            length_str += inner[i]; i += 1
        if not length_str: break
        length = int(length_str)
        seg = inner[i:i+length]
        for enc, dec in [("$LT$","<"),("$GT$",">"),("$u20$"," "),("$C$",",")]:
            seg = seg.replace(enc, dec)
        segments.append(seg); i += length
    return "::".join(segments)[:80] if segments else name
```

### バグ③ Rustのソースコード対応率が0%

**原因：** デマングル後の関数名（例：`core::cmp::impls::_<impl Ord for usize>::cmp`）はトレイト実装名を含むため、元ソースの `fn` 名と一致しない。

3段階フォールバックで対応した。

```python
code       = src_functions.get(func_name, "")
code_match = "exact" if code else "none"

if not code and lang == "rust":
    # ① 末尾セグメントで再検索（"cmp" など）
    short = func_name.split("::")[-1].split("<")[0].strip()
    code  = src_functions.get(short, "")
    if code:
        code_match = "short_name"

    if not code:
        # ② ファイル全体をコンテキストとして保存
        code = src_functions.get("_FULL_FILE_", "")
        if code:
            code_match = "full_file"
```

結果、Rustのソースコード対応率は **0% → 98.8%** に改善した。内訳は `short_name` 167件・`full_file` 348件・`none` 6件。

### バグ④ GPLリポジトリが混入する

**原因：** 著名な `TheAlgorithms/C` が実際は **GPL-3.0** だった。

GPLコードを含むデータセットをHuggingFaceで公開するとライセンス上のリスクがある。GitHub APIのレスポンスで `license.spdx_id` を動的チェックし、GPL系を自動除外するフィルタを追加した。

```python
_BLOCKED_LICENSES = frozenset({
    "GPL-2.0", "GPL-3.0", "LGPL-2.0", "LGPL-2.1", "LGPL-3.0", "AGPL-3.0",
})
_BLOCKED_REPOS = frozenset({"TheAlgorithms/C"})

def _fetch_repo_info(full_name: str) -> Optional[dict]:
    if full_name in _BLOCKED_REPOS:
        log.warning("GPL除外（確認済み）: %s", full_name)
        return None
    resp   = requests.get(f"{GITHUB_API}/repos/{full_name}", ...)
    lic_id = resp.json().get("license", {}).get("spdx_id", "")
    if lic_id in _BLOCKED_LICENSES:
        log.warning("GPL除外（動的検出）: %s (%s)", full_name, lic_id)
        return None
    return {"full_name": full_name, "license": lic_id, ...}
```

代替として `fragglet/c-algorithms`（ISC・MIT互換）・`swenson/sort`（MIT）・`afiskon/c-algorithms`（MIT）を使用している。

### バグ⑤ optimized_change が常に null

**原因：** 重複除去キーが `asm_hash` 単体だったため、`-O0` と `-O3` が同じアセンブラを生成する関数が1件に統合されてしまい、比較対象が消えていた。

```python
# ❌ O0=O3のとき1件に統合されて比較不能
records: dict[str, AsmRecord] = {}   # asm_hash のみ

# ✅ opt_levelも含む複合キーにする
records: dict[tuple, AsmRecord] = {} # (asm_hash, opt_level)
rec_key = (sha256(asm_code), opt_level_int)
```

これにより「最適化で変化しない関数」（`optimized_change: false`）も正確に記録される。

---

## DWARF解析でsize_bytesを取得する

型サイズはアセンブラの命令選択を決定づける。`int`（4バイト）は `eax` レジスタ、`long`（8バイト）は `rax` レジスタを使うなど、AIの学習にとって重要なシグナルだ。

コンパイル時に `-g` / `-C debuginfo=2` を付与し、`pyelftools` でDWARFのDIEツリーを解析する。

```
DW_TAG_subprogram（関数名・戻り値型）
  └─ DW_TAG_formal_parameter（引数名）
       └─ DW_AT_type → DW_TAG_pointer_type
                          └─ DW_TAG_base_type（"char", byte_size=1）
                             → {"type_name": "char *", "size_bytes": 8}
```

`typedef` / `const` / `volatile` / ポインタは再帰的に辿って解決する。x86-64のポインタは常に8バイトとして処理する。

---

## 現在の品質指標

| 指標 | 値 |
|---|---|
| 総レコード数 | 811件 |
| Cソース対応率 | 97.7% |
| Rustソース対応率 | **98.8%** |
| DWARF解析成功率 | **75.5%** |
| 全ソース空欄 | **1.8%** |
| 最適化変化あり | 64.1% |
| Rustマングリング残存 | **0件** |

---

## まとめ

- **`--crate-type lib`** を付けないとRustは全件コンパイルエラーになる
- **Rustマングリング** は除外せずデマングルして使う（`_ZN...` → `core::cmp::max`）
- **ソース対応** は3段階フォールバック（exact → short_name → full_file）で98.8%達成
- **GPL除外** は `license.spdx_id` で動的チェックする。`TheAlgorithms/C` は GPL-3.0
- **複合キー** `(asm_hash, opt_level)` で重複除去しないと `optimized_change` が壊れる
- **DWARF** は `-g` / `-C debuginfo=2` で付与し、pyelftoolsで型サイズを取得する

---

**▶ スクリプト全文・続編 → [gstate.dev](https://gstate.dev/)**

---

<!--
==========================================================
Qiita SEO / AIO / GEO 対応メタ情報
投稿時はこのコメントブロックごと削除してください
==========================================================

【Qiita検索最適化キーワード】
アセンブラ データセット 作り方 / objdump 使い方
rustc --crate-type lib / Rust マングリング デマングル rustfilt
DWARF pyelftools size_bytes / GCC -g debuginfo
GPLライセンス 除外 MIT / SHA256 重複除去 複合キー
HuggingFace Parquet / x86-64 逆コンパイル AI

【AIO（AI検索）想定Q&A】
Q: rustcでライブラリとしてオブジェクトファイルを生成するには？
A: --crate-type lib --emit obj を付ける。--crate-type libなしではmain関数必須でエラーになる。

Q: objdumpのRustマングリングシンボルをデマングルするには？
A: rustfiltにパイプするか、_ZN...形式を正規表現でパースして長さプレフィクス付きセグメントを::区切りに変換する。

Q: DWARFで引数のsize_bytesを取得する方法は？
A: GCCに-g、rustcに-C debuginfo=2を付けてコンパイルし、pyelftoolsでDW_TAG_formal_parameterのDW_AT_typeを辿ってDW_AT_byte_sizeを取得する。

Q: GPLリポジトリをデータセット収集から動的除外するには？
A: GitHub APIのlicense.spdx_idがGPL-2.0/GPL-3.0等に一致する場合にスキップする。TheAlgorithms/CはGPL-3.0なので要注意。

Q: optimized_changeがNullになる原因と対策は？
A: 重複除去キーがasm_hash単体だとO0=O3のとき1件に統合される。(asm_hash, opt_level)の複合キーにして最適化レベルごとに別レコードを保持する。

【GEO対応ポイント】
・バグ①〜⑤を「原因→コード（❌/✅）→解決策」の構造で提示
・コードブロックに bash/python/json の言語指定を明示
・まとめを箇条書きで整理（AI要約の対象になりやすい）
・:::note infoブロックでgstate.devへの導線を冒頭に設置

==========================================================
-->
