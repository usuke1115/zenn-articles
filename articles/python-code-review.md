---
title: "Flaskで作ったアプリのコードレビューをしてもらった話"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Python"]
published: true
---

## はじめに
初めてのZennへの投稿になります！
以前、ハッカソンで開発したアプリのコードレビューをしていただいたので、そのときに指摘していただいたことのまとめです。

以下のコードを修正していくという流れで進めていこうと思います！

:::details サインアップ機能のためのコード
```python:controllers/signup.py
from backend import db
from backend.common.models.user import User
import datetime
from flask import Blueprint
from flask import jsonify
from flask import request

signup_bp = Blueprint("signup", __name__, url_prefix="/signup")

@signup_bp.route("/", methods=["POST"])
def signup():
    email = request.json["email"]
    password = request.json["password"]
    
    user = User.query.filter(User.email==email).first()
    if user is not None:
        return jsonify({
            "status": 409,
            "message": "This email address is already in use."
        }), 409
    else:
        new_user = User(name="匿名ユーザ", email=email, password=password)
        dt = datetime.date.today()
        string_date = dt.strftime("%Y.%m.%d")
        current_date = string_date[2:]
        new_user.create_at = current_date
        db.session.add(new_user)
        db.session.commit()
        return {
            "status": 200,
            "data": {
                "id": new_user.id,
                "name": new_user.name,
            }
        }, 200
```
:::

:::message
警告
controllers/signup.pyの処理内容（例えば、パスワードを平文保存しているなど）に関してはあまり深く言及しません。
あくまで、基本的な処理内容はそのままで、レビュワーの視点からコードレビューされていることに注意してください。
:::


## Pythonのimport文のコードスタイル
PEP 8では以下のようにimport文を記述するべきだそうです。
> Imports should be grouped in the following order:
> 1. Standard library imports.
> 2.  Related third party imports.
> 3. Local application/library specific imports.
>
> You should put a blank line between each group of imports.


https://peps.python.org/pep-0008/#imports

つまり、ライブラリをimportする際は
1. 標準ライブラリ
2. サードパーティライブラリ
3. ローカルライブラリ

の順でimport文を記述し、1・2・3の間には空白行を記述するべきだと書かれています。

❌ コードレビュー前のimport文
```python:before
from backend import db
from backend.common.models.user import User
import datetime
from flask import Blueprint
from flask import jsonify
from flask import request
```

⭕️ コードレビュー後のimport文
```python:after
import datetime

from flask import Blueprint
from flask import jsonify
from flask import request

from backend import db
from backend.common.models.user import User
```

手動でもできますが、[isort](https://pycqa.github.io/isort/)や[black](https://pypi.org/project/black/)といったフォーマッターを使うのも良いでしょう。


## リクエストがJSON形式で送られない場合を考慮
いきなりサインアップの処理を書き始めています。このとき、リクエストがJSON形式であることを期待しています。
しかし、もし、リクエストがJSON形式でなかければ、エラーが起きてしまいます。
そのための、エラーハンドリングをしておくと良いでしょう。

❌ コードレビュー前のコード
```python:before
def signup():
    email = request.json["email"]
    # 略
```

⭕️ コードレビュー後のコード
```python:after
def signup():
    if not request.is_json():
        return jsonify({
            "status": 404,
            "message": "Serve internal error."
        }), 404
    
    email = request.json["email"]
```


## メールアドレスとパスワードのバリデーション
バリデーションには[Flask-WTF](https://flask-wtf.readthedocs.io/en/1.2.x/#api-documentation)などのサードパーティライブラリを使います。

もし、バリデーションを定義しないとメールアドレスやパスワードを空で保存できたり、メールアドレスの形式でないのに（例えば, メールアドレスがexample@gmail@gmail^com）保存できてしまいます。

したがって、バリデーションを行いましょう！

❌ コードレビュー前のコード
```python:before controllers/signup.py
def signup():
    email = request.json["email"]
    password = request.json["password"]
    
    user = User.query.filter(User.email==email).first()
```

⭕️ コードレビュー後のコード
```python:after controllers/signup.py
# 略
from backend.common.models.user import User
from backend.common.validater.loginForm import LoginForm
# 略
def signup():
    email = request.json["email"]
    password = request.json["password"]

    form = LoginForm()
    if not form.validate_on_submit():
        return jsonify({
            "status": 400,
            "message": "Email address is invalid or Password is invalid."
        }), 400
```


## ビジネスロジックの記述場所
開発したアプリでは、MVCアーキテクチャを採用していました。
ここで、モデルとコントローラの役割と責任について改めて確認しましょう。

#### モデルの役割と責任
- データの取得・更新・保存
- データの整合性の維持
- データの更新の通知


#### コントローラの役割と責任
- リクエストの処理
- モデルとビューの橋渡し

(ビューに関しては省略します)

このようなモデルとコントローラの役割を曖昧にして開発していました💦
ビジネスロジックはモデルに記述すべきです。

❌ コードレビュー前のコード
```python:before controllers/signup.py
def signup():
    email = request.json["email"]
    password = request.json["password"]
    
    # ビジネスロジックがコントローラに記述してある
    user = User.query.filter(User.email==email).first()
```

コントローラにビジネスロジックが記述されています。これはFat Controllerにつながるため、よくないコードとされています。
ビジネスロジックはモデルに記述しましょう！！

⭕️ コードレビュー後のコード
```python:after controllers/signup.py
def signup():
    email = request.json["email"]
    password = request.json["password"]
    
    # ビジネスロジックが記述されていないため、処理がわかりやすい！！
    user = User.get_user_by_email(email)
```
```python:after models/user.py
class User(db.Model):
    # 略
    @staticmethod
    def get_user_by_email(email: str) -> User | None:
        return User.query.filter(User.email==email).first()
```

ユーザを新規に作成する処理に関しても同様のことが言えます。

❌ コードレビュー前のコード
```python: before controllers/signup.py
else:
    new_user = User(name="匿名ユーザ", email=email, password=password)
    dt = datetime.date.today()
    string_date = dt.strftime("%Y.%m.%d")
    current_date = string_date[2:]
    new_user.create_at = current_date
    db.session.add(new_user)
    db.session.commit()
```

⭕️ コードレビュー後のコード
```python:after controllers/signup.py
else:
    new_user = User.create(email=email, password=password)
```
```python: after models/user.py
class User(db.Model):
    # 略
    @staticmethod
    def create(email: str, password: str, name="匿名ユーザ") -> User:
        user = User(email=email, password=password, name=name)
        db.session.add(user)
        db.session.commit()
        return user
```

また、このようにすることで`datetime`や`db`をimportする必要はなくなりました。

**コントローラにビジネスロジックを書くのはやめましょう！
モデルに記述してください！！**


## early return
`signup`は関数なので、この中で記述しているif文に関してはearly returnをすると良いでしょう。

❌ コードレビュー前のコード
```python:before
if user is not None:
    return jsonify({
        "status": 409,
        "message": "This email address is already in use."
    }), 409
else:
    new_user = User(name="匿名ユーザ", email=email, password=password)
    # 略
    return {
        "status": 200,
        "data": {
            "id": new_user.id,
            "name": new_user.name,
        }
    }, 200
```

⭕️ コードレビュー後のコード
```python:after
if user is not None:
    return jsonify({
        "status": 409,
        "message": "This email address is already in use."
    }), 409

new_user = User.create(email=email, password=password)
return {
    "status": 200,
    "data": {
        "id": new_user.id,
        "name": new_user.name,
    }
}, 200
```
(ここに関しては少し考えればわかったはずですが、徹夜で開発したせいもあり、気付きませんでした💦)


## まとめ
今回はYouTubeで活動されている方に依頼をして、コードレビューしていただきました。
改めて、指摘されたことをまとめると基本的なことが多いように感じます。いかに基本をおろそかにして開発していたかがわかります。

何をするにも基本が大事ですね！

以下はコードレビュー前のコードとコードレビュー後のコードです。

:::details コードレビュー前のコード
```python:controllers/signup.py
from backend import db
from backend.common.models.user import User
import datetime
from flask import Blueprint
from flask import jsonify
from flask import request

signup_bp = Blueprint("signup", __name__, url_prefix="/signup")

@signup_bp.route("/", methods=["POST"])
def signup():
    email = request.json["email"]
    password = request.json["password"]
    
    user = User.query.filter(User.email==email).first()
    if user is not None:
        return jsonify({
            "status": 409,
            "message": "This email address is already in use."
        }), 409
    else:
        new_user = User(name="匿名ユーザ", email=email, password=password)
        dt = datetime.date.today()
        string_date = dt.strftime("%Y.%m.%d")
        current_date = string_date[2:]
        new_user.create_at = current_date
        db.session.add(new_user)
        db.session.commit()
        return {
            "status": 200,
            "data": {
                "id": new_user.id,
                "name": new_user.name,
            }
        }, 200
```
:::

:::details コードレビュー後のコード
```python:controllers/signup.py
from flask import Blueprint
from flask import jsonify
from flask import request

from backend.common.models.user import User
from backend.common.validater.loginForm import LoginForm


signup_bp = Blueprint("signup", __name__, url_prefix="/signup")

@signup_bp.route("/", methods=["POST"])
def signup():
    if not request.is_json():
        return jsonify({
            "status": 404,
            "message": "Serve internal error."
        }), 404

    email = request.json["email"]
    password = request.json["password"]

    form = LoginForm()
    if not form.validate_on_submit():
        return jsonify({
            "status": 400,
            "message": "Email address is invalid or Password is invalid."
        }), 400

    user = User.get_user_by_email(email)
    if user is not None:
        return jsonify({
            "status": 409,
            "message": "This email address is already in use."
        }), 409

    new_user = User.create(email=email, password=password)
    return {
        "status": 200,
        "data": {
            "id": new_user.id,
            "name": new_user.name,
        }
    }, 200
```
:::

## 参考文献
https://peps.python.org/pep-0008/#imports
https://pycqa.github.io/isort/
https://pypi.org/project/black/
https://flask-wtf.readthedocs.io/en/1.2.x/#api-documentation
