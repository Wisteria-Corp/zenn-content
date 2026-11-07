---
title: 日本語 STT 12 エンジン実測 — AmiVoice・Google・OpenAI・Deepgram・Azure・ElevenLabs・AssemblyAI・ReazonSpeech・SenseVoice・kotoba・Whisper
  を同じ音声で比べた（曜日・固有名詞・数値・単語登録）
emoji: 🗣️
type: tech
topics:
- amivoice
- deepgram
- whisper
- stt
- 音声認識
published: true
---
日本語の音声メモを文字にするとき、どのエンジンで何が壊れるのか。前回はローカルの Whisper 系だけで測りましたが、今回は同じ合成音声ベンチに **クラウド 7 社（AmiVoice・Google・OpenAI・Deepgram・Azure・ElevenLabs・AssemblyAI）とローカル非 Whisper 系 3 本（ReazonSpeech・SenseVoice・kotoba-whisper）** を足して、23 条件を横並びで比べました。先に結果です。

| エンジン（条件） | 場所 | CER | 曜日 | 固有名詞 | 数値 | 応答/文 |
|---|---|---|---|---|---|---|
| **AmiVoice 汎用＋単語登録 3 語（表記＋読み）** | クラウド | **0.0%** | 6/6 | **6/6** | 6/6 | 1.4 秒 |
| **Google Chirp 3＋適応フレーズ 3 語** | クラウド | **0.0%** | 6/6 | **6/6** | 6/6 | 3.1 秒 |
| Deepgram Nova-3＋keyterm 3 語 | クラウド | 0.8% | 6/6 | **6/6** | 6/6 | 1.1 秒 |
| OpenAI gpt-4o-transcribe＋prompt 3 語 | クラウド | 1.4% | 6/6 | **6/6** | 6/6 | 1.3 秒 |
| Whisper medium＋語彙注入 | ローカル | 2.3% | 6/6 | **6/6** | 6/6 | 8.4 秒 |
| Google Chirp 3 | クラウド | 2.3% | 6/6 | 4/6 | 6/6 | 3.4 秒 |
| OpenAI gpt-4o-transcribe | クラウド | 2.3% | 6/6 | 4/6 | 6/6 | 1.4 秒 |
| Azure Fast transcription＋フレーズリスト | クラウド | 2.5% | 6/6 | 5/6 | 6/6 | **0.5 秒** |
| ElevenLabs Scribe v1 | クラウド | 2.7% | 6/6 | 4/6 | 6/6 | 1.1 秒 |
| AmiVoice 汎用（登録なし） | クラウド | 3.2% | 6/6 | 3/6 | 6/6 | 1.1 秒 |
| Azure 短音声 REST | クラウド | 4.6% | 6/6 | 3/6 | 6/6 | 1.9 秒 |
| Whisper medium | ローカル | 5.0% | 6/6 | 4/6 | 6/6 | 8.4 秒 |
| ReazonSpeech（VAD 分割あり） | ローカル | 5.1% | 6/6 | 4/6 | 6/6 | **0.5 秒** |
| AssemblyAI Universal（±word_boost 同値） | クラウド | 5.9% | 6/6 | 4/6 | 6/6 | 3.7 秒 |
| kotoba-whisper v2（±prompt 同値） | ローカル | 5.9% | 6/6 | 4/6 | 6/6 | 12.8 秒 |
| ReazonSpeech＋hotwords | ローカル | 6.1% | 6/6 | 4/6 | 6/6 | 0.4 秒 |
| Deepgram Nova-3（keyterm なし） | クラウド | 6.7% | 6/6 | 2/6 | 6/6 | 1.4 秒 |
| SenseVoice small | ローカル | 8.6% | 6/6 | 2/6 | 6/6 | 0.5 秒 |
| Whisper small | ローカル | 9.3% | **2/6** | 3/6 | 6/6 | 3.5 秒 |
| Whisper base | ローカル | 10.3% | **2/6** | 4/6 | 6/6 | 1.1 秒 |
| Parakeet TDT 0.6B v3 | ローカル | （対象外・日本語非対応でローマ字が出る） | | | | |

6 文・各 3〜4 個の要素（曜日 7 種・架空の固有名詞 3 種・時刻／ページ数／金額）を仕込み、文字誤り率（CER）と「要素が生き残ったか」の両方で採点しています。ローカルは GPU なしの CPU 推論（int8）、クラウドは API の往復時間なので、応答の列は「速さの勝負」ではなく「使う側の待ち時間の目安」として読んでください。

## なぜ同じベンチにクラウドを足したのか

前回の記事では、ローカル Whisper で **曜日が壊れる**（「火曜」が別の語になる）、**数値は壊れない**、**辞書注入は効くが文字数予算がシビア**、という 3 点が分かりました。ただ、ローカルで完結させる前提を一度外すと「クラウドの日本語特化エンジンなら、そもそも壊れないのでは」「よく使われている Deepgram や Google はどうなのか」という疑問が残ります。

AmiVoice API は毎月 60 分までの無料枠、Google と Azure にも月あたりの無料枠、Deepgram と AssemblyAI は登録時の無料クレジット、ElevenLabs は無料プランがあり、今回のベンチは音声合計で 2 分弱です。試さない理由がありませんでした。

## 検証条件

- **入力**: Windows の音声合成（SAPI・日本語女声）で 6 文を 16kHz モノラル WAV に合成。毎回同じ音声なので条件が揃い、何度でも再現できます
- **発話文**: 仕事のメモを想定した文に、曜日（月〜日の 7 種）・架空の固有名詞 3 種（漢字複合「青嵐精機」・カタカナ「ノルディックベーカリー」・英字略称「KTSラボ」）・時刻・ページ数・金額を散らしています。**固有名詞はすべて架空**です
- **クラウド**: AmiVoice 汎用（`-a-general`・`profileWords` に「表記 読み」3 語）／Google Speech-to-Text v2 `chirp_3`・`ja-JP`（inline phrase set・boost 10）／OpenAI `gpt-4o-transcribe`（`prompt` に 3 語）／Deepgram `nova-3`・`language=ja`（`keyterm` 3 語）／Azure AI Speech `ja-JP`（短音声 REST と、Fast transcription API に `phraseLists` 3 語）／ElevenLabs `scribe_v1`（語彙登録の機能なし）／AssemblyAI `universal-3-5-pro`→`universal-2`（`word_boost` 3 語）
- **ローカル**: faster-whisper base/small/medium と kotoba-whisper v2（`initial_prompt` に 3 語）／sherpa-onnx 経由の ReazonSpeech（zipformer・hotwords 3 語）・SenseVoice small・Parakeet TDT 0.6B v3
- **採点**: 期待語が出力に含まれるか（曜日・固有名詞・数値の 3 区分）と、参照文に対する CER。**表記ゆれは認識誤りに数えません**。空白・句読点・中黒の除去、英字の大小、漢数字⇄算用数字（十四時→14 時）、「230 万円」⇄「2300000 円」（Deepgram の smart_format）を正規化してから比較しています

6 文・1 環境・各 1 回という小さな実験です。統計的な主張ではなく「自分の用途で何が壊れるか」の当たりをつけるためのものです。

## 結果 1: 曜日が壊れるのは Whisper の小さいモデルだけだった

クラウド 7 社とローカル非 Whisper 系（ReazonSpeech・SenseVoice・kotoba）は曜日 6/6。壊れたのは Whisper base・small の 2/6 だけでした。壊れ方は前回と同じで、「火曜」→「概要」「通用」、「金曜」→「金用」、「木曜」→「目曜」のように、**文法的には自然に見える別の語**になります。読み返しても気づきにくい種類の誤りです。

曜日は予定のメモで最も致命的な要素です。小さいモデルで運用するなら、ここだけは人が確認するか、曜日を日付に言い換える運用で逃げる必要があります。

## 結果 2: 固有名詞は「登録」が決定打 — 効き方はエンジンでまちまち

登録なしの状態では、どのエンジンも固有名詞を 2〜4/6 しか拾えません。「青嵐精機」が「晴嵐正規」「清卵性器」「セーラン世紀」「青嵐世紀」のように、**音は合っていて漢字だけが外れる**誤りが主流です。

登録機能を使ったときの効き方が、今回一番の収穫でした。

| エンジン | 登録の渡し方 | 固有名詞 | CER の変化 |
|---|---|---|---|
| AmiVoice | 表記＋読み（`profileWords`） | 3/6 → **6/6** | 3.2% → **0.0%** |
| Google Chirp 3 | 表記のみ（adaptation phrase set・boost） | 4/6 → **6/6** | 2.3% → **0.0%** |
| Deepgram | 表記のみ（`keyterm`） | 2/6 → **6/6** | 6.7% → **0.8%** |
| OpenAI | 表記のみ（`prompt`） | 4/6 → **6/6** | 2.3% → 1.4% |
| Whisper medium | 表記のみ（`initial_prompt`） | 4/6 → **6/6** | 5.0% → 2.3% |
| Azure | 表記のみ（Fast transcription の `phraseLists`） | 3/6 → 5/6 | 4.6% → 2.5% |
| ReazonSpeech | 表記のみ（hotwords） | 4/6 → 4/6 | 5.1% → 6.1% |
| AssemblyAI | 表記のみ（`word_boost`） | 4/6 → 4/6 | 変化なし |
| kotoba-whisper | 表記のみ（`initial_prompt`） | 4/6 → 4/6 | 変化なし |

**CER 0 に到達したのは、読みを渡せる AmiVoice と、Google の適応フレーズの 2 つ**でした。Deepgram の keyterm と OpenAI の prompt も表記だけで 6/6 まで持っていきます（Deepgram に残る 0.8% は「KTSラボ」が「kts ラボ」になる表記差のうち、正規化で吸収しきれない分）。Azure はフレーズリストで 5/6 まで改善し、しかも Fast transcription API は 0.5 秒と最速でした。一方、ReazonSpeech の hotwords と AssemblyAI の word_boost は今回の 3 語では効きませんでした。

ここで一つ罠を踏みました。AmiVoice の `profileWords` の値は **URL エンコードが必須**で、エンコードせずに渡すと**エラーにならず黙って無視されます**。最初の計測では登録ありとなしで出力が 1 文字も変わらず、しばらく原因が分かりませんでした。パラメータ文字列 `d` の中で値が空白と `|` を含むため、`表記 読み|表記 読み` をまるごとエンコードして渡す必要があります。

Whisper 側の語彙注入も 6/6 にはなりますが、CER は 2.3% 残ります。「さん」が「3」、「分」が「墳」、「件」が「剣」、「再送」が「再創」のように、固有名詞以外のところで同音異義の取り違えが起きるためです。前回書いたとおり、注入できる文字数には上限があり、増やせば増やすほど別の場所にしわ寄せが出ます。

## 結果 3: 数値はどの条件でも壊れない

時刻・ページ数・金額は Parakeet を除く全条件で 6/6。表記はエンジンごとに違い（Deepgram は「2300000円」、ElevenLabs・ReazonSpeech・Azure 短音声は「二百三十万円」「三ページ」のまま）、これを「誤り」に数えると順位が大きく変わります。**何を正規化するかを決めずに CER を比べると、表記の好みを精度と取り違える**、というのも今回の教訓です。

## ローカル非 Whisper 系で分かったこと

- **ReazonSpeech は速い（0.5 秒）が、1 発話向け**。文と文の間に無音があると、片方の文をまるごと落としました（文頭の句が消える・pad を足すと今度は文末が消える）。Silero VAD で発話区間に分割し、前後に 0.3 秒の無音を足してから渡すと 6/6・CER 5.1% まで回復しました。長い音声をそのまま渡す用途ではなく、VAD と組み合わせる前提のモデルです
- **SenseVoice は速くて曜日・数値は堅いが、固有名詞が弱い**（2/6）。出力に分かち書きの空白が入るので、後処理で詰める必要があります
- **kotoba-whisper v2 は medium 相当の精度**（CER 5.9%・固有名詞 4/6）だが CPU では 1 文 10 秒超。`initial_prompt` は今回効きませんでした
- **Parakeet TDT 0.6B v3 は日本語非対応**。25 言語対応ですが欧州言語が中心で、日本語を入れるとローマ字のような文字列が返ります。対象外にしました

## 仕組みの違い: 読みを渡せるか、表記だけか

| | 読みを渡す | 表記だけ渡す（バイアス） |
|---|---|---|
| 該当 | AmiVoice `profileWords`（マイページ辞書も） | Google 適応フレーズ・Deepgram keyterm・OpenAI/Whisper prompt・Azure フレーズリスト・ReazonSpeech hotwords・AssemblyAI word_boost |
| 効き方 | 読みを明示するので、音が合っていれば表記が確定する | 「この語が出やすい」バイアス。エンジンの内部表現との相性で効き方が大きく違う（今回は Google・Deepgram・OpenAI・Whisper が 6/6、Azure 5/6、ReazonSpeech・AssemblyAI は変化なし） |
| 上限・副作用 | 登録語数に応じてプロファイル読み込みが遅くなる | Whisper は条件付け窓の約 224 トークン。文字数を使うほど他の語に影響。Deepgram は従量の追加料金 |

日本語の固有名詞は「音は合っているが漢字が違う」誤りが主流なので、読み付きの辞書と相性がいい、という当たり前の結論です。ただし表記だけでも、Google と Deepgram のように十分に効くエンジンがあることも分かりました。

## 使い分けの目安（自分の用途で）

- **精度最優先・ネット接続可**: AmiVoice＋単語登録、または Google Chirp 3＋適応フレーズ。どちらも今回 CER 0。AmiVoice は応答 1.4 秒で無料枠 60 分/月、Google は 3 秒前後で無料枠 60 分/月。英語混じりなら Deepgram＋keyterm か OpenAI＋prompt
- **速度最優先・ネット接続可**: Azure Fast transcription＋フレーズリスト（0.5 秒・固有名詞 5/6）
- **完全オフライン必須・精度優先**: Whisper medium＋語彙注入。CPU でも実用になりますが、1 文 8 秒前後の待ちは覚悟が要ります
- **完全オフライン必須・速度優先**: ReazonSpeech＋VAD。0.5 秒で返り、曜日・数値は堅い。固有名詞は後処理の辞書置換で補う
- **やってはいけない組み合わせ**: 小さい Whisper で曜日入りの予定メモ。壊れても気づけません

## 踏んだ罠まとめ

1. AmiVoice `profileWords` は URL エンコードしないと無視される（エラーは出ない）
2. AmiVoice のログ保存なしオプション（`loggingOptOut=True`）は、無料枠の API キーでは「認証不正」になる。ログ保存ありの前提で使う
3. AmiVoice の API キーは「接続情報 → 共通接続情報」の長い文字列。サービス ID（短い方）を入れても同じ「認証不正」が返るので、エラー文だけでは区別できない
4. 環境変数経由でキーを渡すとき、末尾の改行が混じると 3 と同じエラーになる。`strip()` を忘れない
5. AssemblyAI は `speech_model` が廃止済みで 400 になる。`speech_models: ["universal-3-5-pro", "universal-2"]` の配列で渡す
6. Google は Speech-to-Text API の有効化に請求先アカウントのリンクが要る（無料枠内でも）。サービスアカウント鍵の作成は組織ポリシーで止まることがあり、その場合は CLI で制約を外す
7. Azure の短音声 REST にはフレーズリストが無い。使うなら Fast transcription API の `phraseLists`
8. ReazonSpeech に複数文をそのまま渡すと文を落とす。VAD で分割してから
9. 数値の表記（算用／漢数字／万の展開）を正規化しないと、表記の違いを精度差と誤読する

## まとめ

- **曜日**: 壊れるのは Whisper base/small だけ（2/6）。他はすべて 6/6
- **固有名詞**: AmiVoice（読み付き登録）と Google（適応フレーズ）が CER 0.0%。Deepgram 0.8%・OpenAI 1.4% も 6/6。Whisper の語彙注入は 6/6 になるが別の場所に 2.3% 残る。ReazonSpeech・AssemblyAI・kotoba の登録機能は今回効かず
- **数値**: どの条件でも壊れない（表記だけが違う）
- **速度**: クラウド最速は Azure Fast transcription（0.5 秒）、ローカル最速は ReazonSpeech（0.5 秒）。Whisper medium と kotoba は CPU だと 8〜13 秒

限界も書いておきます。合成音声 1 種類・6 文・各 1 回で、肉声やノイズ環境は未検証です。AmiVoice の他エンジン（医療・金融など）、各社のストリーミング API、Google の他モデルも試していません。数字は「この条件でこうだった」以上のことを言っていません。

前回のローカル Whisper だけの実測は[こちら](https://zenn.dev/wisteriacorp/articles/wisteria-218be5072ded)、個人の AI 基盤の全体像は[こちら](https://zenn.dev/wisteriacorp/articles/wisteria-47f423fb1c16)にまとめています。
