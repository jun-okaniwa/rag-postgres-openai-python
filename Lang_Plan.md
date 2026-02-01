# Lang Plan（実装方針・刷新版）

作成日: 2026-02-01（JST）
目的: Lang系を使ってコンピテンスを上げつつ、既存のRAGコア品質を崩さずに段階的導入する

---

## 1. ゴール / 非ゴール

### ゴール
- Lang系を「上位層の制御・観測」に限定して導入し、学習と安定運用を両立する
- 既存の **PostgreSQL/pgvector + Hybrid検索（FTS + Vector + RRF）** を維持する
- 混入事故を防ぐため、**metadata契約**と**根拠リンク（source_uri）**を必須化する

### 非ゴール
- LangChain標準VectorStoreへの置き換え
- LangFlowを本番の正本にする
- いきなりLangGraphで複雑なフロー化

---

## 2. 導入原則（守ること）

1) **既存検索ロジックは不変**（DB・SQL・RRF・メタデータフィルタは現行のまま）
2) **Langは薄く使う**（Prompt/Parser/Chain/Graphは“上位層”に限定）
3) **切り戻し可能**（Lang経路と非Lang経路のスイッチを常に用意）
4) **バージョン固定**（langchain系はrequirementsでピン留め）
5) **評価軸は retrieval/metadata/混入ゼロ**（生成一致は使わない）

---

## 3. 段階的導入プラン

### Phase 0: 現状のベースライン固定
- 目的: Lang導入前の挙動を固定し、比較可能にする
- 具体:
  - 代表クエリ5〜10件の出力を記録（回答 + 根拠 + retrieval ID）
  - レイテンシ・コストの目安を控える

### Phase 1: 依存導入と最小構成
- 目的: Lang系を“安全に”試せる足場を作る
- 具体:
  - langchain-core / langchain-openai（最小）を導入
  - 既存コードは触らず、`fastapi_app/lang/` に隔離
  - 依存バージョンを固定

### Phase 2: Prompt / Output Parser だけLang化
- 目的: 生成部分のみLang化して安全に学習する
- 具体:
  - PromptTemplateをLang化
  - 出力パーサー（JSON/構造化）をLang化
  - 検索ロジックは既存のまま

### Phase 3: Retriever Adapter（最重要）
- 目的: “Langの枠組みで既存検索を呼び出す”
- 具体:
  - 既存retrieverを Runnable としてラップ
  - 返却形式に **metadata_contract** を強制
  - A/B比較で retrieval ID が一致することを確認

### Phase 4: LangGraph（必要時のみ）
- 目的: Query Routing / Planning が必要になった場合のみ導入
- 具体:
  - 分岐・再検索・ループが必要な時に最小導入
  - いきなり本番経路に入れない

---

## 4. 評価・検証（Go/No-Go）

### 必須条件（最低限）
- **retrieval同一性**（Top-k IDの一致率が許容範囲）
- **metadata完全性**（variant/revision/doc_type/source_uri/chunk_idが揃う）
- **混入ゼロ**（比較モードでA/B混在を検出しない）
- **根拠リンク必須**（source_uriが常に出力できる）

### 参考条件
- P95レイテンシの悪化が許容範囲内
- コスト（トークン/再検索回数）が許容範囲内

---

## 5. 観測・ログ（LangSmith相当）

- LangSmithが使える場合: 最優先で導入
- 使えない場合: 内製で以下を必ず記録
  - 入力クエリ
  - 取得Top-k ID/score
  - metadata
  - 生成出力
  - レイテンシ

---

## 6. 具体的な実装イメージ（例）

- `fastapi_app/lang/` を新設
  - `prompts.py` : PromptTemplate
  - `parsers.py` : OutputParser
  - `retriever.py` : 既存検索をRunnable化
  - `chain.py` : 最小Chain（retriever → prompt → model → parser）

---

## 7. 運用・学習ルール

- 各Phaseの終わりに **LEARNING_NOTES.md** に学びを追記
- Lang系の導入差分は「小さく・戻せる」単位で積み上げる
- “動けばOK”ではなく、**retrievalとmetadataが壊れてないか**を常に確認する

---

## 8. まとめ（最短一文）

**Langはコアを置換しない。観測と制御の薄い層として導入し、retrieval品質を守ったまま学習する。**

---

## 9. 具体導入案（最小実装のToDo）

### 9.1 Phase 1（依存導入・土台づくり）
- `requirements-dev.txt` に **langchain-core / langchain-openai** を追加し、バージョンを固定  
  例: `langchain-core==x.y.z`, `langchain-openai==x.y.z`
- 既存コードに触れず、`fastapi_app/lang/` を新設
- `fastapi_app/lang/__init__.py` を作成して「隔離領域」を明確化

### 9.2 Phase 2（Prompt/ParserのみLang化）
- `fastapi_app/lang/prompts.py` に PromptTemplate を定義
- `fastapi_app/lang/parsers.py` に OutputParser を定義
- 生成ロジックだけLang化し、**retrievalは既存関数をそのまま使う**

### 9.3 Phase 3（Retriever Adapter）
- `fastapi_app/lang/retriever.py` に「既存検索をRunnable化」した薄いラッパーを実装
- 返却形式に **metadata_contract** を強制
- A/B比較で retrieval ID の一致率を測定

---

## 10. ルーティング・切替設計（最小）

### 10.1 環境変数スイッチ
- `.env` に `RAG_ENGINE=legacy|lang` を追加（デフォルトは `legacy`）
- `create_app()` で `RAG_ENGINE` を読み、ルートを切替

### 10.2 例: 使い分けの考え方
- `legacy` : 既存ルート（現状のRAG）
- `lang` : Langルート（新規実装）

---

## 11. 検証の流れ（最小）

1) ベースライン5〜10件を記録  
2) Langルートで同じクエリを実行  
3) `retrieval ID / metadata / source_uri` を比較  
4) 不一致が出たら **Lang側のみ**修正（既存は触らない）

---

## 12. 失敗時の戻し方（簡単）

- `.env` を `RAG_ENGINE=legacy` に戻す  
- Lang側の実験は `fastapi_app/lang/` だけなので影響範囲が限定される

---

## 13. 依存管理（Lang系導入時の必須手順）

### 13.1 依存の正本
- **`src/backend/pyproject.toml` を正本**とする  
- `src/backend/requirements.txt` は **生成物（ロック）**なので手で直し続けない

### 13.2 依存更新の手順（固定）
1) `pyproject.toml` の範囲を更新（例: openai / openai-agents）  
2) ロック再生成  
   ```powershell
   python -m piptools compile src\backend\pyproject.toml -o src\backend\requirements.txt --cache-dir .\.piptools-cache
   ```
3) editable再インストール（依存メタデータ更新）  
   ```powershell
   pip uninstall fastapi-app
   pip install -e .\src\backend
   ```
4) 依存反映  
   ```powershell
   pip install -r requirements-dev.txt
   ```

### 13.3 代表的な整合ルール（注意点）
- `openai-agents` を上げるなら **openai / pydantic / pydantic-core / typing-inspection** も連動で整合が必要  
- 片方だけピン留めすると **連鎖的に衝突**する
