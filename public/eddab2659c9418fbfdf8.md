---
title: train_test_splitの使い方
tags:
  - Python
  - ML
  - scikit-learn
private: false
updated_at: '2026-07-12T15:32:56+09:00'
id: eddab2659c9418fbfdf8
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
機械学習タスクを開始する際のデータ分割方法についてです。
機械学習タスクを進める際に、`train`と`test`の分け方,順番をミスる場合が多いので見返す用に書きました。

```bash
pip install scikit-learn
```

```python
from sklearn.model_selection import train_test_split

# X: 特徴量, y: ターゲット変数
# test_size: テストデータの割合 (0.2 = 20%)
# random_state: 実行するたびに結果が変わらないよう固定するシード値
X_train, X_test, y_train, y_test = train_test_split(
    X,
    y,
    test_size=0.2,
    random_state=42
)
```
