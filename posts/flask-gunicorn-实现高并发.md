+++
title = "flask + gunicorn 实现高并发"
date = "2023-07-24"
tags = ["flask"]

+++



基于 flask 写了一个 hanlp 分词 api，想提高 api 的并发量。




## 分词 api 编写

<u>app.py</u>

```python
import os
import hanlp
import json
import logging
from flask import Flask, request, jsonify, current_app as app, Response

app = Flask(__name__)
os.environ["CUDA_VISIBLE_DEVICES"] = "0,1"

tokenizer = hanlp.load(hanlp.pretrained.tok.COARSE_ELECTRA_SMALL_ZH)
tok_fine = hanlp.load(hanlp.pretrained.tok.FINE_ELECTRA_SMALL_ZH)

@app.route("/hanlp", methods=["GET", "POST"])
def hlp():
    if request.method == "GET":
        return jsonify({
            "status": "not ok",
            "message": "Get Method Is Not Allowed."
        }), 405  # HTTP status code for Method Not Allowed

    # Retrieve options
    options = request.get_json(silent=True) or {}
    text = options.get("text", "")
    method = options.get("method", "")

    app.logger.debug(text)
    HanLP = hanlp.pipeline().append(hanlp.utils.rules.split_sentence).append(
                tok_fine if method == "fine" else tokenizer
            ).append(lambda sents: sum(sents, []))
    result = HanLP(text)

    return Response(
        response=json.dumps(result, ensure_ascii=False),
        status=200, mimetype="application/json; charset=utf-8"
    )

@app.after_request
def after_request(response):
    app.logger.debug(response.get_json(silent=True))
    return response

if __name__ == '__main__':
    app.run(host="0.0.0.0", port=5000, debug=True)
```

 

## 安装 gunicorn 库

```bash
pip install gunicorn
```



## 使用 gunicorn 启动 flask app

```bash
gunicorn -w 10 -b "0.0.0.0:5000" "app:app"
```

参数解释：

`-w` worker, 并发数

`-b` 监听地址

