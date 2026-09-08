---
title: "「テストは緑なのに本番だけ500」の正体は、テストと本番で実行主体が非対称だった"
emoji: "🟢"
type: "tech"
topics:
  - "個人開発"
  - "supabase"
  - "nextjs"
  - "テスト"
published: true
---
API エンドポイントへの外部 POST が軒並み 500。なのにローカルの検証スクリプトは全部 pass する。そういう詰まりに出会いました。

環境は Next.js の Route Handler と Supabase（Postgres・RLS 有効）。外部から Bearer トークン付きで POST する構成です。

## 何が起きたか

外部からの upsert が全部 500 で落ちます。同じテーブルに同じ形のデータを入れる検証スクリプトは、手元で何度流しても pass。ログに出るのは RLS の deny だけで、コードのどこが悪いのかは読み取れませんでした。

## 原因

Route Handler が cookie ベースの Supabase クライアントを使っていました。cookie を持たない外部 POST では `auth.uid()` が null になり、RLS で全 upsert が deny されます。

一方、検証スクリプトは `service_role` で走っていました。service_role は RLS を素通りするので、同じ SQL でも pass します。テストと本番で「DB にアクセスする主体（権限）」が違う。この非対称のせいで、テストは赤くなりようがなかった、という話です。

## 根治

`service_role` で RLS をバイパスする admin クライアントを `lib/supabase/admin.ts` として新設し、Route Handler をそちらに差し替えました。Bearer で `user_id` は確定しているので、RLS に頼らず自前でスコープを絞れば安全です。ついでに zod スキーマへ `.strict()` を足して、フィールド名の typo が静かに落ちる穴も塞いでいます（commit `54bf58b`）。

## 一般化

「テスト緑・本番落ち」は、テストと本番で実行主体が非対称なときの典型です。RLS・IAM・認証を検証するなら、本番と同じ主体、同じ権限でリクエストを通すこと。権限を素通りするテストは、権限のバグを一生見つけられません。

自分が実際に詰まって直した事例を、症状・原因・根治の形で逆引きできるようにまとめています。全 24 本です。

https://brain-market.com/u/sukoshi_nantoka/a/bykDO1UjMgoTZsNWa0JXY

---

**著者**：ぽん（@pon_freelance）
C言語実務23年、組み込み／制御系。
副業で技術記事販売と自作ツール販売をやっている。
