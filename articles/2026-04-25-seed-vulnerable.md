---
title: "C言語の危険コードを静的解析ツールcreviewで検出する"
emoji: "🛡️"
type: "tech"
topics: ["c", "embedded", "codereview"]
published: true
---

自分はC言語23年のフリーランスで、組み込みのコードレビューに入ることが多いです。若手のPRに入って一番「またこれか」となるのが、NULL未チェック/未初期化/制御不能な再帰/goto前方ジャンプの4つです。どれも一度教えれば直るのに、週をまたいだ別のPRでまた出てくる。人間のレビューワで全部拾うのは現実的じゃないと割り切って、自作ツールに押し付けることにしました。

## Situation / Complication / Question / Answer

- **状況**: C言語のコードレビューで、若手への指摘の7〜8割が同じパターンに収束します。NULL、未初理化、再帰、goto。
- **問題**: 毎週のPRで同じ指摘を書くのは、自分にとってもレビューされる側にとっても地獄です。しかも「疲れた日」に見逃すと、本番に流れ込みます。
- **問い**: 人間が覚えておくべきことと、機械に任せていいことを分けるには、どの粒度で自動化するのが現実的か？
- **答え**: creviewで「再発する定型的な危険」を全部拾い、人間のレビューは「設計判断」と「ドメイン知識」に集中するように分業しました。

## 検証用コード

```c
/**
 * vulnerable.c
 * テスト用サンプル — 意図的に脆弱なCコード
 * 全検出パターンを含む
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* --- REQ-S1: フラグ未初期化 --- */
static int g_initialized;  /* 未初期化のまま使用される */

/* --- REQ-M1: NULLチェック漏れ --- */
void process_data(char *data) {
    /* data が NULL かどうかチェックしていない */
    printf("Data: %s\n", data);
    int len = strlen(data);
    printf("Length: %d\n", len);
}

/* --- REQ-M2: 未初期化変数 --- */
int calculate_sum(int count) {
    int total;  /* 未初期化 */
    for (int i = 0; i < count; i++) {
        total += i;
    }
    return total;
}

/* --- REQ-M3: malloc戻り値未確認 --- */
char* create_buffer(size_t size) {
    char *buf = malloc(size);
    /* malloc の戻り値チェックなし */
    memset(buf, 0, size);
    return buf;
}

/* --- REQ-M4: free忘れ (メモリリーク) --- */
void leaky_function(void) {
    char *temp = malloc(256);
    if (temp == NULL) return;
    strcpy(temp, "hello");
    printf("%s\n", temp);
    /* free(temp) がない — メモリリーク */
    return;
}

/* --- REQ-M5: 二重free --- */
void double_free_example(void) {
    int *p = malloc(sizeof(int));
    if (p == NULL) return;
    *p = 42;
    free(p);
    /* ... 他の処理 ... */
    free(p);  /* 二重 free */
}

/* --- REQ-A1: buffer overflow --- */
void buffer_overflow_example(void) {
    char small_buf[10];
    strcpy(small_buf, "This string is way too long for the buffer");
}

/* --- REQ-A2: 境界チェック不足 --- */
int access_array(int *arr, int index) {
    /* index の範囲チェックなし */
    return arr[index];
}

/* --- REQ-R1: 戻り値未確認 --- */
void file_operations(void) {
    FILE *fp = fopen("/tmp/test.txt", "r");
    /* fopen の戻り値チェックなし */
    char buf[100];
    fgets(buf, sizeof(buf), fp);
    fclose(fp);
}

/* --- REQ-S2: 状態遷移抜け --- */
typedef enum {
    STATE_INIT,
    STATE_RUNNING,
    STATE_PAUSED,
    STATE_STOPPED
} State;

void handle_state(State state) {
    switch (state) {
        case STATE_INIT:
            printf("Initializing\n");
            break;
        case STATE_RUNNING:
            printf("Running\n");
            break;
        /* STATE_PAUSED と STATE_STOPPED が処理されていない */
    }
}

int main(void) {
    /* g_initialized チェックなしで使用 */
    if (g_initialized) {
        printf("Already initialized\n");
    }

    process_data(NULL);  /* NULL を渡す */

    int sum = calculate_sum(10);
    printf("Sum: %d\n", sum);

    char *buf = create_buffer(1024);
    /* buf が NULL の可能性を考慮していない */
    printf("Buffer: %p\n", (void*)buf);

    leaky_function();
    double_free_example();
    buffer_overflow_example();

    int arr[] = {1, 2, 3};
    int val = access_array(arr, 100);  /* 範囲外アクセス */
    printf("Value: %d\n", val);

    file_operations();
    handle_state(STATE_PAUSED);  /* 未処理の状態 */

    return 0;
}
```

## CWE/MISRA マッピング

| 検出パターン | CWE 番号 | 說明 |
| --- | --- | --- |
| NULL ポインタ参照 | CWE-476 | NULL ポインタを参照するとプログラムがクラッシュする |
| 未初期化変数 | CWE-457 | 変数を初期化せずに使用すると不定値になる |
| malloc 戻り値未確認 | CWE-690 | malloc の戻り値を確認せずに使用するとクラッシュする |
| メモリリーク | CWE-401 | メモリを解放せずに続けて使用するとメモリリークになる |
| 二重 free | CWE-415 | 二重に free するとプログラムがクラッシュする |
| バッファオーバーフロー | CWE-121 | バッファのサイズを超えて書き込むとプログラムがクラッシュする |
| 境界チェック不足 | CWE-129 | 配列のインデックスをチェックせずにアクセスするとクラッシュする |
| 戻り値未確認 | CWE-252 | 関数の戻り値を確認せずに使用するとクラッシュする |

## creview の使い方（CLI 実行例）

```
L26 【重大】: 未初期化変数totalを使用(24行目で宣言、初期化なし)。不定値
L33 【重大】: bufにNULLチェックなし。malloc/calloc/realloc失敗時クラッシュ
L35 【重大】: bufのNULL未検証でmemset実行。NULLポインタ渡しでクラッシュ
L41 【保守危険】: マジックナンバー256。定数定義なしで意味不明・変更困難
L43 【重大】: strcpy使用。コピー元長未検証でオーバーフロー可能
L53 【保守危険】: マジックナンバー42。定数定義なしで意味不明・変更困難
L54 【保守危険】: free(p)直後にNULL代入なし。dangling pointer残存
L56 【重大】: pの二重free。54行目で既にfree済み
L56 【保守危険】: free(p)直後にNULL代入なし。dangling pointer残存
L62 【重大】: strcpy使用。コピー元長未検証でオーバーフロー可能
L76 【重大】: sizeof(buf)はポインタサイズ(4/8byte)を返す。配列長にならない
L111 【保守危険】: マジックナンバー1024。定数定義なしで意味不明・変更困難
```

【重大】: プログラムの正常な動作に影響を与える可能性のある重大な問題です。
【設計不明】: 設計の不明確さや不整合性が原因で生じる問題です。
【保守危険】: 将来の保守性を低下させる可能性のある問題です。

## 3段階の使い分け

| 場面 | 使うツール | 使いどころ |
| --- | --- | --- |
| 自己チェック | c-review-ai | コピペで簡単に試せる |
| レビュー | creview | diff 行のみをチェックする `--preset pr` |
| CI | creview | GitHub Code Scanning 連携 `--format sarif` |
| 監査 | CSAF | libclang AST + 依存グラフで risk A/B/C 自動昇格 |

## まとめ

- creview を使うことで、C 言語のコードから重大な問題を静的解析で検出できます。
- CWE 番号を使用することで、各問題の詳細な情報を参照できます。
- 3 つのツール (c-review-ai / creview / CSAF) を使い分けることで、開発の各段階で効率的にコードをチェックできます。

---

## 訂正履歴（2026-05-09 追記）

公開後、コメント欄で 5 件の重要な指摘をいただきました。以下、訂正します。**コメントいただいた方々ありがとうございました**。

1. **`static int g_initialized;` を「未初期化変数 / CWE-457」と紹介していた件**
   静的記憶域期間を持つ変数は ISO C により 0 に自動初期化されることが保証されており、不定値ではありません。CWE-457 (Use of Uninitialized Variable) は **auto 変数**を初期化せず読み出して未定義動作になるケースに限定されます。今回のケースは「初期化済みフラグを表すつもりが、`g_initialized = 1` を立てる手続きが抜けている」設計バグで、正しくは **CWE-665 (Improper Initialization)** に分類すべきでした。

2. **REQ-M1 (NULL チェック漏れ) のリコメンドが弱かった件**
   組み込みでの第一選択は「関数本体にランタイム NULL チェックを書く」ではなく、**`__attribute__((nonnull(N)))` でコンパイル時契約を表明する**ことでした。呼び出し側が NULL を渡したら `-Wnonnull` で**コンパイル時に**落とすべきです。エラーコードを返す関数には `__attribute__((warn_unused_result))` を必ず付けて戻り値破棄を `-Werror=unused-result` で禁止する（**MISRA C 2012 Dir 4.7 / CWE-252**）。`assert` は libc の場合 release で `abort()` に展開されるため組み込みでそのまま使うべきではなく、プロジェクト定義の `MY_ASSERT` で debug=break / release=フォールトハンドラ→セーフ復帰、にマップする方針が正解でした。

3. **`L76 【重大】: sizeof(buf)はポインタサイズ(4/8byte)を返す。配列長にならない` は creview の false positive**
   ```c
   char buf[100];
   fgets(buf, sizeof(buf), fp);
   ```
   `buf` はローカル**配列**として宣言されているので、`sizeof(buf)` は配列全体のサイズ（=100）を返します。pointer サイズになるのは関数パラメータの配列形（C 標準で decay する）と明示的な pointer 宣言のときだけです。creview の検出ロジック（配列/ポインタ/decay 区別）を直す必要があり、その仕様を別途整理しました。

4. **マジックナンバー検出が不整合だった件**
   `malloc(256)` や `*p = 42` は flag していたのに `char buf[100];` や `calculate_sum(10)` は flag されておらず、原則がありませんでした。正しい統一ルールは「整数リテラル N が allowlist `{-1, 0, 1, 2}` 以外なら**全箇所で flag**、context (malloc 引数 / 配列宣言 / ループ上限 / bitmask / 関数引数 / 代入) で severity を切り替える」です。プロジェクト単位の allowlist 拡張は `.creviewrc.json` で。

5. **記事の構成が「creview から始まっている」のがそもそも誤りだった件**
   「`-Wall -Wextra -Werror -fanalyzer` で取れるものが多いので、それらが消えない PR はレビューしないルール化が先では？」というご指摘の通りで、4 層モデル（**L0: コンパイラ警告 → L1: clang-tidy/cppcheck → L2: creview → L3: CSAF**）に位置づけ直すべきでした。creview は L0/L1 の代替ではなく、その後段で取れない高次パターン（プロジェクト固有 / 日本語ナレッジ / severity モデル / CWE/MISRA / diff のみ走査 / AI 補強）を担うものです。**運用ルールとして「警告ゼロにならない PR はレビューに入らない」を CI で強制する**のが第一手。

これら 5 件を全反映した **改訂版を別記事として 2026-05-09 付で出し直します**。本記事は記録として残し、改訂版で正しい内容を読んでください。

---

### 試すリンク

- **ブラウザで気軽に試す → c-review-ai**（自社プロダクト / MIT / 無料）
  → https://github.com/ponfreelance/c-review-ai
- **CLI で日常運用 → creview**（自社プロダクト / 無料）
  → https://github.com/ponfreelance/creview
- **業務・監査レベル → CSAF**（自社プロダクト / ¥4,980）
  → https://cutt.ly/ItKo4MPY

2026年4月時点の情報です。creview / CSAF のバージョンは随時更新しています。
