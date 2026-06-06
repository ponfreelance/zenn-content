---
title: "creviewでC言語の危険コード11件を一括検出した話"
emoji: "🛡️"
type: "tech"
topics: ["c", "embedded", "codereview"]
published: true
---

---

C言語23年、自分はフリーランスで組み込みとサーバーサイドC両方のコードレビューに入ることが多いです。若手の PR でよく見る「やらかし」の筆頭が `strcpy` と `sprintf` と `sizeof(ポインタ)` の3点セットで、これらは毎週のように別の PR で再登場します。先週指摘したはずの `strcpy` が、週をまたいで別の関数で出てくる。「Zaif は30分迷った」どころか、自分は1つの PR で同じ指摘を4回書いたことがあります。人間のレビューで全部拾い続けるのは現実的ではないと判断して、自社プロダクトの `creview` に押し付けることにしました。

---

## この記事の前提（SCQA）

- **状況**: C言語のコードレビューで、若手への指摘の7〜8割が `strcpy` / `sprintf` / `sizeof(ポインタ)` / 配列添字未検証の4パターンに収束します。どれも教科書に載っている古典的な危険コードです。
- **問題**: 毎週同じ指摘を書き続けると、レビューワ側が疲弊して「まあいいか」になる日が来ます。その日に限って本番に流れ込むのが現実です。
- **問い**: 定型的な危険コードの検出を機械に任せ、人間のレビューを設計判断に集中させるには、どのツールをどの粒度で使えばいいのか？
- **答え**: コンパイラ警告（L0）を最初の砦にして、その上に自社プロダクト `creview` を重ねることで「再発する定型危険」を自動で全部拾い、人間は「なぜそう設計したのか」にだけ口を出す分業体制を作りました。

---

## L0: まずコンパイラ警告を全部潰す

`creview` の話に入る前に、`-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）または `-Wall -Wextra -Weverything -Werror`（Clang）を有効にしてビルドが通るか確認してください。NULL 引数を nonnull 関数に渡す / 未初期化 auto 変数を読む / 戻り値を破棄する / 自明な NULL deref / 境界外配列アクセス（静的決定可なもの）/ switch の case 抜け、これらは現代のコンパイラが標準で拾います。

これらが警告ゼロにならない PR は、`creview` を走らせる前にそもそもレビュー対象にしないルールにしてしまうのが第一手です。L0 を素通りした PR を `creview` で叩いても、コンパイラが取れた警告まで人間が読むことになって意味がありません。

---

## 検証用コード

今回の検証に使ったコードを丸ごと載せます。`strcpy` / `sprintf` / `sizeof(ポインタ)` / `scanf` 入力をそのまま添字にする、という4パターンを1ファイルに詰め込んだサンプルです。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* スタックバッファのオーバーフロー（strcpy + sizeof 誤用） */
void copy_name(const char *src) {
    char buf[16];
    strcpy(buf, src);                    /* 検出すべき: 長さ未検証 */
    printf("name=%s\n", buf);
}

/* sprintf による境界未検証 */
void format_path(const char *dir, const char *file) {
    char path[32];
    sprintf(path, "%s/%s", dir, file);   /* 検出すべき: snprintf を使うべき */
}

/* sizeof(ptr) ハマりパターン */
void wipe_buffer(char *p) {
    memset(p, 0, sizeof(p));             /* 検出すべき: sizeof(p) は常に 8（ポインタサイズ） */
}

/* 配列インデックス未検証（scanf 入力をそのまま添字に） */
void read_at(int *arr) {
    int i;
    scanf("%d", &i);
    printf("%d\n", arr[i]);              /* 検出すべき: i の上下限チェック無し */
}
```

このコードは「コンパイルは通る」「実行時に落ちるかどうかは入力次第」という、静的解析ツールが最も効果を発揮するゾーンに位置します。

---

## CWE/MISRA マッピング

検出された11件を CWE と「実際に起きる事故の形」に対応付けます。

| 行 | 検出パターン | CWE | 攻撃ベクター / 実害 |
|----|------------|-----|-------------------|
| L6 | NULL チェック漏れ（src） | CWE-476 | NULL を渡した呼び出し元がクラッシュをトリガー。組み込みでは watchdog リセット |
| L7 | マジックナンバー `16` | CWE-547 | バッファサイズ変更時に `strcpy` 側と不整合が生じてオーバーフロー |
| L8 | `strcpy` 使用（長さ未検証） | CWE-121 | スタックバッファオーバーフロー。戻りアドレス破壊 → 任意コード実行 |
| L13 | NULL チェック漏れ（dir） | CWE-476 | NULL を渡すと `sprintf` 内部で NULL deref。SIGSEGV |
| L14 | マジックナンバー `32` | CWE-547 | `path` サイズを変えた際に `sprintf` 側が追随しない |
| L15 | `sprintf` 使用（境界未検証） | CWE-121 | `dir` + `file` の合計が 31 バイトを超えるとスタック破壊 |
| L19 | NULL チェック漏れ（p） | CWE-476 | NULL ポインタに `memset` → 即クラッシュ |
| L20 | `memset` に NULL 未検証ポインタ | CWE-476 | 上記と同一経路。NULL 渡しで未定義動作 |
| L20 | `sizeof(p)` がポインタサイズを返す | CWE-467 | 64bit 環境で `sizeof(p) == 8`。バッファ先頭8バイトしか消えず、残りに秘密データが残留 |
| L24 | NULL チェック漏れ（arr） | CWE-476 | NULL 配列に添字アクセス → SIGSEGV |
| L27 | 外部入力 `i` を境界チェックなしで添字に使用 | CWE-129 | 負値や巨大値を `scanf` で入力するだけで範囲外読み取り（CWE-125）/ 書き込み（CWE-787）に発展 |

MISRA C 2012 との対応を補足すると、`strcpy` / `sprintf` の使用は Rule 21.6（標準入出力関数の制限）および Rule 17.7（戻り値の使用）、マジックナンバーは Rule 2.4（使用されない型）ではなく実務的には Dir 4.1（実行時エラーの最小化）と組み合わせて運用します。

---

## creview の使い方（CLI 実行例）

### インストールと実行

```bash
# 単一ファイルを security プリセットで検査
creview --preset security --format markdown dangerous.c

# PR の差分だけを対象にする場合
creview --preset pr --format sarif dangerous.c

# memory 系チェックに絞る場合
creview --preset memory --format markdown dangerous.c
```

`--format json` / `--format markdown` / `--format sarif` の3形式が使えます。CI に組み込む場合は `--format sarif` を選ぶと GitHub Advanced Security の SARIF ビューアにそのまま取り込めます。

### 実際の検出ログ（上記コードに対する出力）

```
L6 【重大】: ATTR-NONNULL-001: copy_name() の src を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L7 【重大】: マジックナンバー16 (バッファサイズ) — allowlist 外。#define / enum / .creviewrc.json で命名 or 許可
L8 【重大】: strcpy使用。コピー元長未検証でオーバーフロー可能
L13 【重大】: ATTR-NONNULL-001: format_path() の dir を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L14 【重大】: マジックナンバー32 (バッファサイズ) — allowlist 外。#define / enum / .creviewrc.json で命名 or 許可
L15 【重大】: sprintf使用。出力バッファ長未検証でオーバーフロー可能
L19 【重大】: ATTR-NONNULL-001: wipe_buffer() の p を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L20 【重大】: pのNULL未検証でmemset実行。NULLポインタ渡しでクラッシュ
L20 【重大】: sizeof(p)は関数パラメータの decay でポインタサイズ(4/8byte)を返す。配列長は別引数で受け取れ
L24 【重大】: ATTR-NONNULL-001: read_at() の arr を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L27 【重大】: 外部入力iを境界チェックなしで配列添字に使用。範囲外アクセス
```

### 3段階ラベルの読み方

`creview` の severity ラベルは3段階です。

| ラベル | 意味 | 対応優先度 |
|--------|------|-----------|
| 【重大】 | メモリ破壊 / NULL deref / 範囲外アクセスなど、実行時クラッシュまたは任意コード実行に直結する欠陥 | PR マージ前に必ず修正。CI でブロック |
| 【設計不明】 | コードの意図が読み取れず、正しいのか誤りなのか機械では判断できない箇所 | 人間のレビューワが設計意図を確認して判断 |
| 【保守危険】 | 今すぐ落ちるわけではないが、将来の変更で事故を引き起こす可能性が高い箇所（マジックナンバー等） | スプリント内に対処。技術負債として記録 |

今回の検出11件はすべて【重大】です。マジックナンバー（L7 / L14）が【重大】になっているのは、バッファサイズのリテラルが `strcpy` / `sprintf` の長さ未検証と組み合わさると直接オーバーフローに繋がるためです。単独のマジックナンバーなら【保守危険】に落ちますが、`creview` はコンテキストを見て severity を引き上げます。

---

## 修正後のコード（参考）

検出された問題を修正するとこうなります。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define NAME_BUF_SIZE  16
#define PATH_BUF_SIZE  32
#define MAX_ARRAY_IDX  255

/* NULL 契約を属性で明示し、長さ検証に strncpy を使う */
__attribute__((nonnull(1)))
void copy_name(const char *src) {
    char buf[NAME_BUF_SIZE];
    strncpy(buf, src, sizeof(buf) - 1);
    buf[sizeof(buf) - 1] = '\0';
    printf("name=%s\n", buf);
}

/* snprintf で出力バッファ長を明示 */
__attribute__((nonnull(1, 2)))
void format_path(const char *dir, const char *file) {
    char path[PATH_BUF_SIZE];
    snprintf(path, sizeof(path), "%s/%s", dir, file);
}

/* バッファ長を別引数で受け取る */
__attribute__((nonnull(1)))
void wipe_buffer(char *p, size_t len) {
    memset(p, 0, len);
}

/* 外部入力の上下限を検証してから添字に使う */
__attribute__((nonnull(1)))
void read_at(int *arr) {
    int i;
    if (scanf("%d", &i) != 1) { return; }
    if (i < 0 || i > MAX_ARRAY_IDX) { return; }
    printf("%d\n", arr[i]);
}
```

`sizeof(p)` の問題は `wipe_buffer` のシグネチャを `(char *p, size_t len)` に変えることで解決します。呼び出し側が配列長を知っている場所で `sizeof(配列名)` を渡す設計にします。`__attribute__((nonnull))` は「NULL を渡すのは呼び出し側の契約違反」とコンパイル時に宣言するもので、外部 API / ISR / 通信受信など「本当に NULL が来うる境界」でだけランタイム NULL チェックを追加します。

---

## 3段階使い分け

| 場面 | 使うツール | 使いどころ |
|------|-----------|-----------|
| L0: ビルド時（まずここ） | gcc / clang の警告 + `-Werror` | `-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）/ `-Wall -Wextra -Weverything -Werror`（Clang）。**警告ゼロにならない PR はそもそもレビューに入らない**ルール化 |
| L1: PR チェック（OSS ツール） | clang-tidy / cppcheck | 主流の静的解析を CI で強制。`-checks=clang-analyzer-*,bugprone-*` 等 |
| L2: PR レビュー（自社プロダクト） | creview（CLI） | プロジェクト固有パターン、日本語ナレッジ、severity モデル（重大/設計不明/保守危険）、CWE/MISRA マッピング、`--preset pr` で差分のみ対象、AI 補強 |
| L2': 自己チェック・PR 投げる前（自社プロダクト） | c-review-ai（ブラウザ版） | 環境構築不要。コピペで試せる軽量版。「CI に入れる前にまず自分で確認したい」場面に向く |
| L3: チーム監査・MISRA 準拠（自社プロダクト） | CSAF | libclang AST + 依存グラフで risk A/B/C を自動昇格。正式な MISRA 準拠レポートが必要な案件向け |

**運用ルール**: L0 の警告がゼロにならない PR は CI でブロックします。`creview` はその上に重ねるものです。L0 を素通りさせた状態で `creview` を走らせると、コンパイラが取れたはずの警告まで人間が読むことになって分業の意味がなくなります。

---

## まとめ

- `strcpy` / `sprintf` / `sizeof(ポインタ)` / 添字未検証の4パターンは、コンパイルが通るにもかかわらず実行時にスタック破壊・任意コード実行・秘密データ残留・範囲外アクセスに直結する。CWE-121 / CWE-467 / CWE-476 / CWE-129 が対応する。
- `creview --preset security` を CI に組み込むと、これら11件の【重大】を人間がレビューに入る前に全件検出できる。週をまたいだ再発を機械が止める。
- マジックナンバー（CWE-547）は単独では【保守危険】だが、バッファサイズのリテラルが長さ未検証の `strcpy` / `sprintf` と同一スコープにある場合は【重大】に昇格する。`creview` はこのコンテキスト判定を自動で行う。
- `sizeof(p)` の問題（CWE-467）は64bit環境で `sizeof == 8` になるため、256バイトのバッファを渡しても先頭8バイトしかゼロ埋めされない。秘密


---

### 試すリンク

- **ブラウザで気軽に試す → c-review-ai**（自社プロダクト / MIT / 無料）
  → https://github.com/ponfreelance/c-review-ai
- **CLI で日常運用 → creview**（自社プロダクト / 無料）
  → https://github.com/ponfreelance/creview
- **業務・監査レベル → CSAF**（自社プロダクト / ¥4,980）
  → https://cutt.ly/ItKo4MPY

2026年4月時点の情報です。creview / CSAF のバージョンは随時更新しています。
