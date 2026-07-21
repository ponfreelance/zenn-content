---
title: "creview で double free と dangling pointer を14件検出した記録"
emoji: "🛡️"
type: "tech"
topics: ["c", "embedded", "codereview"]
published: true
---

C言語23年、自分はフリーランスで組み込みのコードレビューに入ることが多いです。今回は「同じポインタを2回freeする」「free後にNULLを入れ忘れる」という、若手だけでなく中堅も普通にやらかすパターンを、creview に静的解析させて14件の検出を出した実例を書きます。malloc/freeの寿命管理は経験年数と関係なく事故る領域で、自分自身も過去にレビューで見逃して本番障害に繋げたことがあります。今回のコードはそのときの再現に近いものを意図的に書きました。

## L0: まずコンパイラ警告を全部潰す

creview の話に入る前に、`-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）または `-Wall -Wextra -Werror -Weverything`（Clang）を有効にしてビルドが通るか確認してください。NULL引数をnonnull関数に渡す、未初期化auto変数を読む、戻り値を破棄する、自明なNULL deref、use-after-free、境界外配列アクセス（静的決定可なもの）、switch case抜け、これらは現代のコンパイラが標準で拾います。

実際、今回の検証コードを `gcc -Wall -Wextra -fanalyzer` に通すと、`-fanalyzer` は `release()` の二重freeと `load_or_fail()` の分岐によるuse-after-freeの疑いをかなりの精度で拾ってきます。ここで拾えるものを creview に二度手間で読ませても意味がないので、まずビルドを警告ゼロにするのが先です。

これらが警告ゼロにならないPRは、creview を走らせる前にそもそもレビュー対象にしないルールにしてしまうのが第一手です。自分のプロジェクトでは、CIで `-fanalyzer` の警告が1件でも残っているPRはレビュー担当にアサインしない設定にしています。

## SCQA

- **状況**: malloc/freeのペアリングは静的解析ツールの得意領域ですが、分岐をまたいだdouble freeやdangling pointerの見逃しは今でも普通に起きます。
- **問題**: 「free直後にNULLを入れる」はコーディング規約に書いてあっても、レビューで毎回全行チェックするのは現実的ではありません。`load_or_fail()` のようにreturn文が絡む分岐だと、レビュー中に頭の中でパスを追いきれず素通りすることがあります。
- **問い**: 分岐をまたぐfree済みポインタの再利用を、人力レビューに頼らず機械的に洗い出すにはどうするか。
- **答え**: creview に `--preset memory` でメモリ安全性に特化した解析をかけ、14件の検出を得た上で、重大度別に対応の優先順位を分けました。

## 検証用コード

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* 同じポインタを 2 回 free（double free） */
void release(char *p) {
    free(p);
    free(p);                             /* 検出すべき: 二重 free */
}

/* エラーパスで double free */
char *load_or_fail(int ok) {
    char *buf = malloc(128);
    if (!ok) {
        free(buf);
        return NULL;
    }
    free(buf);                           /* 検出すべき: 上の return NULL に行かなかった分岐で再 free */
    return buf;                          /* use-after-free + double-free 候補 */
}

/* free 後 NULL 代入を忘れた dangling pointer */
void cleanup(char **pp) {
    free(*pp);                           /* 検出すべき: *pp = NULL を入れていない */
}

/* 2 つの所有者が同じバッファを free */
void share_and_free(char *a, char *b) {
    if (a == b) {
        free(a);
        free(b);                         /* 検出すべき: 同じバッファに対する 2 回 free */
    }
}
```

`load_or_fail()` は自分が一番ハマったパターンです。`ok == 0` の分岐だけ見ると正しくfree & returnしているので、レビューで上から読んでいくと「ちゃんと解放してるな」で通過しがちです。しかし `ok != 0` のパスでは `free(buf)` した直後に同じ `buf` をそのままreturnしており、呼び出し側は解放済みポインタを受け取ります。この手のバグはレビューでコードを縦に読むだけでは気づきにくく、パスごとに分けて追わないと見えません。

## CWE/MISRA マッピング

| 検出パターン | CWE | 事故の形 |
|------------|-----|---------|
| release() の p を NULL 検査せず使用 | CWE-476 | nonnull契約未表明のまま参照、呼び出し側がNULLを渡すとクラッシュ |
| release() 内の二重free | CWE-415 | 同一ポインタへの2回目のfreeでヒープメタデータ破壊、glibcなら abort または任意コード実行の足がかり |
| free直後にNULL代入なし（各所） | CWE-416（UAFの温床） | 解放済みアドレスが残り続け、後続処理で誤って参照されるとuse-after-free |
| load_or_fail() のbufにNULLチェックなし | CWE-690 | malloc失敗時にNULL deref、組み込みではハードフォールト |
| load_or_fail() のマジックナンバー128 | CWE-547 | バッファサイズの意図が読み取れず、後日のサイズ変更時に他箇所との整合が崩れる |
| load_or_fail() 内の二重free | CWE-415 | 分岐によっては`free(buf)`後に同じ`buf`をreturnし、呼び出し側でのuse-after-freeも誘発（CWE-416） |
| cleanup() の pp を NULL検査せず使用 | CWE-476 | 二重ポインタの契約未表明、NULL渡しでクラッシュ |
| share_and_free() の a を NULL検査せず使用 | CWE-476 | 同上、契約が関数シグネチャから読めない |
| share_and_free() 内、a==bで2回free | CWE-415 | 別名ポインタの同一性チェック後もfreeを2回呼ぶ設計ミス |

## creview の使い方（CLI 実行例）

```
$ creview --preset memory --format markdown src/free_patterns.c
```

```
L6 【重大】: ATTR-NONNULL-001: release() の p を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L7 【保守危険】: free(p)直後にNULL代入なし。dangling pointer残存
L8 【重大】: pの二重free。7行目で既にfree済み
L8 【保守危険】: free(p)直後にNULL代入なし。dangling pointer残存
L12 【保守危険】: 関数load_or_fail()の宣言・定義がファイル内に見つからない。ヘッダinclude漏れまたはリンクエラーの可能性
L13 【重大】: bufにNULLチェックなし。malloc/calloc/realloc失敗時クラッシュ
L13 【重大】: マジックナンバー128 (バッファサイズ) — allowlist 外。#define / enum / .creviewrc.json で命名 or 許可
L15 【保守危険】: free(buf)直後にNULL代入なし。dangling pointer残存
L18 【重大】: bufの二重free。15行目で既にfree済み
L18 【保守危険】: free(buf)直後にNULL代入なし。dangling pointer残存
L23 【重大】: ATTR-NONNULL-001: cleanup() の pp を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L28 【重大】: ATTR-NONNULL-001: share_and_free() の a を NULL 検査せず使用。__attribute__((nonnull(N))) で契約を明示するか、関数先頭で NULL 検査して return
L30 【保守危険】: free(a)直後にNULL代入なし。dangling pointer残存
L31 【保守危険】: free(b)直後にNULL代入なし。dangling pointer残存
```

3段階ラベルの読み方は以下の通りです。

- **【重大】**: double free、NULL deref、境界外アクセスなど実行時にクラッシュ・メモリ破壊に直結する項目。マージ前に必ず潰す。
- **【設計不明】**: ツール側がプロジェクトのドメイン知識を持たないため断定できない項目（今回は出ていませんが、例えば「この関数はコールバックとして登録されるのでシグネチャは変更不可」など人間の判断が必要な指摘に使うラベル）。
- **【保守危険】**: 今すぐ落ちるわけではないが将来の変更で事故る土壌になっている項目。dangling pointerの放置やマジックナンバーはここに入る。

L12の「load_or_fail()の宣言・定義が見つからない」は誤検知気味の指摘で、実際には同一ファイル内に定義があります。creview のシンボル解決がこのケースで甘かったので、実行結果をそのまま信じずに該当行を目視で確認しました。ツールの指摘を鵜呑みにしない姿勢も必要です。

## 3段階の使い分け

| 場面 | 使うツール | 使いどころ |
|------|-----------|-----------|
| L0: ビルド時（まずここ） | gcc / clang の警告 + `-Werror` | `-Wall -Wextra -Wpedantic -Werror -fanalyzer`（GCC）/ `-Weverything -Werror`（Clang）。警告ゼロにならないPRはそもそもレビューに入らないルール化 |
| L1: PRチェック | clang-tidy / cppcheck | OSSの主流チェックをCIで強制 |
| L2: PRレビュー | creview（CLI、自社プロダクト） | プロジェクト固有パターン、日本語ナレッジ、severityモデル（重大/設計不明/保守危険）、CWE/MISRAマッピング、`--preset pr` でdiffのみ、AI補強 |
| L2': 自己チェック（PR投げる前） | c-review-ai（ブラウザ版、自社プロダクト） | 環境構築不要、コピペで試せる軽量版 |
| L3: チーム監査・MISRA準拠 | CSAF（自社プロダクト） | libclang AST + 依存グラフでrisk A/B/C自動昇格 |

運用ルールとして「L0の警告がゼロにならないPRはレビューに入らない」をCIで強制しています。L0を素通りしたPRを creview で叩いても、コンパイラが既に警告として出せる内容まで人間が読むことになって時間の無駄です。

## まとめ

- double free とdangling pointerは、分岐をまたぐパス（`load_or_fail()` のようなケース）で特に見逃しやすく、レビューで縦読みするだけでは検出漏れが出ます。
- creview の `--preset memory` はfree絡みの14件を一括で洗い出しましたが、L12のシンボル解決のようにツール側の誤検知も混じるため、【重大】ラベルでも目視確認は省略しない方針にしています。
- NULLチェックの第一選択は関数本体でのif文ではなく `__attribute__((nonnull))` による契約表明で、release()やcleanup()、share_and_free()はいずれもこの属性が抜けていました。
- マジックナンバー128のような数値も、CWE-547として同じ検出ロジックの対象にしておくと、後からサイズ変更するときの見落としを防げます。
- L0（コンパイラ警告）を通していないコードにcreviewをかけると、コンパイラで拾える指摘まで重複して読む羽目になるので、CIでの警告ゼロ強制が先です。


---

### 試すリンク

- **ブラウザで気軽に試す → c-review-ai**（自社プロダクト / MIT / 無料）
  → https://github.com/ponfreelance/c-review-ai
- **CLI で日常運用 → creview**（自社プロダクト / 無料）
  → https://github.com/ponfreelance/creview
- **業務・監査レベル → CSAF**（自社プロダクト / ¥4,980）
  → https://cutt.ly/ItKo4MPY

2026年4月時点の情報です。creview / CSAF のバージョンは随時更新しています。
