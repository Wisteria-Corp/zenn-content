---
title: temperature 0 なのに出力が変わる — MoE/dense 3モデルで KV キャッシュ状態の決定性を実測する
emoji: 🎲
type: tech
topics:
- llm
- ollama
- localllm
- nvidia
- cuda
published: true
---
NVIDIA GB10 搭載機で Ollama を回し、temperature 0 の出力が実行ごとに変わる件を測りました。以前の運用記事で「おまけの実験」として短く触れた内容の完全版です。この記事では手順・生ハッシュ・状態遷移・再現スクリプトを全部出します。

結論を先に書くと、**真のランダムな揺れは観測されず、出力は「モデル・入力・KV キャッシュ状態」の決定的関数に見えました。** 同じ状態なら bit 単位で再現し、状態が違うと確定的に別の出力になります。MoE だけの現象でもありませんでした。

環境は次のとおりです。

- Lenovo ThinkStation PGX（GB10 / 128GB unified / DGX OS・ARM64）
- Ollama native `/api/chat`
- gpt-oss:120b（MoE）、qwen3:30b-a3b（MoE）、gemma4:31b（dense・対照）

## この記事の結論

| 問い | 観測 |
|---|---|
| 同一条件で出力は毎回違うか | 違わない。同一状態なら全 run でハッシュ一致 |
| MoE だけ揺れるか | 揺れない。dense の gemma4 も初回だけ別変種 |
| 揺れの正体は | 「直前に同じプロンプトを処理したか」で決まる 2 変種 |
| 変種は再現するか | する。cold は cold と、warm は warm と一致 |
| 別プロンプトを挟むと | 直後の 1 回は cold 変種に戻り、次は warm 変種 |
| 実務上の対処 | A/B 比較の前に捨て回を 1 回流して状態を揃える |

## なぜ測ったのか

llama.cpp の開発で、MoE モデルが貪欲デコード（temperature 0）でも実行間で出力が揺れるという報告があります。確認できた範囲では OpenCL バックエンドでの実測で、同一入力 5 回で 2〜3 通りに割れたとのことです（出典の詳細は要検証）。

手元の常用モデルは全部 MoE です。CUDA ではどうなのか、深夜に測ってみました。

## 実験条件

| 項目 | 値 |
|---|---|
| API | Ollama native `/api/chat`（stream=false） |
| temperature | 0（明示） |
| seed | 42 |
| num_predict | 512 |
| num_ctx | 8192 |
| 実行方式 | 逐次（並行なし） |
| 判定 | content と thinking の SHA-256 先頭 12 桁を別々に比較 |

プロンプト（以下「プロンプト A」）は日本語 1 本だけです。

> 自宅サーバーで大規模言語モデルを運用する際の監視設計について、重要な観点を10項目、番号付きで列挙してください。各項目は「観点名: 一文の説明」の形式で書いてください。前置きと後書きは不要です。

## Phase 1: 素朴に 8 回投げる

3 モデルに同じプロンプトを 8 回ずつ、約 17 分かけて逐次実行しました。生のハッシュは次のとおりです。

### gpt-oss:120b（MoE）

| run | content | thinking | eval | 秒 |
|---:|---|---|---:|---:|
| 1 | 09af219d121b | cd3d7d8d4f12 | 510 | 29.1 |
| 2 | bb12e5ffd4be | cd3d7d8d4f12 | 496 | 11.5 |
| 3 | bb12e5ffd4be | cd3d7d8d4f12 | 496 | 11.5 |
| 4 | bb12e5ffd4be | cd3d7d8d4f12 | 496 | 11.6 |
| 5 | bb12e5ffd4be | cd3d7d8d4f12 | 496 | 11.6 |
| 6 | bb12e5ffd4be | cd3d7d8d4f12 | 496 | 11.5 |
| 7 | bb12e5ffd4be | cd3d7d8d4f12 | 496 | 11.6 |
| 8 | bb12e5ffd4be | cd3d7d8d4f12 | 496 | 11.6 |

初回だけ別の出力で、2〜8 回目は 7 回連続で完全一致です。thinking は 8 回とも同一のまま、content だけが割れています。分岐位置は content の 44 文字目でした。

| 変種 | 44 文字目付近 |
|---|---|
| run1 | 取得し、閾値超過時にアラートを発生させる |
| run2〜8 | 測定し、閾値超過時にアラートを発する |

ほぼ同確率の 2 トークンが、わずかな数値差で入れ替わった形です。そこから先の文章が丸ごと変わり、生成トークン数も 510 と 496 に分かれます。

### qwen3:30b-a3b（MoE）と gemma4:31b（dense）

| モデル | thinking の異なり数 | 別変種だった run | 秒の範囲 |
|---|---:|---|---|
| qwen3:30b-a3b | 2 | run1 と run3 | 6.2〜39.1 |
| gemma4:31b | 2 | run1 のみ | 95.7〜109.2 |

qwen3 は thinking のハッシュが 4a9ccc11f686（run1, run3）と adcf794c7c1e（残り 6 回）に割れました。run3 は 39.1 秒と飛び抜けて遅い回です。gemma4 は run1 だけ b83537cf57f8、残りは 2794662df3c5 で揃っています。

「MoE だから揺れる」説なら dense は揺れないはずですが、揺れました。しかも 3 モデルとも「ロード直後の初回が違う」という同じ形です。乱数というより状態の顔をしています。

### 自分で踏んだ罠: 空文字列の SHA-256

qwen3 と gemma4 は content が全 run で `e3b0c44298fc`（空文字列の SHA-256 先頭）になっていました。一瞬「完全一致！」と喜びかけたのですが、実際は num_predict:512 を thinking が使い切って `done_reason: length`、content は空だったというオチです。推論型モデルに小さい上限を渡すと「空の正常終了」になる、まさにそれです。

この 2 モデルの判定は thinking 側のハッシュで行っています。content での比較ではない点は割り引いてください。

## 仮説: 状態を入れれば決定論

Phase 1 の形から次の仮説を立てました。

> 出力は（モデル・入力・KV キャッシュ状態）の決定的関数である。同じ状態なら再現し、状態が違えば別の出力になる。

検証は、状態を操作して測り直せば済みます。操作できるのは 2 つだけです。モデルの unload（状態を初期化）と、別プロンプトの投入（キャッシュを撹乱）です。

## Phase 2: 状態制御プロトコル

gpt-oss:120b で次の手順を実行しました。unload は `keep_alive: 0` を投げて 3 秒待っています。

```
unload → cold-A → warm-A ×2
unload → cold-A → warm-A ×2
B → A ×2
```

B は別プロンプト（「1から20までの素数を列挙してください。」）です。生ハッシュは次のとおりです。

| 順 | label | content | eval | 秒 |
|---:|---|---|---:|---:|
| 1 | cold-A-1 | 09af219d121b | 510 | 30.7 |
| 2 | warm-A-1 | bb12e5ffd4be | 496 | 11.6 |
| 3 | warm-A-2 | bb12e5ffd4be | 496 | 11.6 |
| 4 | cold-A-2 | 09af219d121b | 510 | 29.2 |
| 5 | warm-A-3 | bb12e5ffd4be | 496 | 11.6 |
| 6 | warm-A-4 | bb12e5ffd4be | 496 | 11.6 |
| 7 | B | bb887450af44 | 156 | 4.1 |
| 8 | afterB-A-1 | 09af219d121b | 510 | 12.1 |
| 9 | afterB-A-2 | bb12e5ffd4be | 496 | 11.6 |

遷移を図にするとこうなります。

```
unload ─▶ A:cold変種(09af…)  ─▶ A:warm変種(bb12…) ─▶ A:warm変種(bb12…)
unload ─▶ A:cold変種(09af…)  ─▶ A:warm変種(bb12…) ─▶ A:warm変種(bb12…)
          B(bb88…)           ─▶ A:cold変種(09af…) ─▶ A:warm変種(bb12…)
```

### 読み取れること

| 検証 | 結果 |
|---|---|
| cold と cold は一致するか | 一致。09af219d121b が 2 回とも（eval=510） |
| warm と warm は一致するか | 一致。bb12e5ffd4be が 4 回とも（eval=496） |
| cold と warm は一致するか | 不一致。確定的に 2 変種 |
| B を挟んだ直後の A | cold 変種に戻る。その次の A は warm 変種 |

Phase 1 の gpt-oss の 2 変種とも、ハッシュは完全に同じです（09af… と bb12…）。Phase 1 と Phase 2 は別々のスクリプト実行ですが、同じ値が再び出ているので、変種は「毎回生まれる偶然」ではなく、状態から決まる固定の出力だと言えそうです。

### afterB-A-1 が 12.1 秒だった意味

見逃しやすいのがここです。B のあとの A は cold 変種ですが、所要は 12.1 秒でした。unload 後の cold は 30 秒前後かかっているので、モデルの再ロードが走ったわけではありません。

つまり cold 変種の条件は「モデルがロードされ直した」ではなく、「直前のキャッシュに A の接頭辞が無い」だと読むのが自然です。unload が効く理由も、モデルを降ろすとキャッシュも消えるからと考えると辻褄が合います。cold 変種のほうが 14 トークン長いぶん、わずかに所要が伸びているのも整合的です（12.1 秒と 11.6 秒）。

### 2 変種の中身の差

文章としても別物です。warm 変種との違いを一部挙げます。

| 項目 | cold 変種 | warm 変種 |
|---|---|---|
| 1 | …を取得し、閾値超過時にアラートを発生させる | …を測定し、閾値超過時にアラートを発する |
| 2 | …応答時間を測定し、…パフォーマンス低下を検知する | …応答時間を計測し、…原因解析を促す |
| 5 | ネットワーク帯域・接続監視 | ストレージ容量監視 |
| 10 | 可観測性・メトリクス統合 | 可用性・フェイルオーバー監視 |

項目の並びや選ぶ観点そのものが変わっているので、「誤差レベルの表記ゆれ」ではありません。1 回目の評価だけを見て設定の良し悪しを判断すると、これを拾ってしまいます。

## 機構の解釈（状況証拠）

私の解釈はこうです。同じプロンプトを直前に処理していれば、プロンプト評価でキャッシュ再利用の経路を通り、処理していなければフルの prefill を行う。経路が違えば浮動小数点の丸めがわずかに変わり、logits が変わり、近接確率のトークンが入れ替わり、そこから先がカスケードする。

ただし、これは**状況証拠**です。次の点は確認できていません。

- `prompt_eval_count` は cold でも warm でも A は 139 のままで、キャッシュ再利用がカウンタに出ない（Ollama のバージョン依存かどうかは要検証）
- llama.cpp 内部の経路は追跡していない

Phase 1 の qwen3 run3（39.1 秒・cold 変種側）は、定時で走る別ジョブがキャッシュを撹乱した時間帯と重なっていたと考えると説明がつきますが、ジョブ側のリクエストログまでは突き合わせていません（要検証）。

## 実務ルール

| 場面 | やること |
|---|---|
| モデル間・設定間の単発 A/B | 捨て回を 1 回流して warm に揃えてから測る |
| 再現性が要る検証 | 比較前に状態（cold/warm）を記録する |
| 結果の突合 | 本文の目視ではなく SHA-256 で比べる |
| 別ジョブが同居する環境 | 測定中は外来リクエストを止めるか、時刻をログに残す |

揃えないと「設定の差」ではなく「キャッシュ状態の差」を測ることになります。私自身、過去の reasoning_effort 比較がどちらの状態で走ったのか記録しておらず、あとから頭を抱えました。

## 再現スクリプト

標準ライブラリだけの最小版です。接続先は自分の環境に置き換えてください。

```python
import hashlib, json, time, urllib.request

BASE = "http://<host>:11434"
MODEL = "gpt-oss:120b"
OPTIONS = {"temperature": 0, "seed": 42, "num_predict": 512, "num_ctx": 8192}
PROMPT_A = ("自宅サーバーで大規模言語モデルを運用する際の監視設計について、"
            "重要な観点を10項目、番号付きで列挙してください。"
            "各項目は「観点名: 一文の説明」の形式で書いてください。前置きと後書きは不要です。")
PROMPT_B = "1から20までの素数を列挙してください。"

def post(path, payload, timeout=600):
    req = urllib.request.Request(BASE + path, data=json.dumps(payload).encode("utf-8"),
                                 headers={"Content-Type": "application/json"})
    with urllib.request.urlopen(req, timeout=timeout) as r:
        return json.loads(r.read().decode("utf-8"))

def unload():
    post("/api/generate", {"model": MODEL, "keep_alive": 0}, timeout=120)
    time.sleep(3)

def sha(s):
    return hashlib.sha256(s.encode("utf-8")).hexdigest()[:12]

def chat(label, prompt):
    t0 = time.time()
    msg = post("/api/chat", {"model": MODEL, "stream": False, "options": OPTIONS,
                             "messages": [{"role": "user", "content": prompt}]})["message"]
    print(f"{label}: content={sha(msg.get('content',''))} "
          f"think={sha(msg.get('thinking',''))} {time.time()-t0:.1f}s")

unload(); chat("cold-A-1", PROMPT_A); chat("warm-A-1", PROMPT_A); chat("warm-A-2", PROMPT_A)
unload(); chat("cold-A-2", PROMPT_A); chat("warm-A-3", PROMPT_A); chat("warm-A-4", PROMPT_A)
chat("B", PROMPT_B); chat("afterB-A-1", PROMPT_A); chat("afterB-A-2", PROMPT_A)
```

注意点が 3 つあります。

- `num_predict` を小さくしすぎると、推論型モデルでは content が空になります。空文字列の SHA が全 run で出たら、まずこれを疑ってください
- thinking も比較対象に含めてください。content が空でも thinking で状態差が見えます
- ハッシュは環境ごとに違う値になります。この記事のハッシュとの一致を期待せず、「cold 同士・warm 同士が揃うか」「B の直後に cold 変種へ戻るか」という構造のほうを確認してください

## 限界

- 1 プロンプト・1 ハード・Ollama 1 バージョンでの結果です。プロンプトを変えれば別の結果になる可能性は排除していません
- 逐次実行のみです。並行リクエスト（continuous batching）には別の揺れ要因が知られていて、この記事の範囲外です
- dense の対照は gemma4:31b の 1 モデルだけで、しかも thinking 側での比較です。「MoE 固有ではない」という主張はこの範囲に限られます
- 件数は、スクリプトから数えると Phase 1 が 24 リクエスト、Phase 2 が 10 リクエストの計 34 です。手元の実験ノートには別の合計値の記載があり、突き合わせられていません（要検証）。いずれにせよ、観測した全リクエストで「同一状態でのハッシュ不一致」は 0 でした
- 機構（prefill 経路の差）は状況証拠です

## まとめ

- temperature 0 の出力は、**モデルと入力だけでなく KV キャッシュ状態にも依存する**ように見える
- 同一状態なら bit 単位で再現し、状態を操作すると変種の遷移まで再現できた
- MoE 固有の現象ではなく、dense の対照でも初回だけ別変種になった
- A/B 比較の前に捨て回を 1 回流すだけで、状態差を測るリスクは大きく減らせる

「毎回違う」を乱数のせいにする前に、直前に何を処理していたかを疑ってください。ハッシュを取って並べるだけで、状態の影響は見えます。

---
*実測環境: Lenovo ThinkStation PGX (NVIDIA GB10, 128GB) / DGX OS / Ollama native API。実測日は 2026-09-07。バージョンやプロンプトで変わりうるので、追試するときはご自身の環境の値を信じてください。*
