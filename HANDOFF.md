# 引っ越し手帖 — 引き継ぎ資料（GitHub公開版）

## 1. ファイル構成
```
index.html              ... GitHub Pages公開用本体（localStorage版）
manifest.json           ... PWA設定（ホーム画面に追加するためのアプリ情報）
icon-192.png
icon-512.png
apple-touch-icon.png
move-dashboard.html      ... Claude.aiアーティファクト版（window.storage使用、参考保存用）
HANDOFF.md               ... このドキュメント
```

6つのファイルは**すべて同じ階層（リポジトリのルート）**に置いてください。サブフォルダは不要です。

## 2. 保存方式が変わった点
- `move-dashboard.html`（Claude.ai版）は`window.storage`というClaude.ai専用APIを使用
- `index.html`（GitHub公開版）は標準の**`localStorage`**に変更済み。これでGitHub Pagesのような静的ホスティングでも単体で動作する
- ただしlocalStorageは**ブラウザ・端末ごとに別々**にデータが保存される。iPhoneで使ったデータはPCには無い、という点に注意（同期させたい場合は将来的にバックエンドが必要、9章参照）

## 3. 今のClaude側データをGitHub版に移すには（重要）
1. `move-dashboard.html`をClaude.aiで開く
2. 画面下部の「▼ データの書き出し・読み込み」を開く
3. 「書き出し」のテキストエリアに表示されたJSONを「コピー」ボタンでコピー
4. GitHub Pagesで公開した`index.html`をブラウザで開く
5. 同じく「▼ データの書き出し・読み込み」を開き、「読み込み」のテキストエリアに貼り付けて「読み込む」を押す

これで今のタスク・予算・レイアウトのデータがGitHub版に反映されます。この機能はどちらのファイルにも実装済みなので、今後別の端末にデータを移すときも同じ手順で使えます。

## 4. GitHub Pagesで公開する手順（リポジトリ作成済み：nanopeta/hikkoshi）
1. https://github.com/nanopeta/hikkoshi を開く
2. 「Add file」→「Upload files」を選択
3. `index.html`・`manifest.json`・`icon-192.png`・`icon-512.png`・`apple-touch-icon.png` の5ファイルをまとめて選択してアップロードし、「Commit changes」
4. リポジトリの「Settings」タブ→左メニュー「Pages」→「Build and deployment」の「Source」を **Deploy from a branch** に設定→Branchを **main / (root)** にして「Save」
5. 1〜2分待つと、同じ画面に公開URLが表示される（`https://nanopeta.github.io/hikkoshi/` になるはず）
6. スマホでそのURLを開き、共有メニューから「ホーム画面に追加」を選ぶと、アイコン付きでアプリのように使える

## 5. データモデル（storage keys）
| key | 内容 | 主なフィールド |
|---|---|---|
| `movehub-tasks-v1` | タスク一覧（配列） | `id, title, category, due(YYYY-MM-DD\|null), done` |
| `movehub-budget-v1` | 予算一覧（配列） | `id, category, item, min, max, actual(number\|null), status` |
| `movehub-zones-v1` | レイアウトの部屋（配列） | `id, label, top, left, width, height(すべて%), fill, z` |
| `movehub-furniture-v1` | レイアウトの家具（配列） | `id, label, top, left, width, height(%), color` |

配列の並び順がそのまま表示順（タスク・予算はドラッグ並び替え可）。

## 6. 機能一覧
- **タスク**：追加／削除／完了チェック／ドラッグ並び替え／期限バッジ
- **予算**：追加／削除／編集（min/max/actual/status）／並び替え／自動集計
- **レイアウト**：「間取りを編集」で部屋・家具の追加／削除／ラベル・サイズ・色変更、家具は常時ドラッグ移動可
- **データ書き出し・読み込み**：画面下部のJSONエクスポート／インポート（3章参照）
- **保存の自動リトライ＋トースト通知**：保存失敗時に最大3回再試行、失敗時は再試行ボタン付き警告を表示

## 7. 実装上のポイント
- 全体描画は`render()`によるフルHTML再構築方式（仮想DOMなし）
- テキスト/数値入力中は`render()`を呼ばず、`input`イベントで直接stateとDOMの該当箇所だけ更新（フォーカスが飛ばないようにするため）
- `render()`を呼ぶのは構造が変わる操作のみ：追加／削除／タブ切替／編集モード切替／初回ロード
- リスト並び替え（`enableReorder()`）はドラッグ中にDOM要素を`insertBefore`で直接移動し、`pointerup`時にDOM順序をstate配列に反映
- レイアウトのキャンバスは`padding-top:100%`で正方形固定（幅%と高さ%の基準を揃えるため）

## 8. 既知の制約
- キーボードだけでの並び替えは未対応（Pointer Eventsによるドラッグのみ）
- レイアウトタブの間取りは簡易表現
- `INITIAL_COST_TOTAL`（¥399,846）は2026/6/18付契約金明細書に基づく固定値。変動があれば手動編集
- localStorageは端末・ブラウザごとに独立（3章参照）

## 9. 今後の改善候補
1. 複数端末間でデータ同期したい場合のバックエンド導入（Supabase, Firebase等）
2. 完了済みタスクを隠すフィルタの復活
3. レイアウトのピンチズーム対応
4. Service Worker追加でオフライン完全対応（現状はオフラインでも動くが、明示的なキャッシュ戦略はなし）

## 10. 開発環境
- ビルドステップなし、外部CDN依存なし
- フォントはシステムフォントのみ：'Hiragino Sans','Noto Sans JP',system-ui,sans-serif
- アイコンはPython/Pillowで生成したシンプルな図形（`icons/`内）。差し替えたい場合は同じファイル名・サイズで上書きでOK
