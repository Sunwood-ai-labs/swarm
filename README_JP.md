# Swarm (実験的サンプル)

エルゴノミックで軽量なマルチエージェントオーケストレーションフレームワーク

> [!WARNING]
> Swarmは現在、マルチエージェントシステムのエルゴノミックなインターフェースを探求するための実験的なサンプルフレームワークです。本番環境での使用を意図しておらず、公式サポートもありません。（そのため、PRやIssueのレビューも行いません！）
>
> Swarmの主な目的は、[Orchestrating Agents: Handoffs & Routines](https://cookbook.openai.com/examples/orchestrating_agents)クックブックで探求されているハンドオフとルーチンのパターンを紹介することです。スタンドアロンのライブラリとしてではなく、主に教育目的で提供されています。

## インストール

Python 3.10以上が必要です

```shell
pip install git+ssh://git@github.com/openai/swarm.git
```

または

```shell
pip install git+https://github.com/openai/swarm.git
```

## 使用方法

```python
from swarm import Swarm, Agent

client = Swarm()

def transfer_to_agent_b():
    return agent_b

agent_a = Agent(
    name="Agent A",
    instructions="あなたは役立つエージェントです。",
    functions=[transfer_to_agent_b],
)

agent_b = Agent(
    name="Agent B",
    instructions="俳句でのみ話してください。",
)

response = client.run(
    agent=agent_a,
    messages=[{"role": "user", "content": "Agent Bと話したいです。"}],
)

print(response.messages[-1]["content"])
```

```
希望の光
新しい道交わる
何をお手伝い
```

## 目次

- [概要](#概要)
- [例](#例)
- [ドキュメント](#ドキュメント)
  - [Swarmの実行](#swarmの実行)
  - [エージェント](#エージェント)
  - [関数](#関数)
  - [ストリーミング](#ストリーミング)
- [評価](#評価)
- [ユーティリティ](#ユーティリティ)

# 概要

Swarmは、エージェントの**調整**と**実行**を軽量で、高度に制御可能、そして簡単にテスト可能にすることに焦点を当てています。

これは、`Agent`と**ハンドオフ**という2つの基本的な抽象化を通じて実現されています。`Agent`は`instructions`と`tools`を包含し、いつでも会話を別の`Agent`にハンドオフすることができます。

これらのプリミティブは、ツールとエージェントのネットワーク間の豊かな動的関係を表現するのに十分な力を持っており、急峻な学習曲線を避けながら、スケーラブルで実世界のソリューションを構築することができます。

> [!NOTE]
> SwarmのエージェントはAssistants APIのAssistantsとは関係ありません。便宜上似た名前がついていますが、それ以外は全く無関係です。SwarmはChat Completions APIのみを使用しており、そのため呼び出し間でステートレスです。

## なぜSwarmなのか

Swarmは設計上、軽量で、スケーラブル、そして高度にカスタマイズ可能です。単一のプロンプトにエンコードすることが難しい、多数の独立した機能と指示を扱う状況に最適です。

Assistants APIは、完全にホストされたスレッドと組み込みのメモリ管理および検索機能を求める開発者にとって素晴らしいオプションです。一方、Swarmは、コンテキスト、ステップ、ツール呼び出しに対する完全な透明性と細かい制御を求める開発者に最適です。Swarmは（ほぼ）完全にクライアント上で動作し、Chat Completions APIと同様に、呼び出し間で状態を保存しません。

# 例

インスピレーションを得るために`/examples`をチェックしてください！各例の詳細はそのREADMEをご覧ください。

- [`basic`](examples/basic): セットアップ、関数呼び出し、ハンドオフ、コンテキスト変数など、基本的な要素の簡単な例
- [`triage_agent`](examples/triage_agent): 適切なエージェントにハンドオフするための基本的なトリアージステップを設定する簡単な例
- [`weather_agent`](examples/weather_agent): 関数呼び出しの簡単な例
- [`airline`](examples/airline): 航空会社のコンテキストで異なる顧客サービスリクエストを処理するマルチエージェントセットアップ
- [`support_bot`](examples/support_bot): ユーザーインターフェースエージェントと複数のツールを持つヘルプセンターエージェントを含む顧客サービスボット
- [`personal_shopper`](examples/personal_shopper): 販売や注文の払い戻しを支援できるパーソナルショッピングエージェント

# ドキュメント

![Swarm Diagram](assets/swarm_diagram.png)

## Swarmの実行

まず、Swarmクライアントをインスタンス化します（内部的には単に`OpenAI`クライアントをインスタンス化します）。

```python
from swarm import Swarm

client = Swarm()
```

### `client.run()`

Swarmの`run()`関数は、Chat Completions APIの`chat.completions.create()`関数に類似しています - `messages`を受け取り、`messages`を返し、呼び出し間で状態を保存しません。ただし、重要な点として、エージェントの関数実行、ハンドオフ、コンテキスト変数の参照を処理し、ユーザーに戻る前に複数のターンを取ることができます。

本質的に、Swarmの`client.run()`は以下のループを実装しています：

1. 現在のエージェントから完了を取得
2. ツール呼び出しを実行し、結果を追加
3. 必要に応じてエージェントを切り替え
4. 必要に応じてコンテキスト変数を更新
5. 新しい関数呼び出しがなければ、返却

#### 引数

| 引数                  | 型      | 説明                                                                                                                                                   | デフォルト     |
| --------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------- |
| **agent**             | `Agent` | （初期の）呼び出されるエージェント。                                                                                                                   | (必須)         |
| **messages**          | `List`  | メッセージオブジェクトのリスト。[Chat Completions `messages`](https://platform.openai.com/docs/api-reference/chat/create#chat-create-messages)と同一。 | (必須)         |
| **context_variables** | `dict`  | 関数とエージェントの指示で利用可能な追加のコンテキスト変数の辞書。                                                                                     | `{}`           |
| **max_turns**         | `int`   | 許可される会話のターンの最大数。                                                                                                                       | `float("inf")` |
| **model_override**    | `str`   | エージェントが使用するモデルをオーバーライドするためのオプションの文字列。                                                                             | `None`         |
| **execute_tools**     | `bool`  | `False`の場合、エージェントが関数を呼び出そうとしたときに実行を中断し、即座に`tool_calls`メッセージを返します。                                       | `True`         |
| **stream**            | `bool`  | `True`の場合、ストリーミングレスポンスを有効にします。                                                                                                | `False`        |
| **debug**             | `bool`  | `True`の場合、デバッグログを有効にします。                                                                                                            | `False`        |

`client.run()`が終了すると（潜在的に複数のエージェントとツールの呼び出しの後）、関連するすべての更新された状態を含む`Response`を返します。具体的には、新しい`messages`、最後に呼び出された`Agent`、最新の`context_variables`が含まれます。これらの値（および新しいユーザーメッセージ）を次の`client.run()`実行に渡すことで、中断したところから対話を継続できます - `chat.completions.create()`と同様に。（`run_demo_loop`関数は、`/swarm/repl/repl.py`で完全な実行ループの例を実装しています。）

#### `Response` フィールド

| フィールド            | 型      | 説明                                                                                                                                                                                                                                     |
| --------------------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **messages**          | `List`  | 会話中に生成されたメッセージオブジェクトのリスト。[Chat Completions `messages`](https://platform.openai.com/docs/api-reference/chat/create#chat-create-messages)と非常に似ていますが、メッセージの発信元の`Agent`を示す`sender`フィールドがあります。 |
| **agent**             | `Agent` | メッセージを最後に処理したエージェント。                                                                                                                                                                                                 |
| **context_variables** | `dict`  | 入力変数と同じですが、変更がある場合はそれも含みます。                                                                                                                                                                                   |

## エージェント

`Agent`は単に一連の`instructions`と一連の`functions`（および以下の追加設定）をカプセル化し、実行を別の`Agent`にハンドオフする能力を持っています。

`Agent`を「Xを行う誰か」として擬人化したくなりますが、`instructions`と`functions`によって定義される非常に特定のワークフローまたはステップ（例：一連のステップ、複雑な検索、単一のデータ変換ステップなど）を表すこともできます。これにより、`Agent`を「エージェント」、「ワークフロー」、「タスク」のネットワークに構成することができ、すべて同じプリミティブで表現されます。

## `Agent` フィールド

| フィールド        | 型                       | 説明                                                               | デフォルト                    |
| ----------------- | ------------------------ | ------------------------------------------------------------------ | ----------------------------- |
| **name**          | `str`                    | エージェントの名前。                                               | `"Agent"`                     |
| **model**         | `str`                    | エージェントが使用するモデル。                                     | `"gpt-4o"`                    |
| **instructions**  | `str` または `func() -> str` | エージェントの指示。文字列または文字列を返す呼び出し可能オブジェクト。 | `"You are a helpful agent."` |
| **functions**     | `List`                   | エージェントが呼び出せる関数のリスト。                             | `[]`                          |
| **tool_choice**   | `str`                    | エージェントのツール選択（ある場合）。                             | `None`                        |

### 指示

`Agent`の`instructions`は直接会話の`system`プロンプトに変換されます（最初のメッセージとして）。アクティブな`Agent`の`instructions`のみが常に存在します（例：`Agent`のハンドオフがある場合、`system`プロンプトは変更されますが、チャット履歴は変更されません）。

```python
agent = Agent(
   instructions="あなたは役立つエージェントです。"
)
```

`instructions`は通常の`str`、または`str`を返す関数のいずれかです。関数はオプションで`context_variables`パラメータを受け取ることができ、これは`client.run()`に渡された`context_variables`によって埋められます。

```python
def instructions(context_variables):
   user_name = context_variables["user_name"]
   return f"{user_name}さんの要望を何でも手伝ってください。"

agent = Agent(
   instructions=instructions
)
response = client.run(
   agent=agent,
   messages=[{"role":"user", "content": "こんにちは！"}],
   context_variables={"user_name":"太郎"}
)
print(response.messages[-1]["content"])
```

```
こんにちは、太郎さん！今日はどのようなお手伝いができますか？
```

## 関数

- Swarm `Agent`はPython関数を直接呼び出すことができます。
- 関数は通常`str`を返すべきです（値は`str`としてキャストされようとします）。
- 関数が`Agent`を返す場合、実行はその`Agent`に転送されます。
- 関数が`context_variables`パラメータを定義している場合、`client.run()`に渡された`context_variables`によって埋められます。

```python
def greet(context_variables, language):
   user_name = context_variables["user_name"]
   greeting = "こんにちは" if language.lower() == "japanese" else "Hello"
   print(f"{greeting}, {user_name}さん!")
   return "完了"

agent = Agent(
   functions=[greet]
)

client.run(
   agent=agent,
   messages=[{"role": "user", "content": "greet()を日本語で使ってください。"}],
   context_variables={"user_name": "太郎"}
)
```

```
こんにちは、太郎さん!
```

- `Agent`の関数呼び出しにエラーがある場合（関数の欠落、引数の誤り、エラー）、エラーレスポンスがチャットに追加され、`Agent`が適切に回復できるようにします。
- `Agent`によって複数の関数が呼び出された場合、それらはその順序で実行されます。

### ハンドオフとコンテキスト変数の更新

`Agent`は`function`で別の`Agent`を返すことで、ハンドオフを行うことができます。

```python
sales_agent = Agent(name="営業担当エージェント")

def transfer_to_sales():
   return sales_agent

agent = Agent(functions=[transfer_to_sales])

response = client.run(agent, [{"role":"user", "content":"営業部門に転送してください。"}])
print(response.agent.name)
```

```
営業担当エージェント
```

また、より完全な`Result`オブジェクトを返すことで、`context_variables`を更新することもできます。これには`value`と`agent`も含めることができ、単一の関数で値を返し、エージェントを更新し、コンテキスト変数を更新する（またはその3つの任意の組み合わせ）ことができます。

```python
sales_agent = Agent(name="営業担当エージェント")

def talk_to_sales():
   print("こんにちは、世界！")
   return Result(
       value="完了",
       agent=sales_agent,
       context_variables={"department": "sales"}
   )

agent = Agent(functions=[talk_to_sales])

response = client.run(
   agent=agent,
   messages=[{"role": "user", "content": "営業部門に転送してください"}],
   context_variables={"user_name": "太郎"}
)
print(response.agent.name)
print(response.context_variables)
```

```
営業担当エージェント
{'department': 'sales', 'user_name': '太郎'}
```

> [!NOTE]
> `Agent`が複数の関数を呼び出して`Agent`にハンドオフする場合、最後のハンドオフ関数のみが使用されます。

### 関数スキーマ

Swarmは自動的に関数をChat Completions `tools`に渡されるJSONスキーマに変換します。

- ドキュメント文字列は関数の`description`になります。
- デフォルト値のないパラメータは`required`に設定されます。
- 型ヒントはパラメータの`type`にマッピングされます（デフォルトは`string`です）。
- パラメータごとの説明は明示的にサポートされていませんが、ドキュメント文字列に追加すれば同様に機能するはずです。（将来的にはドキュメント文字列の引数解析が追加される可能性があります。）

```python
def greet(name, age: int, location: str = "東京"):
   """ユーザーに挨拶します。呼び出す前に必ず名前と年齢を取得してください。

   Args:
      name: ユーザーの名前。
      age: ユーザーの年齢。
      location: 最高の場所。
   """
   print(f"こんにちは{name}さん、{location}で{age}歳とは素晴らしいですね!")
```

```javascript
{
   "type": "function",
   "function": {
      "name": "greet",
      "description": "ユーザーに挨拶します。呼び出す前に必ず名前と年齢を取得してください。\n\nArgs:\n   name: ユーザーの名前。\n   age: ユーザーの年齢。\n   location: 最高の場所。",
      "parameters": {
         "type": "object",
         "properties": {
            "name": {"type": "string"},
            "age": {"type": "integer"},
            "location": {"type": "string"}
         },
         "required": ["name", "age"]
      }
   }
}
```

## ストリーミング

```python
stream = client.run(agent, messages, stream=True)
for chunk in stream:
   print(chunk)
```

[Chat Completions APIのストリーミング](https://platform.openai.com/docs/api-reference/streaming)と同じイベントを使用します。例として`/swarm/repl/repl.py`の`process_and_print_streaming_response`をご覧ください。

2つの新しいイベントタイプが追加されています：

- `{"delim":"start"}`と`{"delim":"start"}`は、`Agent`が単一のメッセージ（レスポンスまたは関数呼び出し）を処理するたびに信号を送ります。これは`Agent`間の切り替えを識別するのに役立ちます。
- `{"response": Response}`は、便宜上、ストリームの最後に集計された（完全な）レスポンスを含む`Response`オブジェクトを返します。

# 評価

評価は任意のプロジェクトにとって重要であり、開発者が自身のswarmのパフォーマンスをテストするための独自の評価スイートを持ってくることを推奨します。参考として、`airline`、`weather_agent`、`triage_agent`のクイックスタート例にswarmを評価する方法のいくつかの例があります。詳細はそれぞれのREADMEをご覧ください。

# ユーティリティ

`run_demo_loop`を使用してswarmをテストしてください！これはコマンドラインでREPLを実行します。ストリーミングをサポートしています。

```python
from swarm.repl import run_demo_loop
...
run_demo_loop(agent, stream=True)
```

# 主要な貢献者

- Ilan Bigio - [ibigio](https://github.com/ibigio)
- James Hills - [jhills20](https://github.com/jhills20)
- Shyamal Anadkat - [shyamal-anadkat](https://github.com/shyamal-anadkat)
- Charu Jaiswal - [charuj](https://github.com/charuj)
- Colin Jarvis - [colin-openai](https://github.com/colin-openai)
</antArtifact>


