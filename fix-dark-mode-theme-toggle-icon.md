# 仕様書：ダークモード時のテーマ切替アイコン視認性修正

## 1. 概要

- **対象機能**: マイページ > テーマ設定 > 表示モード切替アイコン
- **対象デバイス**: レスポンシブ（モバイル）
- **修正種別**: UIバグ修正

## 2. 問題の詳細

### 2.1 再現手順

1. レスポンシブ表示（モバイル幅）でアプリを開く
2. ハンバーガーメニューからマイページを開く
3. 表示モードを「ダーク」に変更
4. 再度表示モードを変更しようとする
5. **問題:** 切替アイコン（月アイコン）が背景色と同化して見えなくなる

### 2.2 原因

`ModeToggle.tsx` のアイコンにダークモード用の色指定がない。

現状（L26-27）:

```tsx
<Sun className="h-[1.2rem] w-[1.2rem] scale-100 rotate-0 transition-all dark:scale-0 dark:-rotate-90" />
<Moon className="absolute h-[1.2rem] w-[1.2rem] scale-0 rotate-90 transition-all dark:scale-100 dark:rotate-0" />
```

- アイコンのデフォルト色（currentColor）がダークモードの背景と同系色
- 明示的な色指定がないため視認できない

### 2.3 影響

- ダークモード時にテーマ切替ボタンが見つけにくい
- ユーザーがテーマを変更できない（操作性低下）

## 3. 修正内容

### 3.1 対象ファイル

`src/features/sidebar/components/ModeToggle.tsx`

### 3.2 修正箇所

**Before (L26-27):**

```tsx
<Sun className="h-[1.2rem] w-[1.2rem] scale-100 rotate-0 transition-all dark:scale-0 dark:-rotate-90" />
<Moon className="absolute h-[1.2rem] w-[1.2rem] scale-0 rotate-90 transition-all dark:scale-100 dark:rotate-0" />
```

**After:**

```tsx
<Sun className="h-[1.2rem] w-[1.2rem] scale-100 rotate-0 transition-all dark:scale-0 dark:-rotate-90 text-foreground" />
<Moon className="absolute h-[1.2rem] w-[1.2rem] scale-0 rotate-90 transition-all dark:scale-100 dark:rotate-0 text-foreground" />
```

### 3.3 追加するCSSクラス

- **text-foreground**: テーマに応じた前景色を適用（ライト: 黒系、ダーク: 白系）

## 4. 期待される動作

- **ライトモード**: アイコン表示 ✓（変化なし）
- **ダークモード**: アイコン見えない ✗ → アイコン表示 ✓

## 5. 影響範囲

- マイページのテーマ切替ボタン
- サイドバーで同コンポーネントを使用している場合はそちらも改善
