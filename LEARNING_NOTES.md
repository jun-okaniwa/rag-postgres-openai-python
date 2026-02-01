# 学びメモ（見返し＆クイズ用）

このファイルは「理解したこと」を短く記録して、あとで見返したりクイズに使うためのメモです。

---

## 2026-01-31: `__name__` と `__main__`

### 要点
- `__name__` はモジュールに自動で入る「名前」。
- 直接実行されたときだけ `__name__ == "__main__"` になる。
- `import` されたときは `__name__ == "モジュール名"` になる（呼び出し元の名前にはならない）。
- `if __name__ == "__main__":` は「直接実行時だけ動く入口」を作るための定番パターン。

### ミニ例
```python
# a.py
print(__name__)

# b.py
import a
```
- `python a.py` → `__name__` は `"__main__"`
- `python b.py` → `a.py` の `__name__` は `"a"`

---

## クイズ（復習用）
1) `python some_module.py` で直接実行したとき、`__name__` は何になる？  
2) `import some_module` したとき、`some_module` 内の `__name__` は何になる？  
3) `if __name__ == "__main__":` を書く理由は？

---

## 2026-02-01: `logging` と `logger`

### 要点
- `logging` は標準ライブラリのモジュール、`logger` は Logger インスタンス。
- `logging.basicConfig(...)` はログの基本設定（出力レベルなど）を行う。
- `logger.setLevel(logging.INFO)` は「この logger が出す最小レベル」を INFO に設定する。
- INFO 以上（INFO/WARNING/ERROR/CRITICAL）は出るが、DEBUG は出ない。
- ログは「どこかから集める」のではなく、コード内で `logger.info(...)` のように出したもの。

### ミニ例
```python
import logging
logger = logging.getLogger("ragapp")
logging.basicConfig(level=logging.WARNING)
logger.setLevel(logging.INFO)

logger.debug("debug")   # 出ない
logger.info("info")     # 出る
```

---

## 2026-02-01: `async` / `await`

### 要点
- `async def` で定義した関数は「非同期関数」で、呼ぶ側で `await` が必要。
- `await` は「処理が終わるまで待つ」を意味する。
- `asyncio.run(main())` は非同期関数を最後まで実行するための標準的な呼び方。
- DB操作は I/O が多いので、非同期で書かれることが多い。

### ミニ例
```python
import asyncio

async def main():
    await asyncio.sleep(1)
    print("done")

asyncio.run(main())
```

---

## 2026-02-01: 依存解決（pip / requirements / editable install）

### 要点
- `src/backend/pyproject.toml` が **依存の正本**。`requirements.txt` は生成物（ロック）なので手で直し続けない。
- 依存変更後は **ロック再生成 → 再インストール** の順が安全。
- `pip install -e .\src\backend` は **自分のコードをパッケージ登録**する操作。`pyproject.toml` を変えたら再実行が必要。
- `openai-agents` と `openai` は **要求バージョンが揃っていないと即衝突**する。

### 最小の再生成手順
```powershell
python -m piptools compile src\backend\pyproject.toml -o src\backend\requirements.txt --cache-dir .\.piptools-cache
pip uninstall fastapi-app
pip install -e .\src\backend
pip install -r requirements-dev.txt
```

---

---

## 自分用メモ（追記欄）
```
- 
```
