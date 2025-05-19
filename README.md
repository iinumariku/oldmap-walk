# sv

Everything you need to build a Svelte project, powered by [`sv`](https://github.com/sveltejs/cli).

## Creating a project

If you're seeing this, you've probably already done this step. Congrats!

```bash
# create a new project in the current directory
npx sv create

# create a new project in my-app
npx sv create my-app
```

## Developing

Once you've created a project and installed dependencies with `npm install` (or `pnpm install` or `yarn`), start a development server:

```bash
npm run dev

# or start the server and open the app in a new browser tab
npm run dev -- --open
```

## Building

To create a production version of your app:

```bash
npm run build
```

You can preview the production build with `npm run preview`.

> To deploy your app, you may need to install an [adapter](https://svelte.dev/docs/kit/adapters) for your target environment.

# 古地図さんぽ (Historical Map Walk)

## 概要

「古地図さんぽ」は、大阪・京都・神戸エリアの1922-23年の古地図をモバイルデバイスで閲覧できるWebアプリケーションです。現代の地図と古地図を重ね合わせ、都市の変遷を視覚的に体験することができます。

## 主要機能

- 地図表示機能
  - 国土地理院地図 / OpenStreetMapの切り替え表示
  - 今昔マップ（1922-23年）のオーバーレイ表示
  - 地図の透明度調整による新旧比較
- 位置情報機能
  - GPS位置情報による現在地表示
  - 現在地トラッキング
- メモ機能
  - 地図上の任意の地点へのメモ追加
  - メモの編集・削除
  - メモデータのCSVエクスポート

## 技術スタック

### フロントエンド
- SvelteKit
- TypeScript
- TailwindCSS

### 地図ライブラリ
- MapLibre GL JS

### 開発ツール
- Vite
- ESLint
- Prettier

## 開発環境のセットアップ

```bash
# リポジトリのクローン
git clone https://github.com/yourusername/oldmap-walk.git
cd oldmap-walk

# 依存パッケージのインストール
npm install

# 開発サーバーの起動
npm run dev

# 本番用ビルド
npm run build

# ビルド結果のプレビュー
npm run preview
```

## 実装における技術的特徴

1. **地図表示の最適化**
   - 効率的なタイル管理による描画パフォーマンスの向上
   - ズームレベルに応じた適切な地図データの表示

2. **位置情報の活用**
   - HTML5 Geolocation APIの実装
   - リアルタイムな位置情報トラッキング

3. **ユーザーインターフェース**
   - モバイルファーストのレスポンシブデザイン
   - 直感的な操作性の実現

## プロジェクト構成

```
src/
├── routes/
│   └── +page.svelte  # メインアプリケーション
├── lib/
│   └── components/   # 共通コンポーネント
└── static/          # 静的アセット
```

## ライセンス

本プロジェクトはMITライセンスの下で公開されています。

## 謝辞

- 今昔マップ on the web（京阪神圏1922-23）
- 国土地理院
- OpenStreetMap contributors

---

本プロジェクトは、歴史的地理情報の可視化と最新のWeb技術を組み合わせることで、新しい都市探索体験の創出を目指しています。