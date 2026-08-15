# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

このリポジトリは、日本の技術情報共有サービスQiita（qiita.com）に投稿する技術記事を管理するためのものです。記事はMarkdown形式で作成し、Qiita CLIツールを使用して管理します。

## リポジトリ構造

- `public/` - 公開済み・下書き記事のMarkdownファイル
- `public/.remote/` - Qiitaとの同期用メタデータ（git管理外）
- `qiita.config.json` - Qiita CLIの設定（プレビューサーバー設定）
- `package.json` - 依存パッケージ（@qiita/qiita-cli）

## よく使うコマンド

### 記事管理
```bash
# 新規記事を作成
npx qiita new <記事名>

# 記事をローカルでプレビュー（localhost:8888でブラウザが開く）
npx qiita preview

# Qiitaから記事を取得して同期
npx qiita pull

# 特定の記事を投稿・更新
npx qiita publish <basename>

# 全ての記事を投稿・更新
npx qiita publish --all
```

### 認証
```bash
# Qiitaにログイン（初回投稿前に必要）
npx qiita login
```

## 記事フォーマット

記事はYAMLフロントマター付きのMarkdownで記述します：
```yaml
---
title: 記事タイトル
tags:
  - タグ1
  - タグ2
private: false          # 限定公開記事の場合はtrue
updated_at: ''          # Qiitaが自動更新
id: null                # Qiitaが自動割当
organization_url_name: null
slide: false            # スライドモードの場合はtrue
ignorePublish: false    # --all指定時にスキップする場合はtrue
---
```

## 開発フロー

1. 新規記事作成：`npx qiita new <記事名>`
2. `public/<記事名>.md`を編集
3. ローカルプレビュー：`npx qiita preview`
4. Qiitaに投稿：`npx qiita publish <記事名>`
5. 変更を同期：`npx qiita pull`
