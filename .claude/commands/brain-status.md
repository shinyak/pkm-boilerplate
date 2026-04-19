---
description: brain の健康状態をダッシュボード形式で表示する
---

リポジトリの現状をコンパクトに可視化する。

## 収集する情報

- `00_Inbox/` のファイル数（未処理数）
- `20_Projects/` 内のアクティブなプロジェクト数（`99_Archives/` を除く）
- `30_Tech_Notes/` のファイル数
- `10_Journal/` の最新エントリ日付
- 最新の weekly-review の日付と経過日数

## 出力フォーマット

```
📊 Brain Status
━━━━━━━━━━━━━━━━━━━━
Inbox:           N items
Active Projects: N
Tech Notes:      N files
Last Journal:    YYYY-MM-DD
Weekly Review:   YYYY-MM-DD (N days ago)
━━━━━━━━━━━━━━━━━━━━
```

## 警告条件

以下の条件に該当する場合は、対応する行の末尾に `⚠️` と短い警告文を添える：

- Inbox が 10 件を超える → 未処理が溜まっている
- Weekly Review が 14 日以上前 → 振り返りが途切れている
- Last Journal が 7 日以上前 → 日次記録が途絶えている
- Active Projects が 5 件を超える → フォーカス分散の可能性

警告がある場合は、表示の後に「次の一歩」候補を 1〜3 点提示する。
