---
title: 【月1000円×低スペックPC×ローカルLLM構築】
tags:
  - ローカルLLM
private: false
updated_at: '2025-05-11T13:39:06+09:00'
id: ce17cfcbcec296b870cf
organization_url_name: null
slide: false
ignorePublish: false
---
# Flask × transformers × ngrok

こんにちは。今回は **低スペックローカルPCで大規模言語モデル（LLM）を月1000円で動かす方法**に付いて紹介します。

## 🤔抱えていた課題
- 高スペックなGPUは高コスト
- ローカルPCは、macbook air8GB でLLMを動かすことに対してはロースペック
- ローカルLLMを動かしてみたい(ロマン)
- できれば、3000円(Chat-GPT Plus)以下でレスポンスを行えるようにしたい

## ✅ やること
- Google ColabでColab Proを使えるようにする
- 必要となるパッケージのインストール
- HaggingFaceからモデルのダウンロード
- Flask で API サーバーを構築
- ngrok で外部公開して、APIとして利用可能に

---

## 📈ステップ1: ColabProの有効化
Google Colaboratoryとは、Googleが提供しているPythonのオンラインエディタです。近年、環境構築が不要であることから、人気が高まっています。その中で、Google Colab Proというのに課金することで、コンピューティングユニット制限を緩和することができます。
[このページ](https://colab.research.google.com/signup?utm_source=faq&utm_medium=link&utm_campaign=what_types_of_gpus)から、購入してみてください。
![スクリーンショット 2025-04-10 14.32.38.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3992727/40a81be3-839c-4f3d-8c2f-c2faf32fcd91.png)

購入できたら、Google Colabの新たなファイルを開き、ランタイプ>ランタイプの変更>T4GPUを選択しましょう。


## 🚀 ステップ 2：必要となるパッケージのインストール（Colab）
まずは、以下のコードを実行しパッケージのインストールしましょう。

```Python
!pip install transformers accelerate flask flask-ngrok
```
これが実行できたら、それらをインポートして使えるようにしましょう。
```Python
from flask import Flask, request, jsonify
from flask_ngrok import run_with_ngrok
from transformers import AutoTokenizer, AutoModelForCausalLM, pipeline
import torch
```

## ✅ ステップ3：HaggingFaceからモデルのダウンロード
次は、モデルをダウンロードしましょう。自分は、コーディング時のエラー解決等に利用したかったため、OSSのDeepseek Coder(6.7B)を利用しました。他にも、llama3等もいいと思います。
```Python
model_id = "deepseek-ai/deepseek-coder-6.7b-base"  # 必要に応じて他モデルに変更
tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(
    model_id,
    device_map={"": "cuda"},
    torch_dtype=torch.float16,
)
```

## 🔐 ステップ 4：ngrok のセットアップ

ngrok の [ダッシュボード](https://dashboard.ngrok.com/) から **authtoken** を取得し、Colab に設定します。

```bash
# 最新版 ngrok（v3系）を apt 経由でインストール（初回のみ）
!curl -s https://ngrok-agent.s3.amazonaws.com/ngrok.asc | tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null
!echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | tee /etc/apt/sources.list.d/ngrok.list
!apt update && apt install ngrok -y

# ngrok トークン登録（1回のみでOK、再実行してもOK）
!ngrok config add-authtoken YOUR_NGROK_TOKEN(ここを変更！！)
```

## 🧠 ステップ 5：FlaskAPIのバックグラウンド起動
APIにPOSTして返答を取得するような設計にしたいので、/generateに質問をPOSTしたらjson形式で、回答を取得できるようにしました。また、Colabでは同時に複数のセルを実行できないため、バックグラウンドで起動できるようにしました。

```Python
from flask import Flask
import threading
generator = pipeline("text-generation", model=model, tokenizer=tokenizer)
app = Flask(__name__)

@app.route("/generate", methods=["POST"])
def generate():
    try:
        data = request.json
        prompt = data.get("prompt", "")
        output = generator(
            prompt,
            max_new_tokens=2400,
            temperature=0.7,
            do_sample=True,
            pad_token_id=tokenizer.eos_token_id
        )[0]["generated_text"]
        return jsonify({"output": output})
    except Exception as e:
        return jsonify({"error": str(e)}), 500

# Flask起動（5000番ポート）
threading.Thread(target=app.run, kwargs={"port": 5000, "use_reloader": False}).start()
```

## 🌍 ステップ 6：ngrokで公開
自分のローカルのIDEから使えるようにしたいので、以下のようにngrokで公開しましょう。
```Python
# Flask が起動してることを確認してから実行！
get_ipython().system_raw("ngrok http 5000 &")
```

```Python
import time, requests

get_ipython().system_raw("ngrok http 5000 &")
time.sleep(3)

public_url = requests.get("http://localhost:4040/api/tunnels").json()['tunnels'][0]['public_url']
print("✅ 公開URL:", public_url)
```
ここで、得られたURLをコピーしときましょう。

## 🔁 ステップ 7：APIを叩いてみる！

```python
import requests

url = "https://xxxxxx.ngrok.io/generate"  # 上記でコピーしたURL
res = requests.post(url, json={"prompt": "FlaskでAPIのテンプレートコードを書いて"})
print(res.json()["output"])
```
## ✅ 補足：Colabでの注意点
- セッション切断で URL が変わる → ngrok を毎回再接続する必要あり
- max_new_tokens を大きくしすぎると VRAM オーバーでクラッシュ
- DeepSeek 6.7B は T4 でもOK、13B は厳しい
- Colabのセッション切れにより急に使えなくなる場合がある
    以下のようなコードを実行することで、定期的に叩き起こすことができます
    ```Python
    while True:
        print("✅ 活動中... Flaskは動いてます")
        time.sleep(300)  # 5分おきに出力
    ```


## ✨ おわりに
この構成ができると、ChatGPTのようなLLM の APIを自分で持てる状態になります。
LangChainと組み合わせることで、アプリ生成や評価まで自動化できます。
ただ、ColabProでは制限が早いため注意が必要です。
お気に入り・LGTM👍 していただけたら嬉しいです！
質問・改善提案もお気軽にどうぞ😊

