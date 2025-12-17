## 概要
`/test/todo/select/confirmation`ページにおいて、半角文字で記述された長いゴール名がカードの親要素からはみ出して表示される不具合を修正する。

## 問題の詳細

| 項目 | 内容 |
|------|------|
| **発生箇所** | `/test/todo/select/confirmation` |
| **問題** | 長い半角テキストのゴール名がカードの境界を超えてはみ出す |
| **原因** | `GoalConfirmation.tsx`のゴール名表示部分にテキスト折り返し・オーバーフロー制御が未実装 |

## 現状と理想

### 現状
- `/test/todo/select/confirmation`にて半角で記述された長いゴール名が親要素を飛び出して表示される

### 理想
- `/test/todo/select/[id]`のようにゴール名が親のカード内に収まる

## 原因分析

### 正常に動作している箇所（参考）
- `GoalCard.tsx` (74-85行目)
- `line-clamp-2`クラスを使用してテキストを2行で切り詰め

### 問題のある箇所
- `GoalConfirmation.tsx` (132-144行目)
- `<span>`タグにオーバーフロー制御なし
- 半角文字（英数字等）は自動改行されないため、親要素をはみ出す

## 修正方針

`GoalConfirmation.tsx`のゴール名表示部分に以下のCSSクラスを追加：

- `break-all` または `break-words`: 長い単語を強制的に折り返し
- `overflow-hidden`: はみ出し部分を非表示
- `line-clamp-2`（オプション）: 2行で切り詰め、省略記号を表示

## 影響範囲

| ファイル | 変更内容 |
|----------|----------|
| `src/app/(private)/test/todo/select/_components/GoalConfirmation.tsx` | `<span>`タグにCSS追加 |

## テスト観点

- [ ] 長い半角ゴール名がカード内に収まること
- [ ] 長い全角ゴール名が正常に表示されること
- [ ] 短いゴール名が従来通り表示されること
- [ ] 階層表示（親・子ゴール）で表示崩れがないこと
ブランチ名: fix/goal-confirmation-text-overflow
