---
title: "C言語静的解析ツール creview を使って危険コードを検出"
emoji: "🛡️"
type: "tech"
topics: ["c", "embedded", "codereview"]
published: true
---

C言語23年、自分はフリーランスで組み込みのコードレビューに入ることが多いです。若手の PR に入って一番「またこれか」となるのが、二重 free や use-after-free、dangling pointer などのメモリ関連のバグです。どれも一度教えれば直るのに、週をまたいだ別の PR でまた出てくる。人間のレビューワで全部拾うのは現実的じゃないと割り切って、自社プロダクトの creview に押し付けることにしました。

## この記事の前提（SCQA）

- **状況**: C 言語のコードレビューで、若手への指摘の7〜8割が同じパターンに収束します。二重 free や use-after-free、dangling pointer など。
- **問題**: 毎週の PR で同じ指摘を書くのは、自分にとってもレビューされる側にとっても地獄です。しかも「疲れた日」に見逃すと、本番に流れ込みます。
- **問い**: 人間が覚えておくべきことと、機械に任せていいことを分けるには、どの粒度で自動化するのが現実적か？
- **答え**: creview で「再発する定型的な危険」を全部拾い、人間のレビューは「設計判断」と「ドメイン知識」に集中するように分業しました。

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

## CWE/MISRA マッピング

| 検出パターン | 正しい CWE | 実害 |
|------------|----------|-----|
| 二重 free (double free) | CWE-415 | クラッシュ・メモリ破壊 |
| use-after-free (free 後参照) | CWE-416 | クラッシュ・メモリ破壊 |
| dangling pointer（free 後 NULL 代入無し） | CWE-416 | クラッシュ・メモリ破壊 |

## creview の使い方（CLI 実行例）

```
L7 【保守危険】: free(p)直後にNULL代入なし。dangling pointer残存
L8 【重大】: pの二重free。7行目で既にfree済み
L8 【保守危険】: free(p)直後にNULL代入なし。dangling pointer残存
L12 【保守危険】: 関数load_or_fail()の宣言・定義がファイル内に見つからない。ヘッダinclude漏れまたはリンクエラーの可能性
L13 【重大】: bufにNULLチェックなし。malloc/calloc/realloc失敗時クラッシュ
L13 【保守危険】: マジックナンバー128。定数定義なしで意味不明・変更困難
L15 【保守危険】: free(buf)直後にNULL代入なし。dangling pointer残存
L18 【重大】: bufの二重free。15行目で既にfree済み
L18 【保守危険】: free(buf)直後にNULL代入なし。dangling pointer残存
L30 【保守危険】: free(a)直後にNULL代入なし。dangling pointer残存
L31 【保守危険】: free(b)直後にNULL代入なし。dangling pointer残存
```

【重大】はクラッシュやメモリ破壊につながる深刻なバグです。【保守危険】は、メンテナンス時に問題を引き起こす可能性のある箇所です。

## 3段階の使い分け

| 場面 | 使うツール | 使いどころ |
|------|-----------|-----------|
| 若手が PR を投げる前の自己チェック | c-review-ai（ブラウザ版） | 環境構築不要、コピペで試せる |
| 自分が PR レビュー前に diff チェック | creview（CLI） | `--preset pr` で差分行のみ |
| CI に組み込みたい | creview `--format sarif` | GitHub Code Scanning 連携 |
| チーム監査・MISRA 準拠 | CSAF | libclang AST + 依存グラフで risk A/B/C 自動昇格 |

## まとめ

- 自社プロダクトの creview を使うことで、定型的な危険を自動で検出できる
- 3段階の使い分け（c-review-ai / creview / CSAF）で、効率的なコードレビューが可能
- CWE 番号を正しくマッピングすることで、バグの深刻度を客観的に評価できる


---

### 試すリンク

- **ブラウザで気軽に試す → c-review-ai**（自社プロダクト / MIT / 無料）
  → https://github.com/ponfreelance/c-review-ai
- **CLI で日常運用 → creview**（自社プロダクト / 無料）
  → https://github.com/ponfreelance/creview
- **業務・監査レベル → CSAF**（自社プロダクト / ¥4,980）
  → https://cutt.ly/ItKo4MPY

2026年4月時点の情報です。creview / CSAF のバージョンは随時更新しています。
