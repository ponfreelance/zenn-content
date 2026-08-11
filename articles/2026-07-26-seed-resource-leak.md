---
title: "fdクローズ漏れをマージ前に見抜く：早期returnが生む資源リーク"
emoji: "🛡️"
type: "tech"
topics: ["c", "embedded", "codereview"]
published: true
---

C言語23年、自分はフリーランスで組み込みとサーバサイドの両方のコードレビューに入ることが多いです。

fd（ファイルディスクリプタ）のクローズ漏れは、レビューで一番見逃しやすいパターンだと思っています。理由は単純で、`open` と `close` が離れた行にあり、しかも間に `if` の早期returnが挟まると、視線が「正常系のclose」だけを追ってしまうからです。以前あるPRで、`read()` が失敗した時だけ `close` を通らずにreturnするコードをレビューで見逃し、負荷試験でfdが枯渇して気づいたことがあります。あの時は原因調査に半日かかりました。今回はその教訓から、creviewでこのパターンを機械的に検出できるか検証します。

## SCQA

- **状況**: fd/socket/FILE/pipeのリソース確保と解放はセットで書くのが原則ですが、エラーハンドリングの分岐が増えるほど解放漏れの経路が増えます。
- **問題**: 人間のレビューは「正常系の最後にcloseがあるか」までは見ますが、途中の早期return全部を目で追うのは現実的ではありません。関数が長くなるほど見逃しは指数的に増えます。
- **問い**: マージ前レビューで、全ての早期returnパスに対してリソース解放が対応しているかを機械的に検証する方法はあるか？
- **答え**: creviewで全関数のリソース確保/解放のペアをフロー解析し、`--preset memory` でリーク系だけを抽出してPRチェックに組み込みます。

## L0: まずコンパイラ警告を全部潰す

creviewの話に入る前に、`-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）または `-Wall -Wextra -Werror -Weverything`（Clang）を有効にしてビルドが通るか確認してください。特にGCCの `-fanalyzer` はここ数年で `open`/`malloc` のリークをある程度検出できるようになっており、単純な「開けっぱなしで関数を抜ける」ケースは拾えることがあります。

ただし今回検証したコードのように、`socket()` や `pipe()` の結果を分岐で握りつぶすパターン、`fopen` から `fseek`/`ftell` を経由してリターンするパターンは、`-fanalyzer` でも検出精度にばらつきがあります。これらが警告ゼロになったとしても、creviewを走らせる前提のコードにはまだ足りないというのが自分の実感です。まずビルド警告ゼロを必須条件にした上で、そこから漏れるパターンをcreviewで拾う、という二段構えにしています。

## 検証用コード

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/socket.h>

/* fd を open して close せずに早期 return */
int read_first_byte(const char *path) {
    int fd = open(path, O_RDONLY);       /* 検出すべき: close なし */
    if (fd < 0) return -1;
    char b;
    if (read(fd, &b, 1) != 1) {
        return -1;                       /* 検出すべき: ここで close 漏れ */
    }
    close(fd);
    return (int)b;
}

/* socket を作って close せずに return */
int connect_to(const char *host) {
    int s = socket(AF_INET, SOCK_STREAM, 0);  /* 検出すべき: close なし */
    (void)host;
    return s;
}

/* fopen と FILE のリーク */
long file_size(const char *path) {
    FILE *fp = fopen(path, "rb");        /* 検出すべき: fclose なし */
    if (!fp) return -1;
    fseek(fp, 0, SEEK_END);
    return ftell(fp);                    /* fclose 漏れ */
}

/* pipe の片側だけ close */
void make_channel(int *r, int *w) {
    int p[2];
    pipe(p);                             /* 検出すべき: 両端 close 漏れの典型 */
    *r = p[0];
    /* p[1] を返さずに捨てる → fd リーク */
}
```

`read_first_byte` は一見「正常系はclose済み」に見えるため、レビューで一番見逃しやすい形です。`connect_to` はソケット確保後に条件分岐すらなく即returnしており、これは目視でも気づきやすいのですが、PRの差分が小さいと流されがちです。`file_size` は `fopen` の後にエラー処理はあるものの、成功パスの `fclose` を書き忘れています。`make_channel` は `pipe()` の両端のうち書き込み側 `p[1]` を呼び出し元に渡すこともclose することもせず、丸ごと捨てています。

## CWE/MISRA マッピング

| 検出パターン | CWE | 事故の形 |
|------------|-----|---------|
| `open()` 後、早期returnでclose漏れ (L8) | CWE-401 | fd枯渇。長時間稼働プロセスで `EMFILE` が発生し、以降の `open`/`socket` が全て失敗する |
| `socket()` 後、closeなしでreturn (L21) | CWE-401 | サーバプロセスで接続の度にソケットが漏れ、数時間〜数日でfd上限に到達しサービス停止 |
| `fopen()` 後、`fclose` 漏れ (L28) | CWE-401 | FILE構造体のバッファリークとfdリークの二重コスト。バッチ処理で大量ファイルを開くと顕著 |
| `pipe()` の片方だけcloseせず破棄 (L37) | CWE-401 | 書き込み側fdが漏れ、読み出し側でEOFが来ない（相手がfdを掴んだままなので永久ブロック） |
| NULL引数チェックなしで使用 (L8, L20, L27, L35) | CWE-476 | `path`/`host`/`r` がNULLで呼ばれた場合の未定義動作。契約未表明なので呼び出し側の責任範囲が曖昧 |

fd/socket/FILE/pipeのリークは全てCWE-401（Missing Release of Resource after Effective Lifetime）に分類されますが、実害の出方は異なります。`pipe()` の片側だけ捨てるパターンは特に厄介で、リークだけでなく相手プロセスのブロッキング待ちを誘発する点で単純なfd枯渇より発見が遅れます。

## creview の使い方（CLI 実行例）

```
$ creview check --preset memory --format markdown leak_sample.c
```

```
L8 【重大】: ATTR-NONNULL-001: read_first_byte() の path を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L20 【重大】: ATTR-NONNULL-001: connect_to() の host を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L21 【重大】: socket結果sに対応するcloseなし。ソケットリーク
L27 【重大】: ATTR-NONNULL-001: file_size() の path を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L28 【重大】: fopen結果fpに対応するfcloseなし。ファイルディスクリプタリーク
L35 【重大】: ATTR-NONNULL-001: make_channel() の r を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L37 【重大】: pipe(p)に対応するcloseなし。fdリーク
```

肝心の `read_first_byte` のL13（`return -1` によるclose漏れ）は今回のログには単独で行番号が出ておらず、L8の確保側で「対応するcloseなし」として検出される形でした。フロー解析ツールにありがちですが、確保箇所と解放漏れ箇所のどちらを行番号として報告するかはツールによって挙動が違うので、レビュー時は確保箇所からその関数の全returnパスを目で追う一手間は残ります。

3段階ラベルの意味は以下の通りです。

- **【重大】**: 実行時に確実に問題を起こす、または起こす可能性が高いパターン。今回の7件は全てこのラベルで、リソースリークとNULL契約未表明が該当します。
- **【設計不明】**: ツール側で意図的なものか判断がつかないパターン。例えば「意図的にcloseを呼び出し元に委譲している」設計の場合に出ます。今回のコードには出ていません。
- **【保守危険】**: 即座に事故らないが、将来の変更で壊れやすい構造。今回は該当なしでした。

Zaifのソースコードでこのツールを試した時、`【設計不明】` ラベルが大量に出て、どれが本当に危険か切り分けに30分ほど迷った経験があります。ラベルだけで機械的に切り捨てず、【重大】以外もざっと目を通す運用にしています。

## 3段階の使い分け

| 場面 | 使うツール | 使いどころ |
|------|-----------|-----------|
| L0: ビルド時 (まずここ) | gcc / clang の警告 + `-Werror` | `-Wall -Wextra -Wpedantic -Werror -fanalyzer` (GCC) / `-Weverything -Werror` (Clang)。**警告ゼロにならない PR はそもそもレビューに入らない**ルール化 |
| L1: PR チェック | clang-tidy / cppcheck | OSSの主流チェックをCIで強制 |
| L2: PR レビュー | creview（CLI） | プロジェクト固有パターン、日本語ナレッジ、severityモデル (重大/設計不明/保守危険)、CWE/MISRAマッピング、`--preset memory` でリーク系に絞り込み、AI補強 |
| L2': 自己チェック (PR 投げる前) | c-review-ai（ブラウザ版） | 環境構築不要、コピペで試せる軽量版 |
| L3: チーム監査・MISRA 準拠 | CSAF | libclang ASTと依存グラフでリソース確保/解放の対応をプロジェクト全体で追跡し、risk A/B/C自動昇格 |

**運用ルール**: fd/socket/FILEのリークは単体のPRチェックでは全体像が見えないケースがあります（呼び出し元でclose責任を持つ設計など）。creviewでPR単位のリークを潰した上で、月次でCSAFによる全体監査を回し、確保と解放の対応関係がプロジェクト全体で崩れていないかを別軸で確認しています。

## まとめ

- fd/socket/FILE/pipeのクローズ漏れは全てCWE-401ですが、`pipe()` の片側破棄のようにブロッキングを誘発する実害まで含めて考える必要があります。
- 早期returnの分岐が増えるほど、目視レビューでのリーク見逃しは増えます。creviewの`--preset memory`でフロー解析を機械化し、レビューは設計判断に集中させています。
- NULL契約未表明（CWE-476周辺）は、関数内でのif文チェックより`__attribute__((nonnull(N)))`での明示を優先しています。契約を型システム側に寄せた方が、呼び出し側の責任範囲がコンパイル時にはっきりします。
- creviewの検出行番号は確保箇所に寄ることがあるため、報告された行から関数全体のreturnパスを追う一手間は依然として必要です。
- L0（コンパイラ警告ゼロ）を通過したコードだけをレビュー対象にする運用が、creview導入の前提として効いています。


---

### 試すリンク（無料で試す → 業務で監査する）

C言語の静的解析を、いきなり有料ではなく「無料で手を動かす → 日常運用 → 監査レベル」の順で用意しています。

- **まずブラウザで試す → c-review-ai**（自社プロダクト / MIT / 無料）
  貼り付けた1ファイルをその場でレビュー。導入ゼロで挙動を確かめる用。
  → https://cutt.ly/1yuTys1c
- **CLI で日常運用 → creview**（自社プロダクト / 無料）
  手元のコードをコミット前に単発チェック。CI にも組み込める。
  → https://cutt.ly/syuTpB7z
- **リポジトリ全体を監査 → CSAF**（自社プロダクト / ¥4,980 買い切り）
  MISRA 準拠チェックと CWE データフロー解析をリポジトリ横断で走らせ、監査レポート（HTML）を出力。
  レビューの属人化を止めたいチーム向けで、商用の静的解析ツールなら数十万円かかる基本機能を、個人でも入れられる価格にしています。
  → https://cutt.ly/ItKo4MPY

無料の2つで「使える」と感じたら、リポジトリ全体に監査を回すのが CSAF です。バージョンは随時更新しています。
