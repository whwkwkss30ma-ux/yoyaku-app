# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

奈良県山間部のアウトドア施設（空中ウォーク・ツリーハウス昼・BBQスペース・宿泊施設11棟）向けの予約管理アプリ。**単一HTMLファイル**（`yoyaku (2).html`）で完結している。ビルドツール・パッケージマネージャ・テストフレームワークは一切ない。

**対象ブラウザ**: iPad第6世代 Safari、Android Chrome、PC各種ブラウザ  
**デプロイ**: GitHub Pages（静的ホスティング）

## 開発フロー

ビルドプロセスなし。HTMLファイルを直接編集してブラウザで開くだけ。

```
# ローカル確認
# ブラウザで yoyaku (2).html を直接開く
# または簡易サーバーを使う場合:
python -m http.server 8080
```

## アーキテクチャ

### ファイル構成

すべてのCSS・JS・HTMLが `yoyaku (2).html` 1ファイルに収まっている。

### データ永続化（localStorage）

| キー | 内容 | 変数 |
|------|------|------|
| `yv_f` | カスタムフィールド定義 | `fields` |
| `yv_r` | 日帰り予約一覧 | `reservations` |
| `yv_st` | 宿泊予約一覧 | `stays` |
| `yv_sk` | 備品在庫数 | `stock` |
| `yv_sn` | 外部予約サイト名 | `sn` |
| `yv_gas` | GAS WebアプリURL | — |

ヘルパー: `ld(key)` で読み込み、`sv(key, val)` で保存。

### グローバル状態

```js
// 表示・フォーム状態
let facFilter='all', addFacId='walk', addCourse=55;
let editId=null, editType='day';
let rentalSt={}, stayRentalSt={};
let dateFilter=null;
```

### タブ構成

`switchTab(t)` でタブ切替。各タブの初期化処理:
- `list`: `renderFltTabs()` → `renderList()`
- `day`: `setFac('walk')` → 各フォーム初期化
- `stay`: `initStay()`
- `cal`: `initCal()`
- `settings`: `renderSettings()` / `renderStock()` / `loadSNUI()` / `loadGasUrl()`

### マスターデータ（ハードコード）

- **`FACILITIES`**: 日帰り施設3件（walk/tree/bbq）。各施設に `courses`・`prices`・`close`・`priceUnit`・`fixedCount` を定義
- **`STAY_FACS`**: 宿泊施設11棟。`maxA`（大人定員）・`maxT`（合計定員）・`bat`（バッテリー種別）・`prices`（人数→単価マップ）を持つ
- **`ALL_ITEMS`**: レンタル備品23品目。`col:true` のものは「その他」として折りたたみ表示

価格・施設情報を変更する場合はこれらの配列を直接編集する。

### 予約の紐づけ（双方向同期）

日帰り予約と宿泊予約は相互参照できる:
- 日帰り側: `linkedStayIds[]`, `linkedDayIds[]`
- 宿泊側: `linkedDayIds[]`

保存・編集時に双方向同期コード（`forEach` でお互いのIDを追記・削除）が必ず実行される。紐づけ変更時はこの同期を維持すること。

### 料金計算の仕組み

- **日帰り**: `calcTime()` — `FACILITIES[].prices[course]` × 人数。`mura`（村内）フラグで空中ウォーク/ツリーハウスは半額、BBQ/レンタルは無料
- **宿泊**: `calcStay()` — `getSPrice(fac, totalPpl)` で人数段階別単価を取得 × 大人人数 × 泊数 + ペット料金(3000円/匹/泊) + レンタル
- **レンタル**: `calcRT(state)` — `state[id].checked && state[id].qty` の合計金額

### Android対応: date inputのポーリング

Androidでは `<input type="date">` の `oninput`/`onchange` が安定して発火しない。そのため:

```js
// フォーカス時にポーリング開始、ブラー時に停止
onfocus="startDatePoll('day-date', updDayDateDisp)"
onblur="stopDatePoll()"
```

`startDatePoll` は200msごとに値変化を検知する。新たに `<input type="date">` を追加する場合は同パターンを踏襲すること。

### 日付の正規化

`normDate(d)` は `YYYYMMDD` と `YYYY-MM-DD` の両形式を受け付ける（ブラウザにより返却形式が異なる）。  
`fmtDate(d)` は `YYYY-MM-DD` を「4月27日（日）」形式に変換する。

### カードのイベントデリゲーション

一覧カードのボタンはすべて `data-act` 属性で管理し、`document.addEventListener('click', ...)` でまとめて処理する。新規アクションを追加する場合はこのデリゲーターに分岐を追加する。

## iOS/Android 互換性の注意点

- 入力欄には `-webkit-appearance:none` を指定済み（Safariのデフォルトスタイルを無効化）
- モーダルのスクロールに `-webkit-overflow-scrolling:touch` と `overscroll-behavior:contain` を使用
- `navigator.clipboard` が使えない場合の fallback として `document.execCommand('copy')` を使用（`copyHandover` / `exportData` 参照）
- `<input type="date">` の曜日フィールドを非表示にするCSSが複数ある（Brave/Safari対応）

## 外部連携

Googleスプレッドシートへの同期はGAS Web App経由のPOSTリクエスト（`syncToSheet()`）。URLは設定タブで入力しlocalStorageに保存。本アプリ自体はオフライン動作する。
