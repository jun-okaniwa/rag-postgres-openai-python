# 方針: rag-postgres-openai-python を理解しながら動かすための段階的ガイド

このファイルは「手順を丸暗記する」のではなく、各ステップの目的と検証方法を明確にして、理解しながら進めるための方針です。
既存のつまずきポイントは `TROUBLESHOOTING_GUIDE.md` を参照し、ここでは「進め方の設計」と「具体手順」をまとめます。

---

## 0. 進め方のルール
- 一度に 1 つだけ進める（環境、DB、バックエンド、フロントの順）
- 各ステップで「目的」「やること」「確認方法」をセットで記録
- 不明点が出たら、先にドキュメントを読む → それでも不明なら実験

---

## 1. 事前確認（環境）
**目的**: 実行に必要な前提が揃っていることを確認する

**やること**:
- Python 3.12 / conda の有無
- Docker Desktop 起動状態
- Node.js (18+) の有無
- OpenAI API キー（または Azure OpenAI）の準備

**確認方法**:
- `python --version`
- `docker ps`
- `node --version` / `npm --version`

---

## 2. リポジトリ構成の理解
**目的**: どのフォルダが何を担当しているか把握する

**やること**:
- `src/backend` と `src/frontend` の役割を読む
- `README` と `docs` を確認

**確認方法**:
- どのコマンドがどの層を動かすか説明できる状態になる

---

## 3. Python 環境の構築
**目的**: バックエンド実行に必要な依存を揃える

**やること**:
- conda 環境作成 → アクティベート
- `requirements-dev.txt` をインストール
- `src/backend` を editable install

**確認方法**:
- `python -m uvicorn --help` が動く
- `pip show fastapi` で依存が入っている

---

### 3-2. 依存更新・衝突時のルール（重要）
**目的**: 依存衝突を確実に解消するための固定手順を守る

**やること**
1) `src/backend/pyproject.toml` を正本として更新  
2) `requirements.txt` を再生成  
3) editable 再インストール  
4) 依存を再適用

**コマンド**
```powershell
python -m piptools compile src\backend\pyproject.toml -o src\backend\requirements.txt --cache-dir .\.piptools-cache
pip uninstall fastapi-app
pip install -e .\src\backend
pip install -r requirements-dev.txt
```

**意味**
- `pyproject.toml` が依存の正本  
- `requirements.txt` は **生成物（ロック）**なので手で直し続けない  
- editable 再インストールで **依存メタデータを更新**

---

## 4. PostgreSQL (pgvector) の準備（具体手順）
**目的**: ベクトル検索用の PostgreSQL を立ち上げ、初期データを投入できる状態にする

### 4-1. Docker で pgvector を起動
**コマンド**
```powershell
docker run --name postgres-pgvector -e POSTGRES_PASSWORD=postgres -p 5433:5432 -d pgvector/pgvector:pg17
```
**意味**
- `--name postgres-pgvector`: コンテナ名を固定して後で参照しやすくする
- `-e POSTGRES_PASSWORD=postgres`: DB の管理ユーザー（postgres）のパスワードを設定
- `-p 5433:5432`: PC の 5433 番ポートをコンテナの 5432 に割り当て（既存 5432 を避けるため）
- `pgvector/pgvector:pg17`: pgvector 拡張入り PostgreSQL イメージ

**確認**
```powershell
docker ps
```
コンテナ `postgres-pgvector` が起動していれば OK。

---

### 4-2. `.env` の DB 設定を確認
**やること**
`/.env` に以下が入っているか確認（なければ追記・修正）

```env
POSTGRES_HOST=localhost
POSTGRES_PORT=5433
POSTGRES_USERNAME=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DATABASE=ragdb
POSTGRES_SSL=disable
```

**意味**
- アプリが接続する DB の接続先を定義
- `POSTGRES_PORT=5433` は 4-1 の `-p 5433:5432` と一致させる

---

### 4-3. ragdb を作成
**コマンド**
```powershell
docker exec -it postgres-pgvector psql -U postgres -c "CREATE DATABASE ragdb;"
```
**意味**
- `docker exec -it`: 起動中コンテナの中でコマンドを実行
- `psql -U postgres`: postgres ユーザーで接続
- `CREATE DATABASE ragdb;`: アプリ用 DB を作成

---

### 4-4. テーブル作成 + 初期データ投入
**コマンド**
```powershell
python .\src\backend\fastapi_app\setup_postgres_database.py
python .\src\backend\fastapi_app\setup_postgres_seeddata.py
```
**意味**
- `setup_postgres_database.py`: テーブルや拡張（pgvector）を作成
- `setup_postgres_seeddata.py`: サンプルデータを投入

**補足（よくあるつまずき）**
- `ModuleNotFoundError: No module named 'fastapi_app'` が出る場合は、`src/backend` が Python の検索パスに入っていない。  
  対処: `pip install -e .\src\backend` を実行してパッケージとして登録する。

---

### 4-5. データ投入確認
**コマンド**
```powershell
docker exec -it postgres-pgvector psql -U postgres -d ragdb -c "SELECT COUNT(*) FROM items;"
```
**意味**
- `-d ragdb`: ragdb に接続
- `SELECT COUNT(*) FROM items;`: items テーブル件数を確認

**期待値**
- `101` が返ってくれば成功

---

**補足（よくあるつまずき）**
- `docker ps` でコンテナが見えない → Docker Desktop が起動していない
- `ragdb does not exist` → 4-3 が未実行、または `.env` の DB 名が違う

---

## 5. バックエンド起動（具体手順）
**目的**: FastAPI が DB と連携できる状態を作る

### 5-1. `.env` の OpenAI 設定を確認
**やること**
`/.env` の OpenAI 設定が正しいか確認（どちらか一方のみ使う）

**OpenAI を使う場合（API キー）**
```env
OPENAI_CHAT_HOST=openai
OPENAI_EMBED_HOST=openai
OPENAICOM_KEY=your_openai_api_key_here
```

**Azure OpenAI を使う場合**
```env
AZURE_OPENAI_ENDPOINT=https://YOUR-AZURE-OPENAI-SERVICE-NAME.openai.azure.com
AZURE_OPENAI_VERSION=2024-03-01-preview
AZURE_OPENAI_CHAT_DEPLOYMENT=gpt-4o-mini
AZURE_OPENAI_CHAT_MODEL=gpt-4o-mini
AZURE_OPENAI_EMBED_DEPLOYMENT=text-embedding-3-large
AZURE_OPENAI_EMBED_MODEL=text-embedding-3-large
```

**意味**
- `OPENAICOM_KEY`: OpenAI API キー
- `AZURE_OPENAI_*`: Azure OpenAI のエンドポイントとデプロイ名

---

### 5-2. FastAPI を起動
**コマンド**
```powershell
python -m uvicorn fastapi_app:create_app --factory --reload
```
**意味**
- `fastapi_app:create_app`: アプリ生成関数を使って起動
- `--factory`: 関数からアプリを生成する場合に必要
- `--reload`: ソース変更時に自動再起動（開発用）

---

### 5-3. 起動確認
**確認方法**
- ブラウザで `http://localhost:8000` を開く
- FastAPI のトップページが表示されれば OK

**補足（よくあるつまずき）**
- `ModuleNotFoundError` → step3 の pip install が不足
- `Error loading ASGI app` → conda 環境が別、または `src/backend` 未インストール

---

## 6. フロントエンドビルド（具体手順）
**目的**: React UI を生成してバックエンドに配信できる状態にする

### 6-1. 依存インストール
**コマンド**
```powershell
cd .\src\frontend
npm install
```
**意味**
- `npm install`: フロントエンド依存をローカルにインストール

---

### 6-2. ビルド
**コマンド**
```powershell
npm run build
cd ..\..
```
**意味**
- `npm run build`: 静的ファイルを生成
- `cd ..\..`: リポジトリ直下に戻る

---

### 6-3. 成果物の確認
**確認方法**
- `src/backend/static/` にファイルが生成されているか確認
- ブラウザで `http://localhost:8000` を開き、UI が表示される

**補足（よくあるつまずき）**
- `node` / `npm` が見つからない → Node.js の PATH 未設定
- UI が表示されない → `npm run build` 未実行、または古い成果物

---

## 7. 動作確認（具体手順）
**目的**: DB と検索が動いていることを確かめる

### 7-1. UI で質問を投げる
**やること**
ブラウザの UI から以下を順に入力

**質問例**
- "How many items are in the database?"
- "What is the Wanderer Black Hiking Boots?"
- "Show me items from Daybird brand"

**期待結果**
- 件数や商品情報が返ってくる
- 返答が空やエラーにならない

---

### 7-2. うまく動かない時の切り分け
**確認ポイント**
- DB が起動しているか（`docker ps`）
- `items` テーブルにデータがあるか（`SELECT COUNT(*) FROM items;`）
- `.env` の OpenAI 設定が有効か

---

## 8. つまずいたら（具体手順）
**目的**: 既知の問題から素早く原因を切り分ける

### 8-1. まずやること
**やること**
- エラーメッセージをそのままコピー
- どのステップで起きたかをメモ

---

### 8-2. 既知の問題を探す
**やること**
- `TROUBLESHOOTING_GUIDE.md` を開く
- キーワード一致で該当箇所を探す

---

### 8-3. 解決できなければ追記
**やること**
- 再現条件、解決策、実行したコマンドを追記
- 同じミスを防ぐために「原因」を 1 行で書く

---

## 進捗メモ（テンプレ）
```
### 日付:
### 進めたステップ:
### 目的:
### やったこと:
### 結果:
### つまずき:
### 次にやること:
```

---

必要なら、この方針をもとに「1ステップずつ解説」して進めます。
