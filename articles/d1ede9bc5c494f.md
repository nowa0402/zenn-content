---
title: "Pydantic aliasでjsonキーを自在に操りたい"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "pydantic"]
published: true
---

## モデルのフィールド名とキー名を異なるものにしたい

pydanticでjsonをモデルに読み込む場合、jsonのキーとモデルのフィールドは一致している必要があります。

```python
from pydantic import BaseModel


class User(BaseModel):
    name: str
    age: int


# nameキーとageキーが一致している
data = '{"name": "test_user", "age": 23}'

user = User.model_validate_json(data)
print(user) # name='test_user' age=23

# NameキーはUserモデルのフィールドに存在しないのでValidationErrorが送出される
data2 = '{"Name": "test_user", "age": 23}'
user2 = User.model_validate_json(data2)
print(user2)
```

仮にjsonキーがパスカルケース（単語の先頭を大文字にする）だった場合、
フィールドもそれに合わせる必要があります。

```python
from pydantic import BaseModel


class User(BaseModel):
    Name: str
    Age: int


data = '{"Name": "test_user", "Age": 23}'

user = User.model_validate_json(data)
print(user)  # Name='test_user' Age=23
```

このコード、動きはします。
しかし、Pythonの[命名規則](https://pep8-ja.readthedocs.io/ja/latest/#id37)を破っている感じがして、もやもやします。

これをなんとかする方法を解説します。

### 検証環境

今回はpydantic v2を使用して検証しました。
pydanticのバージョンは以下のとおりです

```bash
pydantic==2.7.2
```

### Fieldのaliasオプションを使う

キー名を別で指定する方法に`Field`の`alias`オプションを使う手があります。
`alias`の引数にフィールドと紐づけたい値を書くことで、フィールドとキー名の紐づけを行います。

https://docs.pydantic.dev/latest/concepts/fields/#field-aliases

`model_dump`メソッドでモデルを辞書化するときに、`by_alias`オプションを`True`にすれば、`alias`で指定した名前をキー名として出力してくれます

```python
from pydantic import BaseModel, Field


class User(BaseModel):
    name: str = Field(..., alias="Name")
    age: int = Field(..., alias="Age")


data = '{"Name": "test_user", "Age": 23}'

user = User.model_validate_json(data)
print(user)  # name='test_user' age=23
print(user.model_dump())  # {'name': 'test_user', 'age': 23}
print(user.model_dump(by_alias=True))  # {'Name': 'test_user', 'Age': 23}
print(user.model_dump_json(by_alias=True))  # {"Name":"test_user","Age":23}
```

別名を指定する方法は他に`validate_alias`と`serialization_alias`が存在します。

`validate_alias`はモデルをインスタンス化するときの検証にしか使われません。モデルの辞書・json変換時に`by_alias`パラメータを`True`にしてもフィールド名が採用されます。

```python
from pydantic import BaseModel, Field


class User(BaseModel):
    name: str = Field(..., validation_alias="Name")
    age: int = Field(..., validation_alias="Age")


data = '{"Name": "test_user", "Age": 23}'

user = User.model_validate_json(data)
print(user)  # name='test_user' age=23
print(user.model_dump())  # {'name': 'test_user', 'age': 23}
print(user.model_dump(by_alias=True))  # {'name': 'test_user', 'age': 23}
print(user.model_dump_json(by_alias=True))  # {"name":"test_user","age":23}
```

`serialization_alias`は挙動が先ほど挙げた2つと異なり、インスタンス化するときのフィールド名とキー名は一緒で、変換時の名前だけを指定して変更することができます。

```python
from pydantic import BaseModel, Field


class User(BaseModel):
    name: str = Field(..., serialization_alias="Name")
    age: int = Field(..., serialization_alias="Age")


data = '{"name": "test_user", "age": 23}'

user = User.model_validate_json(data)
print(user)  # name='test_user' age=23
print(user.model_dump())  # {'name': 'test_user', 'age': 23}
print(user.model_dump(by_alias=True))  # {'Name': 'test_user', 'Age': 23}
print(user.model_dump_json(by_alias=True))  # {"Name":"test_user","Age":23}

```

### 複数取りうる値がある場合はAliasChoiceを使う

1対1の紐づけはできるようになりました。
ただ、場合によっては取りうる値が複数あるときもあると思います。

Userモデルのフィールド`name`に紐づけられるキー名が`Name`と`username`どちらかが来る可能性を考えます。

この場合、先ほどの`alias`オプションでは対応できません。

```python
from pydantic import BaseModel, Field


class User(BaseModel):
    name: str = Field(..., alias="Name")
    age: int = Field(..., alias="Age")


data = '{"Name": "test_user", "Age": 23}'
user = User.model_validate_json(data)
print(user)  # Name='test_user' Age=23

data2 = '{"username": "test_user", "Age": 23}'
user_2 = User.model_validate_json(data2)  # usernameを受け取れないので例外発生
print(user_2)
```

これを受け取れるようにするために、`AliasChoices`クラスを使用します。

https://docs.pydantic.dev/latest/concepts/alias/#aliaspath-and-aliaschoices

このクラスは`validation_alias`パラメータ限定で指定できるものです。
引数に取りうるエイリアスを複数記載することで、いずれかに該当した場合にフィールド値と紐づけを行ってくれます

```python
from pydantic import AliasChoices, BaseModel, Field


class User(BaseModel):
    name: str = Field(..., validation_alias=AliasChoices("Name", "username"))
    age: int = Field(..., validation_alias="Age")


data = '{"Name": "test_user", "Age": 23}'
user = User.model_validate_json(data)
print(user)  # Name='test_user' Age=23

data2 = '{"username": "test_user", "Age": 23}'
user_2 = User.model_validate_json(data2)
print(user_2) # name='test_user' age=23
```

紐づけを行うキーが同時に存在した場合は、AliasChoiceで記載した順で優先度が決まります。

```python
from pydantic import AliasChoices, BaseModel, Field


class User(BaseModel):
    name: str = Field(..., validation_alias=AliasChoices("Name", "username"))
    age: int = Field(..., validation_alias="Age")


# キーの順場ではなく、AliasChoiceの順番で決まる
data = '{"username": "test_user", "Name": "test", "Age": 23}' 
user = User.model_validate_json(data)
print(user)  # name='test' age=23

data_2 = '{"Name": "test", "username": "test_user", "Age": 23}'
user_2 = User.model_validate_json(data_2)
print(user_2)  # name='test' age=23
```

`AliasChoice`は`*args`を受け取るのでリストのアンパックで渡すこともできます

```python
from pydantic import AliasChoices, BaseModel, Field


class User(BaseModel):
    name: str = Field(..., validation_alias=AliasChoices(*["Name", "username"]))
    age: int = Field(..., validation_alias="Age")
```

`alias`と組み合わせることで、入力キーと出力キーを個別に設定できます。

```python
from pydantic import AliasChoices, BaseModel, Field


class User(BaseModel):
    name: str = Field(
        ..., alias="user_name", validation_alias=AliasChoices(*["Name", "username"])
    )
    age: int = Field(..., validation_alias="Age")


data = '{"username": "test_user", "Name": "test", "Age": 23}'
user = User.model_validate_json(data)
print(user)  # name='test' age=23
print(user.model_dump(by_alias=False))  # {'name': 'test', 'age': 23}
print(user.model_dump(by_alias=True))  # {'user_name': 'test', 'age': 23}
print(user.model_dump_json(by_alias=True))  # {"user_name":"test","age":23}
```

## Aliasを使いこなして開発体験を向上させよう

pydanticはFastAPIと組み合わせて使われることが多いですが、機能が優秀なので単独でも大活躍です。
今回紹介したもの以外にも

- [値の検証](https://docs.pydantic.dev/latest/concepts/validators/#field-validators)
- [JSONSchema定義](https://docs.pydantic.dev/latest/concepts/json_schema/)
- [シリアル化](https://docs.pydantic.dev/latest/api/functional_serializers/#pydantic.functional_serializers.field_serializer)

など、便利な機能がたくさんあります。
私の記事ではシリアル化の簡単な説明記事がありますので、興味があればぜひ読んでみてください！

https://zenn.dev/nowa0402/articles/6b423af53274c2

## 参考文献

- [pydantic公式](https://docs.pydantic.dev/latest/)
  - [FieldAlias](https://docs.pydantic.dev/latest/concepts/fields/#field-aliases)
  - [Alias](https://docs.pydantic.dev/latest/concepts/alias/)
