---
title: "creviewでprintf書式文字列脆弱性CWE-134を9件検出した話"
emoji: "🛡️"
type: "tech"
topics: ["c", "embedded", "codereview"]
published: true
---

---

C言語23年、自分はフリーランスで組み込みとサーバーサイドのコードレビューに入ることが多いです。「ログ出力くらい安全だろう」という油断が一番厄介で、`printf(user)` を素通りさせた PR を自分も過去に1件見逃しています。指摘したのが結合テスト前日で、修正が差し戻しになって深夜まで対応した苦い記憶があります。書式文字列脆弱性（CWE-134）は、コードを読んで「ああ、これは `%s` を忘れただけ」と思いがちですが、攻撃者が `%x%x%x%n` を仕込めばスタックの任意アドレスを書き換えられる、れっきとした任意コード実行の入口です。

---

## この記事の前提（SCQA）

- **状況**: ログ系のユーティリティ関数に `printf(user_input)` 形式のコードが混入するのは、C言語プロジェクトでは今でも頻繁に起きます。書き慣れた開発者でも「ログ出力は副作用が薄い」という感覚で書いてしまい、レビューで素通りします。
- **問題**: 書式文字列脆弱性は実行時に再現条件を作らないと見えないため、静的解析なしでは PR レビューだけで全件拾うのは現実的ではありません。さらに NULL 未検査が同時に存在すると、攻撃面がさらに広がります。
- **問い**: `printf` 系関数の書式文字列引数に変数を直接渡すパターンを、人間が毎 PR 目視で追わずに機械的に検出するにはどうすればいいか？
- **答え**: 自社プロダクトの静的解析ツール **creview** を `--preset security` で走らせると、CWE-134 に該当するパターン全件と、NULL 未検査・戻り値未確認を含む9件を一括検出できました。本記事でその検証過程を再現します。

---

## L0: まずコンパイラ警告を全部潰す

creview の話に入る前に、`-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）または `-Wall -Wextra -Werror -Weverything`（Clang）を有効にしてビルドが通るか確認してください。

今回の検証コードで GCC 13 に `-Wformat-security` を付けると、`printf(user)` と `fprintf(fp, msg)` は **`-Wformat-security` 警告**として拾えます。つまり L0 で取れる部分もあります。それでも `snprintf(out, n, user_fmt)` は GCC の警告が出ない場合があり、NULL 未検査・戻り値未確認もコンパイラだけでは網羅できません。

**警告ゼロにならない PR はそもそも creview を走らせる前にレビュー対象にしない**、というルールを CI で強制するのが第一手です。L0 を素通りした PR を creview で叩くと、コンパイラが拾えた警告まで人間が読むことになって時間を無駄にします。

```bash
gcc -Wall -Wextra -Wpedantic -Werror -Wformat-security -fanalyzer \
    -o vuln_log vuln_log.c
```

L0 で警告ゼロを確認してから、次のステップへ進みます。

---

## 検証用コード

今回 creview に食わせたコードをそのまま掲載します。ログ出力ユーティリティとしてよくある4パターンを1ファイルに集約しています。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* printf にユーザー入力を直接渡す（書式文字列脆弱性） */
void log_user_input(const char *user) {
    printf(user);                        /* 検出すべき: printf("%s", user) にすべき */
}

/* fprintf も同様 */
void log_to_file(FILE *fp, const char *msg) {
    fprintf(fp, msg);                    /* 検出すべき */
}

/* puts は実は安全だが、似た形の罠 */
void log_short(const char *msg) {
    puts(msg);                           /* puts は安全（書式解釈しない）が紛らわしい */
}

/* snprintf も書式引数を変数で渡すと危険 */
void format_log(char *out, size_t n, const char *user_fmt) {
    snprintf(out, n, user_fmt);          /* 検出すべき */
}
```

4関数のうち `puts` だけは書式解釈をしないので CWE-134 の直接対象ではありません。ただし creview は「見た目が似ていて誤解を招く」という観点から L17 を報告しています（後述）。

---

## creview の使い方（CLI 実行例）

自社プロダクトの **creview**（CLI 版）は以下のように実行します。`--preset security` はバッファ操作・書式文字列・NULL 検査・戻り値確認を重点的にチェックするプリセットです。

```bash
creview --preset security --format markdown vuln_log.c
```

出力フォーマットは `--format json` / `--format markdown` / `--format sarif` から選べます。CI に組み込む場合は `--format sarif` で GitHub Code Scanning へ流すのが定番です。`--format json` は downstream スクリプトで集計したいときに使います。

### 実際の検出ログ（抜粋・そのまま）

```
L6 【重大】: ATTR-NONNULL-001: log_user_input() の user を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L7 【重大】: printfの書式文字列に変数userを直接使用。攻撃者制御で任意メモリ読み書き可能
L11 【重大】: ATTR-NONNULL-001: log_to_file() の fp を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L12 【重大】: fprintfの書式文字列に変数msgを直接使用。攻撃者制御で任意メモリ読み書き可能
L16 【重大】: ATTR-NONNULL-001: log_short() の msg を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L17 【重大】: putsの書式文字列に変数msgを直接使用。攻撃者制御で任意メモリ読み書き可能
L21 【重大】: ATTR-NONNULL-001: format_log() の out を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L22 【重大】: snprintfの書式文字列に変数user_fmtを直接使用。攻撃者制御で任意メモリ読み書き可能
L22 【保守危険】: snprintf戻り値を未確認。切り詰め発生を検出できない
```

### 3段階ラベルの読み方

| ラベル | 意味 | 対応優先度 |
|--------|------|-----------|
| **【重大】** | 攻撃者が悪用すると任意メモリ読み書き・クラッシュ・権限昇格に直結するパターン。CWE 番号が確定しており、修正しなければリリース不可と判断すべき水準 | 即修正・PR ブロック |
| **【設計不明】** | コードの意図が読み取れず、安全か危険か判定できない設計上の曖昧さ。今回の検出には含まれないが、例として「malloc 戻り値を別変数に保存して元ポインタを先に free する」等が該当 | 設計者への確認必須 |
| **【保守危険】** | 即座に攻撃には使えないが、将来の変更で脆弱性になるか、バグを隠蔽するパターン。今回は `snprintf` 戻り値未確認がここに分類され、切り詰め発生を検出できないまま後続処理が進む | 次スプリントで修正 |

`puts` への L17 【重大】報告は「`puts` 自体は書式解釈しない」という意味では誤検知に近いですが、creview が「引数に外部由来の変数を渡す関数呼び出し」パターンを一律に拾う設計のためです。ここは自分も最初「`puts` まで出るのか」と30分迷いました。ルール設定でホワイトリストに入れるか、コメントで抑制するかはプロジェクト判断です。

---

## CWE/MISRA マッピング

creview が検出した9件を CWE と実害ベクターで整理します。

| 行 | 検出内容 | CWE | 事故の形（攻撃ベクター / 実害） |
|----|---------|-----|-------------------------------|
| L6 | `log_user_input()` の `user` NULL 未検査 | CWE-476: NULL Pointer Dereference | 呼び出し元が NULL を渡すと即クラッシュ（DoS）。組み込みでは watchdog reset に化ける |
| L7 | `printf(user)` — 書式文字列に変数直接渡し | CWE-134: Use of Externally-Controlled Format String | `%x%x%x%n` でスタック上の任意アドレスを書き換え。任意コード実行まで到達可能 |
| L11 | `log_to_file()` の `fp` NULL 未検査 | CWE-476: NULL Pointer Dereference | `fopen` 失敗後に戻り値チェックせず渡すと即クラッシュ |
| L12 | `fprintf(fp, msg)` — 書式文字列に変数直接渡し | CWE-134: Use of Externally-Controlled Format String | `printf` と同様。ファイル出力先でも書式解釈は行われる |
| L16 | `log_short()` の `msg` NULL 未検査 | CWE-476: NULL Pointer Dereference | `puts(NULL)` は未定義動作。glibc では SIGSEGV |
| L17 | `puts(msg)` — 変数を直接渡し（書式解釈なし） | CWE-134 周辺（`puts` 自体は非該当） | `puts` は書式解釈しないため直接の CWE-134 には当たらない。ただし類似パターンとして検出。実害は NULL 渡しによる UB |
| L21 | `format_log()` の `out` NULL 未検査 | CWE-476: NULL Pointer Dereference | 出力バッファが NULL のまま `snprintf` に渡ると未定義動作 |
| L22 | `snprintf(out, n, user_fmt)` — 書式文字列に変数直接渡し | CWE-134: Use of Externally-Controlled Format String | `%n` を含む書式文字列でメモリ書き込み。スタック破壊・情報漏洩 |
| L22 | `snprintf` 戻り値未確認 | CWE-252: Unchecked Return Value（MISRA C 2012 Dir 4.7） | 出力が切り詰められても検出できず、後続の文字列処理が壊れたデータを扱う |

### MISRA C 2012 との対応

- **Dir 4.7**: エラーを返す関数の戻り値は必ずテストすること → L22 の `snprintf` 戻り値未確認が直接違反
- **Rule 17.7**: 非 void 関数の戻り値を破棄してはならない → 同上
- **Rule 21.6**: 標準ライブラリの入出力関数は制限された文脈でのみ使用 → `printf` 系全般に関連

---

## 修正例

creview の指摘を受けて直した版を示します。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* 修正1: nonnull 属性で NULL 契約を明示 + 書式文字列を固定 */
__attribute__((nonnull(1)))
void log_user_input(const char *user) {
    printf("%s", user);                  /* 書式文字列を文字列リテラルに固定 */
}

/* 修正2: nonnull 属性 + 書式文字列固定 */
__attribute__((nonnull(1, 2)))
void log_to_file(FILE *fp, const char *msg) {
    fprintf(fp, "%s", msg);
}

/* 修正3: puts は書式解釈しないが nonnull は明示する */
__attribute__((nonnull(1)))
void log_short(const char *msg) {
    puts(msg);
}

/* 修正4: user_fmt を受け取る設計自体を廃止し、固定書式に変更
   どうしても動的書式が必要なら呼び出し元で vsnprintf を使う設計にする */
__attribute__((nonnull(1)))
void format_log(char *out, size_t n, const char *user_fmt) {
    int ret = snprintf(out, n, "%s", user_fmt); /* 書式固定 */
    if (ret < 0 || (size_t)ret >= n) {
        /* 切り詰め発生: ログに記録してフォールバック */
        out[n - 1] = '\0';
    }
}
```

`__attribute__((nonnull))` はコンパイル時に NULL リテラルを渡す呼び出しを警告に変えます。ランタイム NULL チェックを関数本体に書くのは「呼び出し側が守るべき契約を本体に押し付ける」形になり、組み込みでは穴が別の場所に移るだけです。入力境界（外部 API・通信受信・ISR）でだけランタイム検査を行い、内部関数は `nonnull` で契約化するのが筋です。

---

## 3段階の使い分け

| 場面 | 使うツール | 使いどころ |
|------|-----------|-----------|
| **L0: ビルド時（まずここ）** | gcc / clang の警告 + `-Werror` | `-Wall -Wextra -Wpedantic -Werror -Wformat-security -fanalyzer`（GCC）/ `-Weverything -Werror`（Clang）。**警告ゼロにならない PR はレビューに入らない**ルール化 |
| **L1: PR チェック（OSS）** | clang-tidy / cppcheck | `clang-tidy -checks='cert-*,bugprone-*'` 等。OSS の主流チェックを CI で強制。`-Wformat-nonliteral` でも一部拾える |
| **L2: PR レビュー（日本語ナレッジ）** | **creview（CLI）**（自社プロダクト） | `--preset security` で CWE-134 / CWE-476 / CWE-252 を一括検出。`--preset pr` で diff のみ解析して PR 単位のレビュー時間を削減。`--format sarif` で GitHub Code Scanning に流す |
| **L2': 自己チェック（PR 投げる前）** | **c-review-ai（ブラウザ版）**（自社プロダクト） | 環境構築不要。コードをコピペして即チェック。「PR 投げる前に自分で一回通す」習慣化に使う |
| **L3: チーム監査・MISRA 準拠** | **CSAF**（自社プロダクト） | libclang AST + 依存グラフで risk A/B/C を自動昇格。MISRA C 2012 Dir 4.7 / Rule 17.7 の準拠レポートを自動生成。車載・医療機器の外部監査提出に使う |

**運用ルール**: L0 の警告ゼロを CI で強制し、通過した PR だけ creview（L2）を走らせます。L0 を素通りした PR を creview で叩くと、コンパイラが拾えた警告まで人間が読む羽目になります。週をまたいだ別の PR で同じ `printf(user)` が再登場することは実際にあります。`--preset pr` で diff のみを解析するモードにしておくと、既存コードのノイズを除いて新規追加行だけを監視できるので


---

### 試すリンク

- **ブラウザで気軽に試す → c-review-ai**（自社プロダクト / MIT / 無料）
  → https://github.com/ponfreelance/c-review-ai
- **CLI で日常運用 → creview**（自社プロダクト / 無料）
  → https://github.com/ponfreelance/creview
- **業務・監査レベル → CSAF**（自社プロダクト / ¥4,980）
  → https://cutt.ly/ItKo4MPY

2026年4月時点の情報です。creview / CSAF のバージョンは随時更新しています。
