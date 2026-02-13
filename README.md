# Pix Coords

画像上のピクセル座標を取得・管理するためのWebアプリケーションです。

## 機能

- **画像の読み込み** — ドラッグ&ドロップまたはファイル選択で画像を読み込めます（TIFF形式にも対応）
- **ピクセル座標の取得** — マウスカーソル位置のピクセル座標(X, Y)をリアルタイムに表示
- **ピクセル色の取得** — カーソル位置のピクセルカラー(RGB / HEX)を表示
- **拡大鏡** — カーソル周辺をズームしたルーペを表示し、ピクセル単位の確認が可能
- **ズーム & パン** — ホイールでズーム、Alt+ドラッグ / 中クリックでパン操作
- **座標リストの管理** — クリックした座標をリストに記録し、名前を付けて管理
- **エクスポート** — 座標リストをCSV / XLSX形式でエクスポート

## 技術スタック

- [Svelte 5](https://svelte.dev/) + TypeScript
- [Vite](https://vitejs.dev/)
- [utif2](https://github.com/nicokoenig/utif2) — TIFF画像の読み込み
- [xlsx](https://sheetjs.com/) — XLSXエクスポート

## セットアップ

```bash
npm install
npm run dev
```

## ビルド

```bash
npm run build
npm run preview
```
