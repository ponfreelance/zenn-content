---
title: "Untitled"
emoji: "🔧"
type: "tech"
topics:
  - "Python"
  - "FastAPI"
  - "個人開発"
published: true
---
🚀 AutoTrader v1.15.2 リリース (2026-05-23)

【一行サマリ】
rule 戦略のシグナル反転 SELL が固定 TP/SL に届かないまま手数料負けで連発する事象を根治。手数料負けガード (default ON) を追加。

【根本対策 — _should_suppress_reversal_sell】
• rule 戦略の MA デッドクロス / RSI 反転 / BB 反転 / MACD クロスによる SELL を、固定 TP/SL を設定している BOT では HOLD に置換
• 設定: bot.config.tpSlOverridesSignal (default true)
• TP=1% / SL=3% のような BOT で「TP に届かない反転 SELL → 手数料負け」を構造的に排除
• 旧挙動に戻すには tpSlOverridesSignal=false

【バックストップ — _check_fee_loss_guard】
• SELL 発注直前に「含み益 < taker手数料 × 2 × margin」なら ORDER_SKIPPED でスキップ
• Signal.exit_reason in {TP, SL, TRAIL, RANGE_EXIT, ANOMALY_EXIT} は素通り (強制出口は守る)
• 設定: bot.config.feeGuardEnabled (default true) / feeGuardMargin (default 1.0)
• アラート PUSH せず BOT ログ記録のみ (通常運用ログ扱い)
• 旧挙動に戻すには feeGuardEnabled=false

【副次対応 — exit_reason 一元化】
• engine.py:605-622 固定 TP/SL に exit_reason="TP"|"SL" 付与
• strategies/scalping.py:111-128 / 189-207 (OHLCV+ticker) に exit_reason 付与
• strategies/market_making.py:113 (SL) に exit_reason="SL"
• strategies/dca.py:160 (SL) に exit_reason="SL"
• safety/exit_strategy.py:273 は既設定
• → 「exit_reason is None ⇔ 戦略シグナル反転 SELL」が engine 全体で確定

【admin 用 endpoint 追加】
• POST /api/admin/reconcile-bot-position
• 2026-05-01 incident 修復用 (BOT合計 > 取引所 で balance_shortage ANOMALY 連発したケース)
• dry_run=True で preview / False で ADJUSTMENT Trade を 1 件 INSERT
• Trade DELETE せず ADJUSTMENT 追加投入 → 監査トレース保全
• min_order_amount/10 未満は noop

【検証】
• backend pytest: 692 passed in 18.88s (baseline 663 + 29 新規 + 7 reconcile)
• 安全装置との干渉確認: ドローダウン / 日次週次損失 / 連敗 / 緊急停止 / 日次注文数 / dust SELL / 動的出口 すべて干渉なし
• 水平展開: rule / scalping / dca / market_making / tsumitate / grid / ai の各戦略で exit_reason の付与状況を確認

【影響範囲】
• DB スキーマ変更: なし (bot.config JSONB に新キー追加のみ)
• 後方互換: 完全
• データ移行: 不要
• APK 再配布: 不要 (backend のみ)
• フロント UI (設定タブ ON/OFF スイッチ): 次バージョン以降で追加予定

【既知の制約】
• 指値 SELL の実約定価格滑りまではガードしない (signal.price で判定)
• maker / taker 混在は taker 固定で計算 (将来 feeGuardFeeType で切替可能)
• slippage 想定は別途 feeGuardSlippagePct 拡張で検討
• オーファン在庫 (改修前から existing な open_quantity) は対象外 — fee guard は対象、反転抑止は対象外 (元々 None だが TP/SL は届く)

【配布】
• ZIP 4 種: dist/autotrader-{light,standard,advanced,elite}-v1.15.2.zip
• Google Drive 4 file_id 上書き
• update.sh は backend/VERSION 経由で自動取得 (MEMBER_TOKEN 経路)
• Render / Oracle 反映済 (要確認)

【次】
• Phase E Step 2: bot.trades キャッシュカラム alembic DROP (本番観察 diff=0 で着手)
• fee guard のフロント UI 追加 (BOT 詳細 → 設定タブ)
• maker/taker 切替 + slippage 拡張は要望ベース
