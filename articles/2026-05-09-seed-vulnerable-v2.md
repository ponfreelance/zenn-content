---
title: "C言語の危険コードを静的解析で検出する【4層モデル改訂版】"
emoji: "🛡️"
type: "tech"
topics: ["c", "embedded", "codereview", "staticanalysis"]
published: true
---

これは [前回の記事「C言語の危険コードを静的解析ツールcreviewで検出する」](https://qiita.com/ponfreelance/items/66aeea59d1c4e5592ae5) の **改訂版** です。コメント欄で 5 件の本質的な指摘をもらい、自分の整理が浅かったので順番から書き直しました。指摘いただいた方々、ありがとうございます。

直した点を先に並べておきます。

1. `static int g_initialized;` を「未初期化 / CWE-457」と書いた箇所 → CWE-665 系の「init() set 漏れ」設計バグへ
2. NULL チェック漏れの第一選択 → 「関数本体に NULL チェック」から「`__attribute__((nonnull))` でコンパイル時契約 + `warn_unused_result` で戻り値破棄禁止」へ
3. `L76 sizeof(buf)はポインタサイズ` の検出 → ローカル配列宣言なら配列サイズ。**creview の false positive**
4. マジックナンバー検出 → 「malloc 引数だけ拾って配列宣言は見逃す」不整合を、`{-1,0,1,2}` 以外すべて flag・context で severity を切る統一ルールへ
5. 記事の構成 → creview から始めるのではなく、**L0: コンパイラ警告 → L1: 主流 OSS → L2: creview → L3: CSAF** の 4 層モデルへ。**警告ゼロにならない PR はそもそもレビューに入らない**運用ルールが第一手

この 5 つは全部、自分の現場感（C 23 年・組み込みフリーランス）からすれば最初から分かっていないといけなかった話で、特に 5 番目の「**コンパイラ警告から書き始めなかった**」のは構成として致命的でした。改訂版では順番をまっすぐにします。

## SCQA

- **状況**: C のコードレビューで、若手への指摘の 7〜8 割が同じパターン（NULL / 未初期化 / 戻り値破棄 / 再帰 / goto）に収束する
- **問題**: 毎週の PR で同じことを書く。疲れた日に見逃すと本番に流れる
- **問い**: どこまでをコンパイラ・既存 OSS・自作ツールに分業させ、人間は何に集中するか？
- **答え**: 4 層に分けて、層ごとに別ツールに押し付ける。**L0 を素通りした PR はそもそもレビューしない**

## L0: まずコンパイラ警告を全部潰す（最も重要）

creview の話に入る前に、これを通せないコードはレビューに進ませない、というルールを徹底します。

```sh
# GCC 系
gcc -std=c11 -Wall -Wextra -Wpedantic -Werror -fanalyzer -O2 -c src/*.c

# Clang 系
clang -std=c11 -Wall -Wextra -Wpedantic -Werror -Weverything -O2 -c src/*.c
```

これだけで、自分が前回挙げていた検出項目の多くは **コンパイル段階で落ちます**：

| 検出項目 | コンパイラ警告 |
|---|---|
| auto 変数の未初期化使用 | `-Wuninitialized` / `-Wmaybe-uninitialized` |
| nonnull 引数に NULL を渡す | `-Wnonnull` |
| `warn_unused_result` 関数の戻り値破棄 | `-Wunused-result` |
| 自明な NULL deref | `-Wnull-dereference` (Clang) / `-fanalyzer` (GCC) |
| use-after-free / double-free | `-fanalyzer` (GCC) |
| バッファオーバーフロー（静的決定可） | `-Warray-bounds` / `-fanalyzer` |
| `sizeof(pointer)` 配列長誤用 | `-Wsizeof-pointer-memaccess` |
| switch case 抜け（enum 全網羅でない） | `-Wswitch-enum` |
| 戻り値型なし関数 | `-Wreturn-type` |

### 運用ルール（これが本体）

> **L0 の警告がゼロにならない PR は、人間レビューに入らない**

CI に `-Werror` を入れてしまえば物理的に強制できます。これで「またこれか」案件の半分以上は人間が見る前に消えます。

ランタイムでは `-fsanitize=address,undefined`（AddressSanitizer / UBSan）を CI のテストフェーズで走らせて、L0 で取れない動的なバグも捕まえます。

## L1: clang-tidy / cppcheck — 主流 OSS の静的解析

L0 をクリアした PR に対して、次は OSS の主流静的解析を当てます。

- **clang-tidy**: `clang-tidy --checks='cert-*,bugprone-*,readability-*,performance-*'`
- **cppcheck**: `cppcheck --enable=all --inconclusive --std=c11 src/`

これも CI に組み込み、HIGH 以上の警告は `-Werror` 相当で落とします。L0 + L1 で「教科書に載っている定型バグ」はほぼ全部潰せる、というのが現代 C の前提です。

## L2: creview — プロジェクト固有 + 日本語ナレッジ

ここでようやく自作ツール creview の出番です。creview の存在意義は L0 / L1 の **代替** ではなく **後段** で、以下を担います。

- プロジェクト固有のパターン検出（例: 自社規約で禁止されているライブラリ呼び出し / ハンドラ命名規則違反 / レビュー規約違反）
- **日本語**による解説と若手向けのナレッジ提示（「なぜ駄目か」「どう直すか」を業務文脈で）
- severity モデル（**重大 / 設計不明 / 保守危険**）と CWE/MISRA マッピング
- **diff のみ走査**（`creview --preset pr` で touch された行だけチェック）
- AI 補強（曖昧な検出を Claude に判定させる）

```sh
$ creview --local-only --preset pr --format json src/feature.c
```

「local-only」モードは Claude API key 不要で、ルールベースのみで動きます。

## L3: CSAF — 監査レベル

最後に、libclang AST + 依存グラフベースで risk A/B/C を自動昇格する CSAF。これは個人開発の PR レビューには重すぎますが、リリース前の監査・MISRA C 準拠チェック・古い大規模コードベースの初回監査に使います。

## 検証用コード（改訂版）

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* --- REQ-S1: 初期化フラグの set 漏れ (CWE-665 Improper Initialization) ---
   静的記憶域は ISO C により 0 に自動初期化される（不定値ではない）。
   問題は init() を呼ばないと意味的な「初期化済み」状態にならない設計バグ。
   CWE-457 (auto 変数の不定値読み = UB) ではない。 */
static int g_initialized;

void init(void) {
    g_initialized = 1;  /* main から呼び忘れる設計バグの代表例 */
}

/* --- REQ-M1a: NULL契約未表明 ---
   期待形:
     void process_data(char *data) __attribute__((nonnull(1)));
   こうすれば process_data(NULL) は -Wnonnull でコンパイル時に警告される。 */
void process_data(char *data) {
    printf("Data: %s\n", data);
    int len = strlen(data);
    printf("Length: %d\n", len);
}

/* --- REQ-M1b: 戻り値検査の強制不足 ---
   期待形:
     int validate_data(const char *data)
         __attribute__((warn_unused_result, nonnull(1)));
   呼び出し側で戻り値を捨てると -Werror=unused-result で落ちる。
   MISRA C 2012 Dir 4.7 / CWE-252。 */
int validate_data(const char *data) {
    if (data == NULL) return -1;
    return 0;
}

/* --- REQ-M2: auto 変数の真の未初期化使用 (CWE-457) --- */
int calculate_sum(int count) {
    int total;  /* 不定値 = UB */
    for (int i = 0; i < count; i++) total += i;
    return total;
}

int main(void) {
    /* REQ-S1: init() 呼び忘れ。g_initialized は 0 のまま */
    if (g_initialized) printf("Already initialized\n");

    process_data(NULL);   /* REQ-M1a: nonnull 契約なしで警告も出ない */
    validate_data(NULL);  /* REQ-M1b: 戻り値破棄をコンパイラが咎めない */
    return 0;
}
```

## CWE/MISRA マッピング（訂正版）

| 検出パターン | CWE / MISRA | 説明 |
| --- | --- | --- |
| **auto 変数の未初期化使用 (= 不定値読み = UB)** | CWE-457 | 自動記憶域の変数を初期化せず読むと不定値・未定義動作 |
| **静的記憶域フラグの set 漏れ** | **CWE-665** Improper Initialization | 値は 0 自動初期化されるが、フラグ運用の手続きが抜けている設計バグ。CWE-457 ではない |
| **NULL 契約未表明 (nonnull 属性なし)** | CWE-476 周辺 | 呼び出し側に契約を伝えていない。`__attribute__((nonnull))` で表明すべき |
| **戻り値検査強制不足** | **CWE-252** + MISRA C 2012 **Dir 4.7** | エラーを返す関数に `__attribute__((warn_unused_result))` を付け、戻り値破棄を `-Werror=unused-result` で禁止 |
| NULL ポインタ参照 | CWE-476 | NULL を deref するとクラッシュ |
| メモリリーク | CWE-401 | free 忘れ |
| 二重 free | CWE-415 | double-free。`-fanalyzer` で多くは取れる |
| use-after-free | CWE-416 | 解放済みポインタの参照 |
| スタックオーバーフロー | CWE-121 | strcpy 等で長さ未検証 |
| 境界チェック不足 | CWE-129 | 配列インデックス未検証 |
| `sizeof(pointer)` 配列長誤用 | CWE-467 | 関数パラメータの配列形は decay する。ローカル配列宣言なら正しく配列サイズを返す |
| ハードコード定数 (マジックナンバー) | CWE-547 | allowlist `{-1,0,1,2}` 以外は flag、context で severity |

## creview の使い方（CLI 実行例 — 訂正後）

```
L29 【重大】: 未初期化変数totalを使用。auto 変数の不定値読み = UB
L33 【重大】: validate_data に warn_unused_result 属性なし。戻り値破棄を禁止できない (CWE-252 / Dir 4.7)
L40 【保守危険】: process_data に nonnull 属性なし。NULL 引数を -Wnonnull で検出できない
L45 【保守危険】: g_initialized は 0 自動初期化される。CWE-457 ではなく CWE-665 系のフラグ set 漏れ
L52 【重大】: マジックナンバー256。malloc 引数 (#define / enum 化推奨)
L62 【重大】: マジックナンバー100。配列宣言サイズ
```

前回の記事に出していた `L76 【重大】: sizeof(buf)はポインタサイズ` は **誤検出** だったので外しています。`char buf[100];` のような **ローカル配列宣言** に対する `sizeof(buf)` は配列サイズ (=100) を返すのが C 標準の挙動で、警告対象ではありません。pointer サイズになるのは関数パラメータの配列形（decay する）と明示的な pointer 宣言だけです。

## 4 段階の使い分け表

| 場面 | 使うツール | 使いどころ |
| --- | --- | --- |
| **L0: ビルド時 (まずここ)** | gcc / clang の警告 + `-Werror` | `-Wall -Wextra -Wpedantic -Werror -fanalyzer` (GCC) / `-Weverything -Werror` (Clang) |
| L1: PR チェック | clang-tidy / cppcheck | OSS の主流チェックを CI で強制 |
| L2: PR レビュー | creview | プロジェクト固有 / 日本語 / severity / CWE/MISRA / `--preset pr` で diff のみ |
| L2': 自己チェック | c-review-ai | ブラウザでコピペ。環境構築不要 |
| L3: 監査 | CSAF | libclang AST + 依存グラフで risk A/B/C 自動昇格 |

## 運用ルール（これが本記事の核）

> **L0 の警告がゼロにならない PR は、人間レビューに入らない**
>
> L1 の clang-tidy / cppcheck も CI で `-Werror` 相当で落とす。
>
> L2 (creview) はその後段。L0/L1 の代替として走らせない。

このルールが効いているチームでは、人間レビュアの「またこれか」コストが大幅に下がります。creview のような自作ツールに着手するのは、L0/L1 を運用に乗せ切ったあとです。

## まとめ

- 前回の記事で creview から書き始めてしまったのは構成ミスでした。コンパイラ警告 (L0) を CI で強制するのが第一手で、creview はその後段です
- 静的記憶域の自動 0 初期化 (CWE-665) と auto 変数の不定値読み (CWE-457) は別物です。混同しないこと
- NULL チェック漏れの第一選択は `__attribute__((nonnull))` でコンパイル時契約。エラーを返す関数には必ず `__attribute__((warn_unused_result))`
- マジックナンバー検出は原則統一: `{-1,0,1,2}` 以外を全箇所 flag、context で severity
- `sizeof` はローカル/ファイルスコープ配列宣言・構造体メンバ配列・typedef 配列なら配列サイズを返す。pointer サイズになるのは関数パラメータの decay と明示 pointer 宣言のみ

コメントで指摘いただいたみなさん、改めてありがとうございました。

---

### 試すリンク

- **ブラウザで気軽に試す → c-review-ai**（自社プロダクト / MIT / 無料）
  → https://github.com/ponfreelance/c-review-ai
- **CLI で日常運用 → creview**（自社プロダクト / 無料）
  → https://github.com/ponfreelance/creview
- **業務・監査レベル → CSAF**（自社プロダクト / ¥4,980）
  → https://cutt.ly/ItKo4MPY

2026年5月時点の情報です。creview / CSAF のバージョンは随時更新しています。
