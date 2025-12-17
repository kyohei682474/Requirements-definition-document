# 仕様書: レスポンシブヘッダー検索バー修正

## ブランチ名

`fix/mobile-search-bar-width`

## 概要

レスポンシブデザイン時のモバイルヘッダーにおいて、検索バーの幅がFigmaデザインと異なっている問題を修正する。

## 現状の問題

| 項目 | 現在の実装 | Figmaデザイン |
|------|-----------|--------------|
| 検索バー幅 | `460px` 固定 | 可変（コンパクト） |
| 高さ | `26px` | 同等 |
| 表示 | モバイルでも460px（はみ出る） | 画面幅に収まるサイズ |

### 原因

`SpotLightSearchButton.tsx` L43 で `width: "460px"` がインラインスタイルで固定されている。
モバイル画面（390px幅）では画面からはみ出す。

## 対象

全ページ共通のモバイルヘッダー（`md:hidden` で表示）

## 修正対象ファイル

- `src/goals/[id]/components/search-bar/SpotLightSearchButton.tsx`

## 現在のヘッダー構成

```
MobileHeader (src/components/responsive/MobileHeader.tsx)
├── 左側
│   ├── HamburgerMenu (≡)
│   └── MobileGoalTree (サイドバーアイコン)
└── 右側
    └── SpotLightSearchButton (検索)
```

## 修正内容

### SpotLightSearchButton のレスポンシブ対応

**Before (L41-53):**
```tsx
style={{
  display: "flex",
  width: "460px",        // ← 固定幅が問題
  height: "26px",
  // ...
}}
```

**After:**
```tsx
style={{
  display: "flex",
  height: "26px",
  // width を削除
  // ...
}}
className="group w-auto px-4 md:w-[460px]"  // ← Tailwindでレスポンシブ対応
```

### 修正ポイント

1. インラインスタイルの `width: "460px"` を削除
2. Tailwindクラスで `w-auto md:w-[460px]` を追加
3. モバイル時のパディング調整 `px-4`

## 受け入れ条件

- [ ] モバイル表示（390px幅）で検索バーが画面内に収まる
- [ ] Figmaデザインと検索バーのサイズ感が一致する
- [ ] デスクトップ表示（md以上）では既存の460px幅を維持
- [ ] 検索モーダルが正常に開く
- [ ] キーボードショートカット（Cmd+F / Ctrl+F）が動作する

## 備考

- ユーザーアイコンはFigmaにあるが現在未実装（本チケット対象外）


 

