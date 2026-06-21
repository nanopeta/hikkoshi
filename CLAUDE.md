# 引っ越し手帖 — CLAUDE.md

## プロジェクト概要
北九州市小倉北区への引越し管理ダッシュボード。  
やること・予算・レイアウト・基本情報の4タブ構成。PWA対応（ホーム画面追加可）。

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
CLAUDE.md        # このファイル（開発ガイド）
```

## 開発方法

**スマホ（メイン）**: Claude Code Web でコードを編集 → PR マージ → GitHub Pages で確認。

```
# ローカルPC がある場合のみ（localStorage 動作確認用）
python3 -m http.server 8080
```

`index.html` を直接編集 → ブラウザリロードで確認。

---

## アーキテクチャ

### 状態管理（`state` オブジェクト）
```js
const state = {
  tab: "tasks" | "budget" | "layout" | "info",
  tasks: Task[],
  budget: BudgetItem[],
  zones: Zone[],
  furniture: Furniture[],
  info: Info,
  layoutEdit: boolean,     // 間取り編集モード
  dataPanelOpen: boolean,  // JSON書き出し/読み込みパネル
  loaded: boolean,
  taskFilter: string|null, // カテゴリーフィルター（null=すべて）
  budgetFilter: string|null,
  planWidth: number,       // フロアプラン実寸 mm（デフォルト5000）
  planHeight: number,
};
```

### レンダリングパターン
- `render()` → `#app` の `innerHTML` を全置換（仮想DOM不使用）
- テキスト入力中のみ DOM を直接更新（フォーカス保持のため）、`debSave()` で遅延保存
- タブ切替・追加・削除・フィルター変更時は必ず `render()` を呼ぶ
- `render()` の最後に `enableReorder()` / `enableCanvasDrag()` を呼び直してイベントを再アタッチ

### 永続化
```js
STORAGE_KEYS = {
  tasks:     "movehub-tasks-v1",
  budget:    "movehub-budget-v1",
  furniture: "movehub-furniture-v1",
  zones:     "movehub-zones-v1",
  planSize:  "movehub-plansize-v1",
  info:      "movehub-info-v1",
}
```
- 保存: `saveNow()` / 遅延: `debSave()`（250ms）/ リトライ: 最大3回、失敗時はトーストで通知
- デバイス間移行: 「データの書き出し・読み込み」パネルで JSON コピペ

### イベント処理パターン
- クリック: `document.addEventListener("click", ...)` でイベント委譲
  - `data-action` 属性でアクション識別、`data-id` で対象ID
- 入力: `document.addEventListener("input", ...)` + `document.addEventListener("change", ...)`
  - `data-field` / `data-kind` / `data-id` で対象を特定
- 基本情報フィールドは `data-info-field` 属性を使う（`kind` なし）

### ドラッグ実装
**家具・ゾーンの移動（`enableCanvasDrag()`）**
- Pointer Events API（`pointerdown` / `pointermove` / `pointerup`）+ `setPointerCapture`
- `.floor-plan` に `touch-action:none` を CSS で設定済み → `e.preventDefault()` 不要
- 移動中は `drag._live` に座標を保持し、`pointerup` 時のみ state に書き込む
- 家具ボックス内の `.furn-rotate-btn` を押した場合はドラッグ開始をスキップ

**スナップ（`snapToEdges()`）**
- 閾値 `SNAP_THRESHOLD = 3`（%）でX・Y軸独立にスナップ
- キャンバス4辺 + 他の全アイテムの4辺に対して最近傍エッジを探索

**並び替え（`enableReorder()`）**
- カードを **400ms 長押し** → そのままドラッグで並び替え
- `pointerdown` で 400ms タイマー開始、8px 以上移動でキャンセル（スクロール優先）
- タイマー発火後に `setPointerCapture` してドラッグモードに入る
- ドラッグ中は `container.addEventListener("touchmove", e=>e.preventDefault(), {passive:false})` でスクロール抑制
- フィルター中は `enableReorder()` を呼ばない（並び替えを無効化）

### mm ↔ % 変換
```js
const pctToMm = (pct, axis) => Math.round(pct / 100 * (axis==='x' ? state.planWidth : state.planHeight));
const mmToPct = (mm, axis)  => mm / (axis==='x' ? state.planWidth : state.planHeight) * 100;
```
- ゾーン・家具の位置・サイズは内部的に `%` で保持
- 編集パネルの数値入力は `mm` 表示 → 変換して保存

---

## 初期データの変更手順

### タスク
`DEFAULT_TASKS` 配列（index.html 内）を編集。`id` は `t1`, `t2`… の連番。

### 予算
`DEFAULT_BUDGET` 配列を編集。`INITIAL_COST_ITEM`（id:`b15`）は物件初期費用の固定項目。

### 間取り（ゾーン）
`DEFAULT_ZONES` 配列を編集。単位はフロアプラン全体に対する `%`。  
ゾーンはドラッグ移動可能（編集モード時）。位置・サイズは数値入力（mm）でも変更可。  
色は `fill` フィールドで hex 指定。編集モードで `ZONE_SWATCHES` から選択可。

### 家具
`DEFAULT_FURNITURE` 配列を編集。  
位置はドラッグのみで変更可（数値入力UI は意図的に非表示）。  
サイズ（幅・高さ）は編集パネルの数値入力（mm）で変更可。  
回転は常時表示の ↻ ボタン（または編集パネル）で 90° ずつ変更。  
形状は `shape: "rect" | "circle"` で切替可（編集パネルの □/○ ボタン）。

### 基本情報
`DEFAULT_INFO` オブジェクトを編集。localStorage に保存済みの場合は初期値は使われない。

---

## データ型定義

```ts
type Task = {
  id: string;           // "t1", "t2"...
  title: string;
  category: string;
  due: string | null;   // "YYYY-MM-DD" | null
  done: boolean;
};

type BudgetItem = {
  id: string;           // "b1", "b2"...
  category: string;
  item: string;
  min: number;
  max: number;
  actual: number | null;
  status: "未購入" | "商品確定" | "発注済み" | "購入済み" | "支払済み" | "後回し";
};

type Zone = {
  id: string;           // "z1", "z2"...
  label: string;
  top: number;          // % (0-100)
  left: number;         // % (0-100)
  width: number;        // % (0-100)
  height: number;       // % (0-100)
  fill: string;         // hex color（フローリング色）
  z: number;            // z-index
};

type Furniture = {
  id: string;           // "f1", "f2"...
  label: string;
  top: number;          // % (0-100)
  left: number;         // % (0-100)
  width: number;        // % (0-100)
  height: number;       // % (0-100)
  color: string;        // hex color
  rotate: number;       // 0 | 90 | 180 | 270
  shape: "rect" | "circle";
};

type Info = {
  address: string;
  moveInDate: string;   // "YYYY-MM-DD"
  floorPlan: string;
  rent: string;
  managementFee: string;
  monthlyInsurance: string;
  monthlySupport: string;
  electricCompany: string;
  electricNo: string;
  electricOpenDate: string;
  gasCompany: string;
  gasPhone: string;
  gasNo: string;
  gasOpenDate: string;
  waterCompany: string;
  waterPhone: string;
  waterNo: string;
  internetProvider: string;
  internetType: string;
  internetOpenDate: string;
  agentName: string;
  agentPhone: string;
  agentPerson: string;
  memo: string;
};
```

---

## カラーパレット（`COLORS` 定数）
```
accent:     #71889F  // ブルーグレー（アクセント）
accentDeep: #54677A  // ダークブルー
beige:      #CDBFA7  // ベージュ
gray:       #AFAFAE  // グレー
danger:     #C16456  // レッド
done:       #7E9A82  // グリーン
```

`ZONE_SWATCHES` — ゾーン（フローリング）用の10色（淡いトーン）  
`SWATCHES` — 家具色用の6色（`COLORS` から選出）

---

## 今後の開発アイデア
- [ ] 期日過ぎタスクの強調表示（赤ボーダー等）
- [ ] タスク・予算の due date ソート機能
- [ ] 完了済みタスクの折りたたみ表示
- [ ] 「後回し」ステータスの予算を別セクション折りたたみ表示
- [ ] 予算カテゴリー別小計の表示
- [ ] タスク・予算へのメモ欄追加
- [ ] Service Worker によるオフラインキャッシュ
- [x] タスクのカテゴリーフィルター
- [x] 予算のカテゴリーフィルター
- [x] タスク完了率プログレスバー
- [x] 家具の回転（90° ずつ）
- [x] 家具の形状切替（□/○）
- [x] mm 単位の数値入力
- [x] スナップ吸着
- [x] 基本情報タブ
- [x] ゾーンのフローリング色選択

---

## 注意事項
- localStorage はブラウザ・デバイスごとに独立。複数デバイス運用は JSON 書き出し/読み込みで同期。
- 初期データは localStorage に保存済みデータがある場合は使われない。
- 「初期レイアウトに戻す」ボタンは zones・furniture のみリセット。tasks・budget は JSON 読み込みで上書き。
- `render()` 後に `enableReorder()` / `enableCanvasDrag()` を毎回呼び直す設計のため、イベントリスナーの二重登録は起きない（DOM が毎回置換されるため）。
