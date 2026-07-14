---
title: "strcpy・sprintf・sizeof(ptr) 誤用を creview で検出した記録"
emoji: "🛡️"
type: "tech"
topics: ["c", "embedded", "codereview"]
published: true
---

C言語23年、自分はフリーランスで組み込みのコードレビューに入ることが多いです。

先週入った案件で、`sizeof(p)` を「バッファ全体のサイズ」だと思い込んで `memset(p, 0, sizeof(p))` と書かれたコードに遭遇しました。ポインタ引数の sizeof は 8（64bit環境）を返すだけで、確保した領域のクリアにはなっていません。本人に指摘したら「え、sizeof って便利なマクロじゃないんですか」と真顔で返ってきて、30分説明に費やしました。今回はこの手のバグを含む4関数のサンプルコードを creview に食わせて、検出結果を実際に見ていきます。

## この記事の前提（SCQA）

- **状況**: strcpy / sprintf / sizeof(ptr) / 配列添字の未検証、この4パターンは C 言語のバグ報告の中でも息が長く、20年経っても新人コードに出続けます。
- **問題**: 「危険なのは知ってるけど、レビューで毎回目視するのは疲れる」というのが現場のリアルです。しかも `sizeof(p)` はコンパイラが警告を出さないケースがあり、目視でも見逃しやすいです。
- **問い**: コンパイラの警告と静的解析ツールで、どこまでを機械的に潰せて、どこから人間の判断が必要になるのか？
- **答え**: L0（コンパイラ警告）で拾えない「意味論的な誤用」を creview に任せ、CWE番号付きで理由まで残すことで、レビューのやり取りを減らしました。

## L0: まずコンパイラ警告を全部潰す

creview の話に入る前に、`-Wall -Wextra -Wpedantic -Werror -fanalyzer`
(GCC) または `-Wall -Wextra -Werror -Weverything` (Clang) を有効に
してビルドが通るか確認してください。NULL 引数を nonnull 関数に渡す /
未初期化 auto 変数を読む / 戻り値を破棄する / 自明な NULL deref /
use-after-free / 境界外配列アクセス（静的決定可なもの）/ switch case
抜け、これらは現代のコンパイラが標準で拾います。

これらが警告ゼロにならない PR は、creview を走らせる前にそもそも
レビュー対象にしないルールにしてしまうのが第一手です。

実際、今回のサンプルコードを `gcc -Wall -Wextra -fanalyzer` に通すと、`strcpy` については `_FORTIFY_SOURCE` 環境下で警告が出ることがありますが、`sizeof(p)` の誤用や `scanf` からの添字未検証は、コンパイラの静的解析だけでは拾いきれません。ここが creview の出番です。

## 検証用コード

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

## CWE/MISRA マッピング

| 検出箇所 | 検出パターン | CWE | 事故の形 |
|---------|-------------|-----|---------|
| L6 | src の NULL 契約未表明 | CWE-476 周辺 | 呼び出し側が NULL を渡すとクラッシュ。nonnull 属性で契約化すべき |
| L7 | マジックナンバー16 | CWE-547 | バッファサイズが散在し、変更時の追従漏れで別箇所とサイズ不整合 |
| L8 | strcpy 使用 | CWE-121 | スタックバッファオーバーフロー。src が16文字以上でリターンアドレス破壊の可能性 |
| L13 | dir の NULL 契約未表明 | CWE-476 周辺 | NULL 渡しでの sprintf クラッシュ |
| L14 | マジックナンバー32 | CWE-547 | path 用バッファサイズのハードコードで保守時の見落とし |
| L15 | sprintf 使用 | CWE-121 | dir+file の合計長が32byte超でスタック破壊 |
| L19 | p の NULL 契約未表明 | CWE-476 周辺 | NULL ポインタを memset に渡してクラッシュ |
| L20 | p の NULL 未検証で memset | CWE-476 | NULL 渡しの実行時クラッシュ（契約違反の実害側） |
| L20 | sizeof(p) をポインタサイズと誤用 | CWE-467 | 確保領域の一部しかゼロクリアされず、機密情報の残存や不定値混入 |
| L24 | arr の NULL 契約未表明 | CWE-476 周辺 | NULL 配列参照でクラッシュ |
| L27 | scanf 入力を境界チェックなしで添字使用 | CWE-129 | 範囲外読み取り（CWE-125相当の実害）、負値でメモリ破壊の入口にもなる |

L20 の sizeof 誤用について補足すると、これは `char buf[100]` のようなローカル配列宣言の sizeof ではありません。`wipe_buffer(char *p)` の `p` は明示的なポインタ宣言なので、sizeof は常にポインタサイズ（4/8byte）を返します。配列宣言そのものの sizeof が pointer サイズになるわけではない点は区別が必要です。

## creview の使い方（CLI 実行例）

```bash
creview scan buffer_bugs.c --preset security --format markdown
```

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

3段階ラベルの意味は以下の通りです。

- **【重大】**: 実行時クラッシュやメモリ破壊に直結する。マージ前に必ず修正。今回の11件は全部これに該当します。
- **【設計不明】**: バグではないが意図が読み取れない実装。設計者に確認が必要（今回は該当なし）。
- **【保守危険】**: 今は動くが将来の変更で壊れやすい構造（マジックナンバーの散在などが該当しやすいが、今回は重大側に分類されています）。

今回全部が【重大】に振られているのは、`--preset security` が「バッファ破壊・NULL契約・境界チェック」を最優先カテゴリにしているためです。`--preset pr` で diff だけに絞ると、レビュー対象の行数によっては件数がもっと絞られます。

## 4段階の使い分け

| 場面 | 使うツール | 使いどころ |
|------|-----------|-----------|
| L0: ビルド時 (まずここ) | gcc / clang の警告 + `-Werror` | `-Wall -Wextra -Wpedantic -Werror -fanalyzer` (GCC) / `-Weverything -Werror` (Clang)。**警告ゼロにならない PR はそもそもレビューに入らない**ルール化 |
| L1: PR チェック | clang-tidy / cppcheck | OSS の主流チェックを CI で強制 |
| L2: PR レビュー | creview（CLI、自社プロダクト） | プロジェクト固有パターン、日本語ナレッジ、severity モデル (重大/設計不明/保守危険)、CWE/MISRA マッピング、`--preset pr` で diff のみ、AI 補強 |
| L2': 自己チェック (PR 投げる前) | c-review-ai（ブラウザ版、自社プロダクト） | 環境構築不要、コピペで試せる軽量版 |
| L3: チーム監査・MISRA 準拠 | CSAF（自社プロダクト） | libclang AST + 依存グラフで risk A/B/C 自動昇格 |

**運用ルール**: 「L0 の警告がゼロにならない PR はレビューに入らない」を CI で強制します。L0 を素通りした PR を creview で叩いても、消せた警告まで人間が読むことになって意味がありません。

自分の現場では、PR を投げる前に個人で c-review-ai に貼って軽く流し、CI 上で creview を `--preset pr` で走らせ、月次のリリース判定でだけ CSAF の依存グラフ監査を回す、という3段構えにしています。CSAF まで毎回回すとビルド時間が伸びすぎるので、そこは頻度を分けるのがコツです。

## まとめ

- strcpy / sprintf の境界未検証は CWE-121 に直結し、20年前と変わらず現役のバグパターンです。snprintf への置き換えは機械的にできるので、レビューで毎回議論するより allowlist ルールに落とし込んだ方が早いです。
- `sizeof(p)` のポインタサイズ誤用（CWE-467）はコンパイラ警告に出にくく、目視でも「あるある」すぎて逆に見落とされます。配列長は別引数で渡す設計に直すのが根本対応です。
- NULL 契約は関数本体でチェックするより `__attribute__((nonnull))` で先に明示する方が、呼び出し側の責任範囲がはっきりします。組み込みでは入力境界（外部API・ISR・通信受信）だけランタイムチェックに絞るのが現実的です。
- マジックナンバー(16, 32)はバッファサイズの散在を招き、後日別の関数で同じサイズ違いのバグが再発します。`.creviewrc.json` の allowlist で機械的に弾く運用が効きます。
- scanf からの入力をそのまま配列添字に使う CWE-129 は、境界チェック漏れの中でも実害（範囲外読み取り、書き込みなら CWE-787）に直結しやすいので、L2 の creview チェックで確実に拾う設計にしています。


---

### 試すリンク

- **ブラウザで気軽に試す → c-review-ai**（自社プロダクト / MIT / 無料）
  → https://github.com/ponfreelance/c-review-ai
- **CLI で日常運用 → creview**（自社プロダクト / 無料）
  → https://github.com/ponfreelance/creview
- **業務・監査レベル → CSAF**（自社プロダクト / ¥4,980）
  → https://cutt.ly/ItKo4MPY

2026年4月時点の情報です。creview / CSAF のバージョンは随時更新しています。
