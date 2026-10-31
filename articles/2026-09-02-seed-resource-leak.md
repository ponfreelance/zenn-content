---
title: "fdリークの見落としは「早期return」に集中する、7件の内訳から言えること"
emoji: "🛡️"
type: "tech"
topics: ["c", "embedded", "codereview"]
published: true
---

C言語23年、自分はフリーランスで組み込みのコードレビューに入ることが多いです。fd・socket・FILEのclose忘れは静的解析の定番テーマですが、実際のPRで見逃されるのは「close自体を書き忘れた行」ではなく「正常系ではcloseしているのに、途中のreturnでcloseし損ねている行」です。今回検証したコードは4関数・37行程度の小さなものですが、creviewは7件を検出し、そのうち3件がリソースリーク、4件がNULL契約未表明でした。件数の内訳を見ると、レビューワが読み飛ばしやすい箇所がどこかが逆算できます。

## SCQA

- **状況**: fd/socket/FILEのclose忘れは、CWE-401としてどの静的解析ツールの資料にも載っている定番の指摘項目です。
- **問題**: 定番だからといって手動レビューで拾えるとは限りません。特に「異常系のreturnパス」は正常系のロジックを追うのに気を取られて読み飛ばされやすく、今回の`read_first_byte()`のように、正常系ではcloseしているのに異常系のreturnではcloseし忘れる、というパターンが典型です。
- **問い**: 37行程度の小さな関数群でも7件の指摘が出るということは、レビューワは何を根拠に「ここは大丈夫」と判断してしまっているのか。
- **答え**: creviewでopen/socket/fopen/pipeそれぞれの戻り値に対して、全returnパスでのclose有無を機械的に突き合わせ、人間は「そのリソースを本当にここで手放してよいか」という設計判断だけに集中する分業にしました。

## L0: まずコンパイラ警告を全部潰す

creviewの話に入る前に、`-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）または`-Wall -Wextra -Werror -Weverything`（Clang）を有効にしてビルドが通るか確認してください。NULL引数をnonnull関数に渡す・未初期化auto変数を読む・戻り値を破棄する・自明なNULL deref・use-after-free・境界外配列アクセス（静的決定可なもの）・switch case抜け、これらは現代のコンパイラが標準で拾います。

今回の検証コードでいえば、`-fanalyzer`を付けたGCCは`file_size()`のfclose忘れパスの一部を検出できる可能性があります。ただしopen/socket/pipeのfdリークは処理系依存で拾えたり拾えなかったりするため、これらが警告ゼロになった前提の上で、creviewのリソースリーク検出を次の砦として使うのが現実的な順番です。警告ゼロにならないPRは、creviewを走らせる前にそもそもレビュー対象にしないルールにしてしまうのが第一手です。

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

`read_first_byte()`は一見「ちゃんとcloseしている関数」に見えます。18行目で`close(fd)`を書いているからです。しかし12〜14行目の`read()`失敗パスにはcloseが無く、正常系だけ見て「このコードは大丈夫」と判断すると通り抜けます。手動レビューでこの手のパターンを見落としやすいのは、まさに「一度closeを書いた実績があるコード」を後半だけ読んで安心してしまう時です。

## CWE/MISRA マッピング

| 検出パターン | 該当行 | CWE | 事故の形 |
|---|---|---|---|
| open()後、read失敗時にclose無しでreturn | L8-14 | CWE-401 | 異常系を通るたびにfdを1つ消費、長時間稼働プロセスでfd枯渇 |
| socket()後、closeせずreturn | L20-21 | CWE-401 | 接続確立の成否に関わらずfdリーク、サーバプロセスでは早期に上限到達 |
| fopen()後、fclose無しでreturn | L27-28 | CWE-401 | FILE構造体とfdの二重リーク、ファイルサイズ取得のような軽い処理でも蓄積する |
| pipe()の書き込み側を呼び出し元に返さず破棄 | L35-37 | CWE-401 | 読み出し側だけ返す設計自体が両端管理を難しくし、書き込み側が永久にリークする |
| path/host/r のNULL未検査使用 | L8, L20, L27, L35 | CWE-476 | 外部境界からNULLが渡ると即座にクラッシュ、契約が関数シグネチャに現れていない |

`read_first_byte()`のCWE-401は、`open()`直後の単純な閉じ忘れではなく「returnパスが複数あるうちの1本だけ閉じ忘れている」点が特徴です。この形はコードレビューでのdiff読みでは、変更差分に異常系のreturn文が含まれていないと見えにくく、既存コードの手直し時に見落としが起きやすい典型例です。

## creview の使い方（CLI 実行例）

```bash
creview review resource_leak_sample.c --preset security --format markdown
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

3段階ラベルの読み方は以下の通りです。

- **【重大】**: 実害に直結する、または契約未表明でクラッシュや資源枯渇に至る指摘。今回の7件は全てこのラベルで、resource leak系4件とnonnull契約未表明系4件が該当します（`connect_to()`はsocketリークとhostのnonnull両方でL20に集約）。
- **【設計不明】**: 今回の検出結果には出ていませんが、意図的な設計かバグか判断がつかない箇所（例えばエラーコードを握りつぶすswitch文など）に付くラベルです。
- **【保守危険】**: 今すぐ落ちはしないが、将来の変更で壊れやすい書き方（マジックナンバー、暗黙の型変換など）に付くラベルです。

今回の7件が全て【重大】に振られている点は、材料としてそのまま受け取っておくべきです。「重大」が並ぶこと自体が、この4関数がレビュー観点では要注意コードだという裏付けになります。

pipe周りのL37の指摘は、`pipe(p)`という1行に対する指摘でありながら、実際に見るべきは`*r = p[0]`だけ代入して`p[1]`を握り潰している34〜37行目の設計です。creviewの指摘行と、人間が読むべき行がずれることがあるため、行番号だけでなく前後の文脈まで読む必要があります。

## 3段階の使い分け

| 場面 | 使うツール | 使いどころ |
|------|-----------|-----------|
| L0: ビルド時（まずここ） | gcc / clangの警告 + `-Werror` | `-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）/ `-Weverything -Werror`（Clang）。警告ゼロにならないPRはそもそもレビューに入らないルール化 |
| L1: PRチェック | clang-tidy / cppcheck | OSSの主流チェックをCIで強制 |
| L2: PRレビュー | creview（CLI、自社プロダクト） | プロジェクト固有パターン、日本語ナレッジ、severityモデル（重大/設計不明/保守危険）、CWE/MISRAマッピング、`--preset pr`でdiffのみ、AI補強 |
| L2': 自己チェック（PR投げる前） | c-review-ai（ブラウザ版、自社プロダクト） | 環境構築不要、コピペで試せる軽量版 |
| L3: チーム監査・MISRA準拠 | CSAF（自社プロダクト） | libclang AST + 依存グラフでrisk A/B/C自動昇格 |

`make_channel()`のようなpipe両端管理の設計ミスは、L2のcreviewで「closeなし」として拾えても、「そもそも呼び出し元にp[1]を返す設計に直すべきか」まではツールは判断してくれません。ここは【重大】ラベルを起点に人間が設計を見直す領域で、CSAFで依存グラフを見ながら「このpipeを使っている全呼び出し元がp[1]を握り潰しているか」を横断チェックするのがL3の役目です。

## まとめ

- 7件中3件はfd/socket/FILEのclose忘れ（CWE-401）、4件はNULL契約未表明（CWE-476）で、件数比は「リソース管理」と「契約表明」がほぼ同格の指摘対象であることを示しています。
- `read_first_byte()`は正常系でcloseを書いているにもかかわらず異常系のreturnで漏れており、手動レビューは「closeが書かれている実績」を見て安心し、returnパスの本数を数え切れずに読み飛ばしやすいパターンです。
- `make_channel()`のpipe両端管理は、指摘行（L37）と実際に直すべき行（p[1]を捨てている箇所）がずれるため、行番号だけを見て終わらせず前後の設計まで読む必要があります。
- 4関数全ての第一引数・第一出力引数がNULL未検査であり、`__attribute__((nonnull))`を関数シグネチャに付けるだけで、呼び出し側の契約違反をコンパイル時に近い段階で洗い出せます。
- 7件全てが【重大】ラベルという結果は、この検証コードがレビュー観点で軽視できない密度の問題を抱えていたことの裏付けであり、ラベルの偏り自体もレビュー優先度判断の材料になります。


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
