# RAG PostgreSQL OpenAI Python プロジェクト - トラブルシューティングガイド

## ⚠️ 重要：会社PCでの完全再現に必要な手順

**Gitで管理されていない必須ファイルがあるため、以下の手順が必要です：**

### 1. `.env`ファイルの作成（必須）
```bash
# .env.company-templateをコピーして.envファイルを作成
cp .env.company-template .env

# .envファイルを編集してOpenAI APIキーを設定
# OPENAICOM_KEY=your_openai_api_key_here を実際のAPIキーに置き換え
```

### 2. フロントエンドのビルド（必須）
```bash
cd src/frontend
npm install
npm run build
cd ../../
```

**注意**: `src/backend/static/`ディレクトリはReact UIのビルド出力で、.gitignoreされているため手動でビルドが必要です。

## 概要
このガイドは、RAG PostgreSQL OpenAI Pythonプロジェクトをクローンから実行まで行う際に発生する可能性のあるエラーと解決策をまとめたものです。

## 前提条件
- Windows環境
- Anaconda/Miniconda インストール済み
- Docker Desktop インストール済み
- Node.js 18+ インストール済み
- OpenAI API キー取得済み

## 発生したエラーと解決策

### 1. Python仮想環境の問題

#### 症状
```
No module named uvicorn
No module named fastapi_app
ModuleNotFoundError: No module named 'xxx'
```

#### 原因
- conda環境が適切にアクティベートされていない
- 依存関係がインストールされていない
- editable installが実行されていない

#### 解決策
```bash
# 新しい仮想環境を作成
conda create -n rag-postgres-openai python=3.12

# 環境をアクティベート
conda activate rag-postgres-openai

# 依存関係をインストール
pip install -r requirements-dev.txt

# バックエンドパッケージをeditable modeでインストール
pip install -e src/backend
```

### 2. Docker Desktop未起動エラー

#### 症状
```
error during connect: Get "http://%2F%2F.%2Fpipe%2FdockerDesktopLinuxEngine/v1.51/containers/json": open //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified.
```

#### 原因
- Docker Desktopアプリケーションが起動していない

#### 解決策
```bash
# Docker Desktopを手動起動
Start-Process "C:\Program Files\Docker\Docker\Docker Desktop.exe"

# 起動を待ってからPostgreSQLコンテナを起動
Start-Sleep -Seconds 15
docker run --name postgres-pgvector -e POSTGRES_PASSWORD=postgres -p 5433:5432 -d pgvector/pgvector:pg17
```

### 3. PostgreSQLデータベース設定エラー

#### 症状
```
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL: database "ragdb" does not exist
```

#### 原因
- `.env`ファイルで`POSTGRES_DATABASE=postgres`になっているが、実際には`ragdb`データベースが必要
- データベース作成とテーブル作成が分離されている

#### 解決策
```bash
# 1. .envファイルを修正
# POSTGRES_DATABASE=ragdb に変更

# 2. ragdbデータベースを作成
docker exec -it postgres-pgvector psql -U postgres -c "CREATE DATABASE ragdb;"

# 3. テーブルを作成
python ./src/backend/fastapi_app/setup_postgres_database.py

# 4. サンプルデータを挿入
python ./src/backend/fastapi_app/setup_postgres_seeddata.py

# 5. データ確認
docker exec -it postgres-pgvector psql -U postgres -d ragdb -c "SELECT COUNT(*) FROM items;"
# 結果: 101件のアイテムが表示されるべき
```

### 4. Node.js環境変数エラー

#### 症状
```
'node' is not recognized as an internal or external command
'npm' is not recognized as an internal or external command
```

#### 原因
- Node.js v24.7.0インストール後にPATH環境変数が更新されていない

#### 解決策
```bash
# オプション1: システム再起動（推奨）
shutdown /r /t 0

# オプション2: 環境変数の手動更新
# System Properties > Environment Variables で Node.js のパスを追加
# C:\Program Files\nodejs\ を PATH に追加
```

### 5. 【最重要】RAG検索機能のTypeErrorエラー

#### 症状
```
ERROR:root:Exception while generating response: 'str' object has no attribute 'query'
Traceback (most recent call last):
  File "...\rag_advanced.py", line 134, in prepare_context
    description=search_results.query,
                ^^^^^^^^^^^^^^^^^^^^
AttributeError: 'str' object has no attribute 'query'
```

#### 原因
- AIエージェントの`search_agent`ツールが`SearchResults`オブジェクトではなく文字列を返すケースがある
- TypeScriptとは異なり、Pythonでは実行時の型チェックが必要

#### 解決策
`src/backend/fastapi_app/rag_advanced.py`の134行目周辺を修正:

**修正前:**
```python
description=search_results.query,
```

**修正後（簡易版）:**
```python
description=search_results.query if hasattr(search_results, 'query') else str(search_results),
```

**修正後（完全版）:**
```python
# search_resultsの型をチェック
if isinstance(search_results, str):
    description = search_results
    filters = []
    items = []
else:
    description = search_results.query
    filters = search_results.filters  
    items = search_results.items
```

### 6. フロントエンド関連エラー

#### 症状
- UIが表示されない
- 静的ファイルが見つからない

#### 原因
- React フロントエンドがビルドされていない
- Node.js依存関係がインストールされていない

#### 解決策
```bash
cd src/frontend

# Node.js依存関係をインストール
npm install

# フロントエンドをビルド
npm run build

# プロジェクトルートに戻る
cd ../../
```

## 最終確認手順

### 1. データベース確認
```bash
# PostgreSQLコンテナの状態確認
docker ps

# データベース内のアイテム数確認
docker exec -it postgres-pgvector psql -U postgres -d ragdb -c "SELECT COUNT(*) FROM items;"
# 期待値: 101件

# サンプルデータ確認
docker exec -it postgres-pgvector psql -U postgres -d ragdb -c "SELECT id, name, type, brand, price FROM items LIMIT 5;"
```

### 2. サーバー起動確認
```bash
# conda環境をアクティベート
conda activate rag-postgres-openai

# FastAPIサーバー起動
python -m uvicorn fastapi_app:create_app --factory --reload
```

### 3. UI確認
ブラウザで `http://localhost:8000` にアクセスして、React UIが正常に表示されることを確認

## テスト用質問（データベース内容確認済み）

以下の質問はデータベースに確実に存在する内容に基づいているため、正常に回答されるべきです：

1. **"How many items are in the database?"** (101件存在)
2. **"What is the Wanderer Black Hiking Boots?"** (ID:1のアイテム)
3. **"Show me items from Daybird brand"** (ブランド検索)
4. **"What's the most expensive item?"** (価格に関する質問)
5. **"List items in the Footwear category"** (タイプ別検索)

## 環境設定ファイル（.env）の最終形

```env
# PostgreSQL設定
POSTGRES_HOST=localhost
POSTGRES_PORT=5433
POSTGRES_USERNAME=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DATABASE=ragdb
POSTGRES_SSL=disable

# OpenAI設定
OPENAI_CHAT_HOST=openai
OPENAI_EMBED_HOST=openai
OPENAICOM_KEY=your_openai_api_key_here

# Azure OpenAI設定（使用する場合）
AZURE_OPENAI_ENDPOINT=https://YOUR-AZURE-OPENAI-SERVICE-NAME.openai.azure.com
AZURE_OPENAI_VERSION=2024-03-01-preview
AZURE_OPENAI_CHAT_DEPLOYMENT=gpt-4o-mini
AZURE_OPENAI_CHAT_MODEL=gpt-4o-mini
AZURE_OPENAI_EMBED_DEPLOYMENT=text-embedding-3-large
AZURE_OPENAI_EMBED_MODEL=text-embedding-3-large
```

## 追加のトラブルシューティング

### ポート競合エラー
```bash
# ポート5433が使用中の場合
docker stop postgres-pgvector
docker rm postgres-pgvector
# 別のポートを使用してコンテナを再作成
```

### 権限エラー（Windows）
```bash
# PowerShellを管理者権限で実行
# Docker コマンドが失敗する場合は、Docker Desktopの設定でWSL2統合を確認
```

### メモリ不足エラー
```bash
# Docker Desktop の Settings > Resources でメモリ割り当てを増加
# 推奨: 最低4GB以上
```

## サポート情報

- **プロジェクト**: RAG on PostgreSQL with OpenAI
- **GitHub**: https://github.com/Azure-Samples/rag-postgres-openai-python
- **Python版本**: 3.12推奨
- **主要依存関係**: FastAPI, SQLAlchemy, OpenAI, React, FluentUI

このガイドに従って段階的に問題を解決することで、プロジェクトを正常に動作させることができます。
