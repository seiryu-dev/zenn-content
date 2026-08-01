---
title: "OpenTelemetryで会話本文を止めても、IPアドレスとAPIキー識別情報は残る —— LLMゲートウェイのspan属性を55個数えた"
emoji: "🕵️"
type: "tech"
topics: ["opentelemetry", "observability", "llmops", "litellm", "python"]
published: true
---

LiteLLM に `turn_off_message_logging: true` を入れれば、会話本文はトレースに出なくなります。
**ですが、IPアドレス・APIキーの識別情報・User-Agent・ユーザーIDは実値のまま残ります**。
私はこれを、設定を入れて「これで安全になった」と思ったあとに見つけました。

トレースは収集基盤へ送るために作るものです。送り先が外部のSaaSなら、**「どこから・どのキーで叩いたか」が社外のサーバーへ渡ります**。自社基盤に送る場合でも、トレースの閲覧権限をコードより広く開けている環境では、同じ情報が同じ範囲に見えます。

実測したところ、1リクエストで作られる主spanの属性は **55個**。会話本文はそのうち **2個**だけで、`metadata.*` が **29個（53%）** でした。`turn_off_message_logging` を入れると 55→51 になりますが、**消えた4個の内訳は会話本文2個（`gen_ai.input.messages` / `gen_ai.output.messages`）と、本文ではない2個（`gen_ai.operation.name` / `gen_ai.response.finish_reasons`）です**。`metadata.*` の29個は丸ごと残ります。

`turn_off_message_logging` が本文だけを止めることは、公式ドキュメントにも書かれています。**この記事で測ったのは、その先です**。「では何が残るのか」を55属性ぜんぶ数え、SDK直呼びと proxy 経由で値の入り方が違うことを確かめ、属性の境界を allow list で引いて15属性まで落としました。

まず数え方を置き、そのあとに実装を書きます。

**LiteLLM を例にしていますが、「計装したら何が span に載るか数える」という手順自体は、OTel SDK の `SpanExporter` を通る計装であれば同じように使えます**。

## まず、自分の環境で数える

実APIを呼ばずに、いま数えられます。

```python
# inventory.py
import time
import litellm
from litellm.integrations.opentelemetry import OpenTelemetry, OpenTelemetryConfig
from opentelemetry.sdk.trace.export import SpanExporter, SpanExportResult


class InventoryExporter(SpanExporter):
    """span名・属性数・全キーを出すだけ"""

    def export(self, spans):
        for sp in spans:
            attrs = dict(sp.attributes or {})
            print(f"span={sp.name}  attrs={len(attrs)}")
            for k in sorted(attrs):
                print("   ", k)
        return SpanExportResult.SUCCESS

    def shutdown(self):
        return None

    def force_flush(self, timeout_millis: int = 30000):
        return True


litellm.callbacks = [
    OpenTelemetry(config=OpenTelemetryConfig(exporter=InventoryExporter()))
]

litellm.completion(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "hello"}],
    mock_response="ok",              # 実APIを呼ばない
    api_key="sk-dummy-never-used",
)

time.sleep(3)   # BatchSpanProcessor の flush 待ち
```

私の環境（litellm 1.90.2）での出力です。**同系統のキーは読みやすさのため圧縮しています**（実際は1キー1行で出ます）。

```
span=litellm_request  attrs=55
    gen_ai.cost.discount_amount
    ...（コスト系10個）
    gen_ai.input.messages
    gen_ai.output.messages
    gen_ai.operation.name
    gen_ai.request.model
    gen_ai.response.finish_reasons
    gen_ai.response.id
    gen_ai.response.model
    gen_ai.system
    gen_ai.usage.input_tokens / output_tokens / total_tokens
    hidden_params
    litellm.call_id
    litellm.provider.model
    llm.is_streaming
    llm.request.type
    metadata.requester_ip_address
    metadata.user_agent
    metadata.user_api_key_hash
    metadata.user_api_key_user_id
    metadata.user_api_key_user_email
    metadata.user_api_key_team_id / team_alias / org_id / org_alias
    metadata.user_api_key_project_id / project_alias / alias
    metadata.user_api_key_spend / max_budget / budget_reset_at
    metadata.user_api_key_end_user_id / auth_metadata / request_route
    metadata.team_id / team_alias
    metadata.requester_metadata / requester_custom_headers
    metadata.spend_logs_metadata / usage_object / applied_guardrails
    metadata.mcp_tool_call_metadata / prompt_management_metadata
    metadata.vector_store_request_metadata / cold_storage_object_key
span=raw_gen_ai_request  attrs=1
    llm.openai.stringified_raw_response
```

**`metadata.user_api_key_*` だけで16個あります**。認証まわりの情報が、キー単位で属性に展開されています。

:::message alert
**proxy 経由で数える場合は注意があります**。LiteLLM 1.90.2 では、親span（`Received Proxy Server Request`）が存在すると、既定では `litellm_request` span が作られません。

```python
# litellm/integrations/opentelemetry.py
should_create_primary_span = parent_span is None or get_secret_bool(
    "USE_OTEL_LITELLM_REQUEST_SPAN"
)
```

`else` 側では `self.set_attributes(parent_span, ...)` が呼ばれるので、**属性が消えるわけではなく、親spanに載ります**。span名で探していると「何も出ていない」と誤読します。

**この記事の55属性はSDK直呼びで数えた値です**。proxy で同じ名前のspanを見たい場合は `USE_OTEL_LITELLM_REQUEST_SPAN=true` を設定してください。
:::

数えたあとに見るべき観点は5つです。LiteLLM に限らず、自動計装を入れたライブラリ全般で使えます。

1. **属性の数を知る**。まず `len(span.attributes)` を見る
2. **spanが何個作られるか数える**。1リクエスト＝1span とは限りません。私の環境では `litellm_request` と `raw_gen_ai_request` の2つが作られ、**プロバイダの生レスポンスは後者に載っていました**
3. 各属性を「**観測に必要 / ついてきただけ**」で仕分ける。この記事の例では、最終的に残したのは55個中15個でした
4. **停止スイッチの名前が約束している範囲を確認する**。私の環境では `turn_off_message_logging` で4属性が消えましたが、そのうち会話本文は2つでした
5. **「残すもの」を列挙する**。「消すもの」を列挙すると、ライブラリ更新で増えた属性が既定で漏れます

LiteLLM proxy を立てて実値まで確認する手順は、記事末尾の「再現手順」にあります。

## 環境


| | |
|---|---|
| litellm | 1.90.2 |
| OpenTelemetry | `opentelemetry-api` / `opentelemetry-sdk` ともに 1.44.0 |
| OTel実装 | **v1**（`litellm/integrations/opentelemetry.py`） |
| exporter | `console`（既定値。**受け先サーバは不要**） |
| OS | Windows + Git Bash |

:::message
litellm には OTel の実装が **v1 と v2 の2系統**あります。v2 は `litellm/integrations/otel/logger.py` の `OpenTelemetryV2` で、環境変数 `LITELLM_OTEL_V2` で有効化します（`otel/model/config.py` で `default=False`）。**この記事はすべて v1（既定）の挙動です**。
:::

計装は callback を1つ足すだけです。

```yaml
litellm_settings:
  callbacks: ["otel"]
```

exporter を指定しなければ既定で `console` になります（`opentelemetry.py` の `exporter: Union[str, SpanExporter] = "console"`）。**まず1トレース出すだけなら、Docker も収集基盤も要りません**。

## 1. 棚卸し：1リクエストで何がspanに載るか


冒頭で数えた55属性を、種類ごとに分けるとこうなります。**すべてSDK直呼びで測った値です**（`metadata.*` に実値が入るかは proxy 経由でないと分からないので、§2 で別に確認します）。

| span | 属性数 | 内訳 |
|---|---|---|
| `litellm_request` | **55** | **`metadata.*` 29**（認証・呼び出し元・周辺情報） / コスト 10 / モデル・リクエスト 6 / トークン 3 / **会話本文 2** / その他 5 |
| `raw_gen_ai_request` | 1 | **プロバイダの生レスポンス 1** |

**`metadata.*` が29個で、全体の53%を占めています**。会話本文は2個（`gen_ai.input.messages` と `gen_ai.output.messages`）だけです。

値はこう入っていました。

```json
// litellm_request span
"gen_ai.input.messages":  "[{\"role\": \"user\", \"parts\": [{\"content\": \"...\"}]}]",
"gen_ai.output.messages": "[{\"role\": \"assistant\", \"parts\": [{\"content\": \"...\"}]}]",
"gen_ai.usage.total_tokens": 30,
"gen_ai.request.model": "gpt-4o-mini"

// raw_gen_ai_request span（プロバイダの生レスポンス用に別途作られる）
"llm.openai.stringified_raw_response": "..."
```

トークン数もモデル名もコストも取れます。ここは狙いどおりでした。**問題は、狙っていないものまで一緒に載ることです**。

そして次節のとおり、**この29個は kill-switch を入れても減りません**。

## 2. 本文を止めても、53%は残る


litellm には kill-switch があります。

```yaml
litellm_settings:
  turn_off_message_logging: true
```

実装上も最優先で効きます（`_resolve_capture_mode()` が真っ先にこれを見て `NO_CONTENT` を返す）。本文は消えました。`raw_gen_ai_request` spanごと出なくなります。

**それでも `litellm_request` は55→51属性にしか減りません**。残った中身がこれです。

```json
"metadata.requester_ip_address":  "127.0.0.1",
"metadata.user_agent":            "curl/8.17.0",
"metadata.user_api_key_hash":     "litellm_proxy_master_key",
"metadata.user_api_key_user_id":  "default_user_id"
```

**空文字ではありません。実値です**。`metadata.*` の29属性（認証・呼び出し元・周辺情報）は、丸ごと残ります。

:::message
`user_api_key_hash` の中身は認証方法で変わります。**master_key なら `"litellm_proxy_master_key"` という固定エイリアス**（上の実測値がこれ）、**virtual key ならそのキーのハッシュ**が入ります。鍵そのものが平文で出るわけではありません。

問題は、**どのキーで・どのIPから・どのクライアントで叩いたかが紐づく**ことです。**virtual key 運用ならキー単位で追跡できてしまう**ぶん、影響はむしろ大きくなります。
:::

:::message alert
**測り方に落とし穴があります。SDKを直接呼んだ検証では、これらの値は空文字になります**。`metadata` はspanに無条件でセットされますが、値が None のとき `""` に変換されるためです。**今回の測定で認証由来の実値が自動的に入ったのは、proxy 経由のときでした**（SDK直呼びでも `metadata` を明示的に渡せば値は入ります）。私は最初これで「キーは出るが中身は空」を見て、危うく「実害なし」と誤読しかけました。**本番と同じ経路（proxy + auth）で測ってください**。
:::

本文（What）は消える。しかし**誰が（Who）・どこから（Where）は残る**。「本文を止めたから安全」は、**55個のうち2個を止めただけ**でした。

## 3. 対策：allow list を書いて、proxy に入れる


### なぜ allow list なのか

**なぜ deny list ではなく allow list なのか**。最初は「消したいものを並べる」deny list で書いて、**1個漏らしました**。`llm.openai.stringified_raw_response` を列挙し忘れて、それだけがそのまま出力に残ったのです。**列挙漏れが、そのまま漏洩になります**。allow list なら、知らない属性が増えても既定で落ちます。

### 実装：exporter層で落とす


**ここが本命です**。`OpenTelemetryConfig.exporter` の型が `Union[str, SpanExporter]` なので、**自前の SpanExporter を差し込めます**。

```python
from opentelemetry.sdk.trace.export import ConsoleSpanExporter, SpanExporter

ALLOW = ("gen_ai.usage.", "gen_ai.response.model", "gen_ai.request.model",
         "gen_ai.cost.", "gen_ai.operation.name", "service.")

class RedactingExporter(SpanExporter):
    """内側のexporterに渡す前に、許可リスト以外の属性を落とす"""
    def __init__(self, inner: SpanExporter, allow=ALLOW):
        self.inner, self.allow = inner, allow

    def export(self, spans):
        for sp in spans:
            sp._attributes = {
                k: v for k, v in (sp.attributes or {}).items()
                if any(k.startswith(p) for p in self.allow)
            }
        return self.inner.export(spans)

    def shutdown(self):
        return self.inner.shutdown()

    def force_flush(self, timeout_millis: int = 30000):
        return self.inner.force_flush(timeout_millis)
```

**これで55属性→16属性になります**（`raw_gen_ai_request` は1→0）。残るのはトークン・コスト・モデル名と、`gen_ai.operation.name` です。

`gen_ai.operation.name` は `ALLOW` に入れてあるので残ります。次節の `turn_off_message_logging` を併用すると、この属性も消えて15属性（トークン・コスト・モデル名だけ）になります。


:::message alert
`ReadableSpan.attributes` は読み取り専用プロパティなので、内部の辞書を直接差し替えています。**内部APIの書き換えです**。`_format_attributes` が「dict でなければ `dict()` に変換して返すだけ」なので素の dict で動きますが、**SDKのバージョンが変われば壊れる可能性があります**（本記事は `opentelemetry-sdk==1.44.0`）。

内部APIを触りたくない場合は、**OTel Collector 側の `attributes` プロセッサで落とす**方法もあります（こちらは未検証）。Collector を自分で管理しているならそちらが安全です。管理下にないなら、アプリを出る前に落とすことになります。
:::

### proxy に custom callback として入れる（実測済み）


上の `RedactingExporter` は、SDKに直接渡すコードです。**proxy の `callbacks: ["otel"]` という組み込み指定では、exporterのインスタンスを渡せません**。

ただし LiteLLM proxy は、config の文字列を `get_instance_fn()` で解決して**実インスタンスを読み込めます**（`proxy/proxy_server.py`）。つまり `module.instance` 形式で書けば通ります。

`otel_redact_callback.py`（上の `RedactingExporter` を定義したうえで）:

```python
from litellm.integrations.opentelemetry import OpenTelemetry, OpenTelemetryConfig

otel_redacted = OpenTelemetry(
    config=OpenTelemetryConfig(exporter=RedactingExporter(ConsoleSpanExporter()))
)
```

```yaml
litellm_settings:
  callbacks: otel_redact_callback.otel_redacted
```

**この構成で実際に proxy を起動し、リクエストを通しました**。結果：

| 属性 | 結果 |
|---|---|
| `metadata.requester_ip_address` | **落ちた** |
| `metadata.user_api_key_hash` | **落ちた** |
| `metadata.user_agent` | **落ちた** |
| `gen_ai.input.messages` | **落ちた** |
| `gen_ai.usage.total_tokens` | 維持 |
| `gen_ai.cost.total_cost` | 維持 |
| `gen_ai.response.model` | 維持 |

callback の読み込みエラーは0件でした。**SDK限定の話ではありません**。ただし実測したのは最小構成のproxyを別ポートで立てたものなので、**「proxy構成に投入できることは確認した」までです**。

:::message
OTel Collector 側（`attributes` プロセッサ等）で落とす方法もありますが、**こちらは未検証です**。アプリ側で落とすかCollectorで落とすかは、Collectorを自分が管理しているかで変わります。管理下にないなら、**アプリを出る前に落とすのが確実**だと考えました。
:::

### ライブラリ側のスイッチは併用する


`turn_off_message_logging: true`。**消えるのは4属性で、そのうち会話本文は2つです**（残る2つは `gen_ai.operation.name` と `gen_ai.response.finish_reasons`）。呼び出し元は残ります（§2）。

litellm v1 側で `metadata.*` をキー単位で落とす設定は見つけられませんでした。属性フィルタらしき実装（`_build_metric_attribute_filter` など）は**すべてメトリクス用**です。

:::message
`litellm_core_utils/redact_messages.py` に `redact_user_api_key_info()` という関数自体は存在します。ただし v1 の OTel span属性の経路には適用されていませんでした。
:::

## 4. 4構成の比較


同じ測り方（exporter内で `len(span.attributes)`）で比べた結果です。

| 構成 | `turn_off` | allow list | `litellm_request` | `raw_gen_ai_request` | 会話本文 | `metadata.*` |
|---|---|---|---|---|---|---|
| ① 素 | ✗ | ✗ | 55 | 1 | 露出 | 露出（29個） |
| ② `turn_off` のみ | ✓ | ✗ | 51 | **span消滅** | 消える | **29個残る** |
| ③ allow list のみ | ✗ | ✓ | **16** | 属性0 | 消える | 消える |
| **④ 併用** | ✓ | ✓ | **15** | span消滅 | 消える | 消える |

**検証環境で採用したのは ④ です**。③だけでも今回確認した項目（会話本文・生レスポンス・IP・User-Agent・キー識別情報・ユーザーID）は落ちますが、あえて二重にしました。**allow list に書き漏らしがあっても本文は litellm 側で止まり、litellm の実装が変わっても allow list が効く**——片方が破れてももう片方が残る形にしたかったからです。

観測に必要な属性（トークン・コスト・モデル名）は、どの構成でも維持されました。

## 測るときの落とし穴（3つとも踏みました）


**1プロセスで複数構成を回すと、先頭しか出力されません**（`BatchSpanProcessor` の非同期エクスポートと出力キャプチャの相性）。構成ごとにプロセスを分けてください。

**`ConsoleSpanExporter` は `__init__` 時点の `sys.stdout` を握ります**。`redirect_stdout` の外でインスタンス化すると出力が捕捉できず、私は「フィルタが効きすぎた」と誤判定しました。実際は正常でした。

### 日本語は二重エスケープされる


日本語の入力でトレースを出し、`grep` で名前を探しました。**出てきません**。一瞬「日本語は載らないのか」と思いましたが、違いました。

```
"gen_ai.input.messages": "[{\"role\": \"user\", ... \"content\": \"\\u60a3\\u8005\\u306e\\u6c0f\\u540d...\"}]"
```

バックスラッシュが**2つ**あります。`json.dumps` が2回かかっているからです。

1. litellm が `safe_dumps` で messages を文字列化。`json.dumps(..., default=str)` は **`ensure_ascii` 未指定＝既定 True** なので、**この時点で属性値の中身が `\u60a3\u8005...` という文字列になる**（日本語のままではない）
2. その文字列がspan属性の値になる
3. `ConsoleSpanExporter` が `span.to_json()` を呼ぶ。その中の `json.dumps(f_span, indent=indent)` も **既定 True** なので、**値に含まれるバックスラッシュがさらにエスケープされて `\\u60a3` になる**

Console出力に対して3通りで探した結果です。

| 検索した文字列 | 結果 |
|---|---|
| `山田太郎`（生の日本語） | ヒットせず |
| `\u5c71\u7530`（1段エスケープ＝属性値そのもの） | **ヒットせず** |
| `\\u5c71\\u7530`（バックスラッシュ2つ） | **ヒット** |

検証は Console 出力をファイルに落として Python の固定文字列検索（`in` 演算子）で行いました。`grep` を使う場合は `-F`（固定文字列）を付け、シェルのエスケープに注意してください。

**「見つからない」のではなく「探し方を知らないと見つからない」が正確です。「ログをgrepしたがPIIは無かった」は、そのままでは「安全である」を意味しません**。

:::message
この見え方は `ConsoleSpanExporter` での話です。OTLP exporter やバックエンドのUIで同じになるかは未検証です。
:::

## まだ確かめていないこと


- **v2実装**（`LITELLM_OTEL_V2=true`）での挙動
- **OTLP exporter や実バックエンドUI** での日本語の見え方
- **OTel Collector 側**（`attributes` プロセッサ）で落とす方法
- **virtual key + DB 構成**での `user_api_key_user_email` の実値
- **本番ゲートウェイへの適用**（この記事の実測はすべて別ポートの検証環境）

## 付録：この記事の再現手順


実測に使った最小構成です。**実APIは呼ばないので課金ゼロ**で試せます。

:::message
**実行環境は Windows + Git Bash** です。PowerShell で実行する場合、`curl` は `Invoke-WebRequest` のエイリアスなので **`curl.exe` を明示**してください（PowerShellでの動作は未検証です）。
:::

`otel_probe.yaml` — **コメントも含めてASCIIのみ**にしてください。日本語が入るとWindowsのcp932で読めず起動に失敗します（私はこれで1回落としました）。

```yaml
model_list:
  - model_name: test-model
    litellm_params:
      model: openai/gpt-4o-mini
      api_key: sk-dummy-never-used
      mock_response: "ok"

litellm_settings:
  callbacks: ["otel"]
  # turn_off_message_logging: true   # enable this to test the kill-switch

general_settings:
  master_key: sk-otel-probe-local
```

起動（**本番とポートを分けること**）:

```bash
unset LITELLM_OTEL_V2   # v1 で確認する（v2 は未検証）
PYTHONIOENCODING=utf-8 litellm --config otel_probe.yaml --port 4999 > proxy.log 2>&1 &
```

`PYTHONIOENCODING=utf-8` は、今回の Windows + cp932 環境では必要でした。起動バナーの罫線文字が cp932 で書けず落ちます。

リクエスト（**起動待ちを兼ねてリトライを付けます**）:

```bash
curl -s --retry 30 --retry-delay 3 --retry-connrefused \
  -X POST http://localhost:4999/v1/chat/completions \
  -H "Authorization: Bearer sk-otel-probe-local" \
  -H "Content-Type: application/json" \
  -d '{"model":"test-model","messages":[{"role":"user","content":"KANJA no shimei ha Yamada Taro desu"}]}'
```

§3の日本語エスケープを再現するには、`content` に**日本語をそのまま入れて**送ってください（上の例はシェルのエンコード差を避けてローマ字にしています）。

確認（`BatchSpanProcessor` の flush を待つため数秒おきます）:

```bash
sleep 8
grep -F "metadata." proxy.log            # 誰が・どこから
grep -F '\\u5c71\\u7530' proxy.log       # 2段エスケープされた日本語（山田）
```

**`mock_response` は config 側に置いてください**。リクエストボディに入れても効かず、実APIを叩きにいきました。

## おわりに


自動計装は便利です。callback を1行足すだけで、トークンもレイテンシもコストも見えるようになりました。

**同時に、主spanだけで55個（2span合計56個）の属性が外に出るようになりました**。最終的に残したのは15個で、残りは「ついてきた」ものです。`turn_off_message_logging` を入れると4属性が消えますが、`metadata.*` の29個はそのまま残ります。**名前が約束している範囲と、自分が守りたい範囲は別物でした**。

計装を入れたら、**まず何が載っているかを数えることをおすすめします**。属性名を全部ダンプして眺めるだけです。そのうえで「残すもの」を決めれば、あとから増える属性も既定で落ちます。

トレースは「システムの中で何が起きたか」を外に出す仕組みです。**何を出すかを決めるのは、ライブラリではなく自分です**。
