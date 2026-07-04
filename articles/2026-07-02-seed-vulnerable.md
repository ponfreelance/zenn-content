---
title: "creview で「危険コード20件」を検出：CWE対応表付き実践レビュー"
emoji: "🛡️"
type: "tech"
topics: ["c", "embedded", "codereview"]
published: true
---

C言語23年、自分はフリーランスで組み込みのコードレビューに入ることが多いです。今回、意図的に脆弱なコードを1本用意して、自作の静的解析ツール creview に食わせてみました。結果は20件。malloc戻り値未チェック、二重free、strcpyのバッファオーバーフロー、状態遷移の抜け漏れ——どれも「見たことある」パターンばかりで、実際に過去のPRレビューで指摘した記憶が何個も蘇りました。特にdangling pointerの指摘は、自分が3年前に見逃して本番障害になった件とほぼ同じ形をしていて、地味に刺さりました。

## SCQA

- **状況**: C言語の脆弱性パターンは種類が絞られているのに、レビューのたびに同じ指摘を人力で繰り返しています。NULL未チェック、未初期化変数、malloc失敗の未処理、二重free、strcpyのオーバーフロー、この5つだけで指摘の大半を占めます。
- **問題**: これらは「知っていれば防げる」類の欠陥ですが、大きい関数やswitch文の奥に埋もれると、レビューワの目視では拾いきれません。特にfree後のNULL代入忘れ（dangling pointer）は、その場では動くのでレビューでもすり抜けやすいです。
- **問い**: コンパイラの警告で拾える範囲と、プロジェクト固有ルールとして機械にチェックさせるべき範囲は、どこで線を引くべきか？
- **答え**: まずコンパイラ警告をゼロにする運用を敷いた上で、creviewでCWEマッピング済みの危険パターンを機械的に洗い出し、人間は設計判断（状態遷移の抜け漏れの妥当性、マジックナンバーの命名要否）に集中する形に分業しました。

## L0: まずコンパイラ警告を全部潰す

creview の話に入る前に、`-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）または `-Wall -Wextra -Werror -Weverything`（Clang）を有効にしてビルドが通るか確認してください。NULL引数をnonnull関数に渡す／未初期化auto変数を読む／戻り値を破棄する／自明なNULL deref／use-after-free／境界外配列アクセス（静的決定可なもの）／switch case抜け、これらは現代のコンパイラが標準で拾います。

実際、今回の検証コードを `gcc -Wall -Wextra -fanalyzer` に通すと、`calculate_sum` の未初期化変数 `total` は `-Wmaybe-uninitialized` 相当で引っかかりますし、`-fanalyzer` はmalloc戻り値未チェックの一部も検出します。これらが警告ゼロにならないPRは、creview を走らせる前にそもそもレビュー対象にしないルールにしてしまうのが第一手です。

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

/* --- REQ-S1: 初期化フラグの set 漏れ (CWE-665 Improper Initialization) --- */
/* 値は ISO C により 0 に自動初期化される（不定値ではない）。
   問題は init() を呼ばないと g_initialized が 1 にならず、
   フラグの意味的な「未初期化」状態のまま判定される設計バグ。
   静的記憶域の自動 0 初期化と CWE-457 (auto 変数の不定値読み = UB)
   は別物なので混同しないこと。 */
static int g_initialized;

void init(void) {
    /* main から呼び忘れる設計バグの代表例。
       本来ここで 1 を立て、main では if (g_initialized) で
       初期化済み判定するつもりだった。 */
    g_initialized = 1;
}

/* --- REQ-M1a: NULL契約未表明 (nonnull 属性なし) --- */
/* この関数は data を unconditional に dereference するが、
   呼び出し側に「NULL を渡すな」を伝えるコンパイル時契約
   (__attribute__((nonnull))) が宣言にない。呼び出し側のミスを
   コンパイラが検出できない。
   期待形:
     void process_data(char *data) __attribute__((nonnull(1)));
   こうすれば process_data(NULL) は -Wnonnull で警告される。 */
void process_data(char *data) {
    printf("Data: %s\n", data);
    int len = strlen(data);
    printf("Length: %d\n", len);
}

/* --- REQ-M1b: 戻り値検査の強制不足 (warn_unused_result 属性なし) --- */
/* エラーコードを返すが、呼び出し側が return 値を捨てても
   コンパイラは何も言わない。NULL を防ぐコードを書いても無視される。
   期待形:
     int validate_data(const char *data)
         __attribute__((warn_unused_result, nonnull(1)));
   呼び出し側で戻り値を捨てると -Werror=unused-result で落ちる。
   MISRA C 2012 Dir 4.7 / CWE-252 に対応。 */
int validate_data(const char *data) {
    if (data == NULL) return -1;
    return 0;
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
    /* REQ-S1: init() を呼び忘れる設計バグ。g_initialized は 0 のまま。
       値は不定値ではなく 0 だが、フラグとしては未確立 = 設計上の
       「未初期化」状態。CWE-665 (Improper Initialization)。 */
    if (g_initialized) {
        printf("Already initialized\n");
    }

    process_data(NULL);  /* REQ-M1a: nonnull 契約がないので警告も出ない */
    validate_data(NULL); /* REQ-M1b: 戻り値破棄してもコンパイラが咎めない */

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

| 検出パターン | 該当行 | CWE | 事故の形 |
|---|---|---|---|
| 静的記憶域フラグの set 漏れ | L23 | CWE-665 | `init()` を呼び忘れても `g_initialized` は0のまま残る。フラグの意味が確立しない設計バグ |
| NULL契約未表明 | L34, L97 | CWE-476周辺 | `nonnull` 属性がなく、呼び出し側のNULL渡しをコンパイラが検出できない |
| 戻り値検査強制不足 | L48 | CWE-252 / MISRA C 2012 Dir 4.7 | `warn_unused_result` がなく、エラーコード破棄をコンパイラが咎めない |
| auto変数の未初期化使用 | L57 | CWE-457 | `total` が不定値のまま加算され、結果が実行ごとに変わる |
| malloc戻り値未チェック | L64 | CWE-690 | `malloc` 失敗時に `buf` がNULLのまま返る |
| NULL未検証でmemset | L66 | CWE-476 | NULLポインタへの書き込みでクラッシュ |
| マジックナンバー（バッファサイズ） | L72, L92, L106 | CWE-547 | サイズ変更時に複数箇所を漏れなく直せず、オーバーフローの温床になる |
| strcpyによるオーバーフロー | L74, L93 | CWE-121 | コピー元長未検証でスタックバッファオーバーフロー |
| マジックナンバー（リテラル） | L84, L142, L145, L153, L154 | CWE-547 | 意味不明な数値がハードコードされ、変更漏れ・誤読を誘発 |
| free後のNULL代入なし | L85, L87 | CWE-416（UAFの温床） | dangling pointerが残り、後続コードでuse-after-freeに発展しうる |
| 二重free | L87 | CWE-415 | 同じポインタを2回freeし、ヒープ破壊やクラッシュを引き起こす |
| グローバル変数の直接更新 | L23 | CWE番号不明 | 競合状態や意図しない副作用のリスク（マルチスレッド化時に顕在化） |

## creview の使い方（CLI 実行例）

```
$ creview scan vulnerable.c --preset security --format markdown
```

```
L23 【保守危険】: グローバル変数g_initializedを関数内で直接更新。競合・副作用リスク
L34 【重大】: ATTR-NONNULL-001: process_data() の data を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L48 【設計不明】: ATTR-WUR-001: validate_data() に warn_unused_result 属性なし。呼び出し側が戻り値破棄しても検出できない (CWE-252 / MISRA C 2012 Dir 4.7)
L57 【重大】: 未初期化変数totalを使用(55行目で宣言、初期化なし)。不定値
L64 【重大】: bufにNULLチェックなし。malloc/calloc/realloc失敗時クラッシュ
L66 【重大】: bufのNULL未検証でmemset実行。NULLポインタ渡しでクラッシュ
L72 【重大】: マジックナンバー256 (バッファサイズ) — allowlist 外。#define / enum / .creviewrc.json で命名 or 許可
L74 【重大】: strcpy使用。コピー元長未検証でオーバーフロー可能
L84 【保守危険】: マジックナンバー42 (リテラル) — allowlist 外。#define / enum / .creviewrc.json で命名 or 許可
L85 【保守危険】: free(p)直後にNULL代入なし。dangling pointer残存
L87 【重大】: pの二重free。85行目で既にfree済み
L87 【保守危険】: free(p)直後にNULL代入なし。dangling pointer残存
L92 【重大】: マジックナンバー10 (バッファサイズ) — allowlist 外。#define / enum / .creviewrc.json で命名 or 許可
L93 【重大】: strcpy使用。コピー元長未検証でオーバーフロー可能
L97 【重大】: ATTR-NONNULL-001: access_array() の arr を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L106 【重大】: マジックナンバー100 (バッファサイズ) — allowlist 外。#define / enum / .creviewrc.json で命名 or 許可
L142 【保守危険】: マジックナンバー10 (リテラル) — allowlist 外。#define / enum / .creviewrc.json で命名 or 許可
L145 【保守危険】: マジックナンバー1024 (リテラル) — allowlist 外。#define / enum / .creviewrc.json で命名 or 許可
L153 【保守危険】: マジックナンバー3 (リテラル) — allowlist 外。#define / enum / .creviewrc.json で命名 or 許可
L154 【保守危険】: マジックナンバー100 (リテラル) — allowlist 外。#define / enum / .creviewrc.json で命名 or 許可
```

3段階ラベルの読み方は次の通りです。

- **【重大】**: 実行時クラッシュ・メモリ破壊・未定義動作に直結する箇所。マージ前に必ず直す。今回だと未初期化変数、malloc未チェック、二重free、strcpyのオーバーフローがここに入ります。
- **【設計不明】**: 動作は壊れないが、コンパイラに契約を伝えていない箇所。`warn_unused_result` の欠落がこれで、機能的には問題なくても事故の芽を放置している状態です。
- **【保守危険】**: 今すぐ壊れないが、将来の変更で壊れやすい箇所。マジックナンバーやdangling pointer残存（NULL代入忘れ）はここに分類され、レビューでの優先度は落としつつも放置しない扱いにしています。

自分の運用では、【重大】はマージブロック、【設計不明】は次スプリントで属性付与のタスク化、【保守危険】はリファクタのバックログに積む、という3段階で扱っています。マジックナンバーだけ全部潰そうとすると工数が膨らむので、ここは意図的に優先度を落としています。

## 3段階の使い分け

| 場面 | 使うツール | 使いどころ |
|------|-----------|-----------|
| L0: ビルド時（まずここ） | gcc / clang の警告 + `-Werror` | `-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）/ `-Weverything -Werror`（Clang）。警告ゼロにならないPRはそもそもレビューに入らないルール化 |
| L1: PRチェック | clang-tidy / cppcheck | OSSの主流チェックをCIで強制 |
| L2: PRレビュー | creview（CLI、自社プロダクト） | プロジェクト固有パターン、日本語ナレッジ、severityモデル（重大/設計不明/保守危険）、CWE/MISRAマッピング、`--preset pr` でdiffのみ、AI補強 |
| L2': 自己チェック（PR投げる前） | c-review-ai（ブラウザ版、自社プロダクト） | 環境構築不要、コピペで試せる軽量版 |
| L3: チーム監査・MISRA準拠 | CSAF（自社プロダクト） | libclang AST + 依存グラフでrisk A/B/C自動昇格 |

「L0の警告がゼロにならないPRはレビューに入らない」をCIで強制する運用です。以前、この運用を敷く前は `-fanalyzer` が拾える未初期化変数まで人間がレビューで指摘していて、時間の無駄でした。L0を素通りしたPRを creview で叩いても、消せた警告まで人間が読むことになって意味がありません。

余談ですが、今回の検証で `access_array` へのNULL契約未表明（L97）を creview が拾った時、最初は「配列引数にnonnullは大げさでは」と思いました。ただ `main` 側で `access_array(arr, 100)` と範囲外インデックスを渡している実例（L154付近）を見て、境界チェックの欠如とセットで見ると確かに危険な組み合わせだと納得しました。1つのパターン単体だと軽視しがちですが、組み合わせで見ると深刻度が上がる典型例です。

## まとめ

- CWE-457（auto変数未初期化）とCWE-665（静的記憶域フラグのset漏れ）は別物であり、コード中のコメントでも混同しない設計にしておくと後のレビューで迷わない
- 二重free（CWE-415）とdangling pointer残存（CWE-416の温床）は同じ `free` 呼び出し周辺で同時に検出されることが多く、free直後のNULL代入をコーディング規約に入れるだけで片方は防げる
- マジックナンバー検出はバッファサイズ・ループ上限・リテラル代入まで一律にflagされるため、`.creviewrc.json` でallowlistを運用しないと指摘が20件中7件を占めるような偏りが出る
- 【重大】【設計不明】【保守危険】の3段階はマージブロック／次スプリントタスク化／バックログ、という運用上の優先度分けにそのまま使える
- L0（コンパイラ警告ゼロ）を通過したコードにだけ creview を回す運用にしないと、警告ゼロで消せる指摘まで人間が読む羽目になる


---

### 試すリンク

- **ブラウザで気軽に試す → c-review-ai**（自社プロダクト / MIT / 無料）
  → https://github.com/ponfreelance/c-review-ai
- **CLI で日常運用 → creview**（自社プロダクト / 無料）
  → https://github.com/ponfreelance/creview
- **業務・監査レベル → CSAF**（自社プロダクト / ¥4,980）
  → https://cutt.ly/ItKo4MPY

2026年4月時点の情報です。creview / CSAF のバージョンは随時更新しています。
