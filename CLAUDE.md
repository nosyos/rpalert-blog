# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RPAlert（rpalert.dev）の公式ブログサイト。Hugo + PaperMod テーマで静的サイトを生成し、GitHub Actions でビルド、GitHub Pages で公開する。記事はマークダウンで管理する。

## Build & Development Commands

- `hugo server -D` — ローカル開発サーバー起動（ドラフト記事を含む、デフォルト: http://localhost:1313/blog/）
- `hugo` — サイトを `public/` にビルド
- `hugo new content posts/<slug>.md` — 新しい記事を作成

## Architecture

- **テーマ**: PaperMod（`themes/PaperMod/` に git submodule として配置）
- **カスタム CSS**: `assets/css/extended/custom.css` — rpalert.dev のブランドカラー（ダークネイビー `#0f172a`、アンバー `#f59e0b`）に合わせたスタイル上書き
- **記事**: `content/posts/` にマークダウンファイルとして配置
- **設定**: `hugo.toml` に Hugo 全体の設定（メニュー、パラメータ等）

## Deployment

main ブランチへのプッシュで `.github/workflows/deploy.yml` が Hugo ビルドを実行し、GitHub Pages にデプロイされる。

## Design Notes

rpalert.dev のデザインに合わせたカスタマイズ済み:
- ヘッダー: ダークネイビー背景（`#0f172a`）に白テキスト
- アクセントカラー: アンバー（`#f59e0b` / `#d97706`）をリンク・ホバーに使用
- ライトテーマ固定（ダークモードトグル無効化）
