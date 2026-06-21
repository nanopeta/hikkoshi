# 引っ越し手帖 — CLAUDE.md

## プロジェクト概要
北九州市小倉北区への引越し管理ダッシュボード。  
やること・予算・間取りの3タブ構成。PWA対応（ホーム画面追加可）。

## 技術スタック
- **言語**: Vanilla HTML / CSS / JavaScript（ビルドツールなし）
- **依存**: ゼロ（外部ライブラリ・npm不使用）
- **永続化**: `localStorage`（キー名は `STORAGE_KEYS` 定数を参照）
- **ホスティング**: GitHub Pages（`index.html` のみ静的配信）

## ファイル構成
```
index.html       # アプリ全体（HTML + CSS + JS を1ファイルに集約）
manifest.json    # PWAマニフェスト
icon-192.png     # PWAアイコン 192px
icon-512.png     # PWAアイコン 512px
HANDOFF.md       # 開発経緯・詳細メモ
CLAUDE.md        # このファイル（開発ガイド）
```

## 開発方法

**スマホ（メイン）**: Claude Code Web でコードを編集 → PR マージ → GitHub Pages で確認。

```
# ローカルPC がある場合のみ（ localStorage 動作確認用）
python3 -m http.server 8080
```

`index.html` を直接編集して保存 → ブラウザリロードで確認。

## アーキテクチャ

### 状態管理（index.html `state` オブジェクト）
```js
const state = {
  tab: "tasks" | "budget" | "layout",
  tasks: Task[],
  budget: BudgetItem[],
  zones: Zone[],
  furniture: Furniture[],
  layoutEdit: boolean,    // 間取り編集モード
  dataPanelOpen: boolean, // JSON書き出し/読み込みパネル
  loaded: boolean,
};
```

### レンダリングパターン
- `render()` → `#app` の `innerHTML` を全置換（仮想DOM不使用）
- 入力中のみ DOM を直接更新（フォーカス保持のため）、`debSave()` で遅延保存
- タブ切替・追加・削除時は `render()` を呼ぶ

### 永続化
```js
STORAGE_KEYS = {
  tasks:     "movehub-tasks-v1",
  budget:    "movehub-budget-v1",
  furniture: "movehub-furniture-v1",
  zones:     "movehub-zones-v1",
}
```
- 保存: `saveNow()` / 遅延: `debSave()`（250ms）/ リトライ: 最大3回
- デバイス間移行: 「データの書き出し・読み込み」パネルでJSONコピペ

## 初期データの変更手順

### タスク
`DEFAULT_TASKS` 配列（index.html 内）を編集。
`id` は重複しないよう `t1`, `t2`… の連番で管理。

### 予算
`DEFAULT_BUDGET` 配列を編集。`INITIAL_COST_ITEM`（id:`b15`）は物件初期費用の固定項目。

### 間取り（ゾーン）
`DEFAULT_ZONES` 配列を編集。単位はフロアプラン全体に対する % 。  
ゾーンはドラッグ移動可能（編集モード時）。位置・サイズは数値入力でも変更可。

### 家具
`DEFAULT_FURNITURE` 配列を編集。  
位置はドラッグのみで変更可（数値入力UI は意図的に非表示）。  
サイズ（幅・高さ）は編集パネルの数値入力で変更可。

## データ型定義（参考）

```ts
type Task = {
  id: string;          // "t1", "t2"...
  title: string;
  category: string;
  due: string | null;  // "YYYY-MM-DD" | null
  done: boolean;
};

type BudgetItem = {
  id: string;          // "b1", "b2"...
  category: string;
  item: string;
  min: number;
  max: number;
  actual: number | null;
  status: "未購入" | "商品確定" | "発注済み" | "支払済み" | "後回し";
};

type Zone = {
  id: string;          // "z1", "z2"...
  label: string;
  top: number;         // % (0-100)
  left: number;        // % (0-100)
  width: number;       // % (0-100)
  height: number;      // % (0-100)
  fill: string;        // hex color
  z: number;           // z-index
};

type Furniture = {
  id: string;          // "f1", "f2"...
  label: string;
  top: number;         // % (0-100)
  left: number;        // % (0-100)
  width: number;       // % (0-100)
  height: number;      // % (0-100)
  color: string;       // hex color
};
```

## カラーパレット（COLORS 定数）
```
accent:     #71889F  // ブルーグレー（アクセント）
accentDeep: #54677A  // ダークブルー
beige:      #CDBFA7  // ベージュ
gray:       #AFAFAE  // グレー
danger:     #C16456  // レッド
done:       #7E9A82  // グリーン
```

## 今後の開発アイデア
- [ ] タスクのカテゴリーフィルター
- [ ] タスク完了率プログレスバー（ヘッダー表示）
- [ ] 期日過ぎタスクの強調表示（赤ボーダー等）
- [ ] 予算カテゴリー別集計・サマリー
- [ ] タスク due date でのソート機能
- [ ] 完了済みタスクの折りたたみ表示
- [ ] Service Worker によるオフラインキャッシュ
- [ ] タスク・予算へのメモ欄追加
- [ ] 「後回し」ステータスの予算を別セクション折りたたみ表示

## 注意事項
- localStorage はブラウザ・デバイスごとに独立。複数デバイス運用は JSON 書き出し/読み込みで同期。
- 初期データはlocalStorageに保存済みデータがある場合は使われない（リセットは「初期レイアウトに戻す」ボタン）。
- リセットボタンは zones・furniture のみリセット。tasks・budget は JSON 読み込みで上書きする。
