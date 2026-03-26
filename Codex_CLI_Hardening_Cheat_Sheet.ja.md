# Codex CLI Hardening Cheatsheet

Codex CLI はファイルの読み書き、シェルコマンド実行、設定変更、場合によっては外部通信まで扱えます。これは強力で、同時にリスクが伴います。

このチートシートは、そのリスクをコントロールするための設定ガイドです。`sandbox / approval / network / history` を中心に、安全寄りの構成を Codex CLI の公式ドキュメントに合わせて整理しています。初心者はまず Quick Start のテンプレートだけでも入れてください。設計判断から理解したい人は、通して読んでもらえれば根拠が分かるようにしてあります。

## リスク — なぜハードニング（セキュリティ堅牢化）設定が必要なのか

便利な反面、設定が緩いと次のようなリスクが生じます。

- **間接的な prompt injection で意図しない行動を取る**
  たとえば取得した README、Issue、ドキュメント、生成物の中に埋め込まれた指示に引っ張られ、「このパッケージを追加しておきましょう」「先にこのスクリプトを実行しますね」と想定外の行動を始めることがあります。（[OWASP LLM01:2025 Prompt Injection](https://genai.owasp.org/llm-top-10/)）
- **権限が広すぎて、被害がそのまま拡大する**
  `danger-full-access` や広い writable root を与えると、誤った判断がそのまま強い操作になります。「整理しておきますね」とワークスペース外のファイルまで手を出したり、「修正しておきました」と `.gitconfig` を書き換えたり——技術的には正しくても、あなたの意図は超えています。（[OWASP LLM06:2025 Excessive Agency](https://genai.owasp.org/llm-top-10/)）
- **履歴や文脈に機密情報が残る**
  プロンプト、差分説明、接続先、内部 URL、トークン断片などが端末ローカルに残る可能性があります。後から見返すと、想像以上に情報量があるものです。（[OWASP LLM02:2025 Sensitive Information Disclosure](https://genai.owasp.org/llm-top-10/)）
- **もっともらしい提案を過信する**
  モデルの提案や設定解釈が正しそうに見えても、検証なしで適用すると危険です。自信たっぷりに間違えるのは人間も AI も同じですが、AI は迷った顔を見せてくれません。（[OWASP LLM09:2025 Misinformation](https://genai.owasp.org/llm-top-10/)）
- **profile 名や承認方針が曖昧で、運用事故が起きる**
  これは構成管理上の問題ですが、実際には権限の誤用や review 漏れにつながります。名前が曖昧だと、設定が壊れるより先に人間が間違えます。

これらは仮定の話ではありません。問題が起きたとき——いずれ必ず起きますが——被害を封じ込めたい。このチートシートは、普段の開発を過度に重くせずにそれを実現するための設定ガイドです。

## 基本的なアプローチ

ではどうするか。Codex CLI のハードニングは、`config.toml` の以下の軸で考えます。

1. **sandbox** — `sandbox_mode` で、AI がファイルシステムのどこまで書き込めるかを制御します。`workspace-write` ならワークスペース内だけ、`read-only` なら一切書けません。これは OS レベルの制約なので、AI 側から迂回できません。基本的かつ最強の防御層です。
2. **approval** — `approval_policy` で、sandbox 境界を超える操作に人間の承認を挟むかどうかを決めます。`on-request` にしておけば、想定外の操作で止まります。Human-In-The-Loop の実装です。
3. **network** — `network_access` で、外部通信を許すかどうかを決めます。sandbox 配下の設定ですが、独立した防御層として機能します。閉じておけば、indirect prompt injection で誘導された操作が外部に到達しません。
4. **history** — `history.persistence` で、セッション履歴をどこまで残すかを決めます。利便性と情報残留のトレードオフです。
5. **ログ** — ここはエンタープライズ現場や、この仕組み自体のデバッグで必要になるかもしれません。Codex CLI は OpenTelemetry 統合とセッション rollout を持っており、公式ドキュメントで目立つ扱いではないものの、実は相当充実しています。詳細は後述します。

> **ポイント:** これら複数の防御を重ねるアプローチを**多層防御**といいます。どれか 1 つが不十分でも、他の層が事故の影響を小さくします。

### `instructions.md` に書いておけばいいのでは

Codex CLI では、`AGENTS.md` のような project 文脈用ファイルや、`~/.codex/instructions.md` のようなグローバル指示ファイルを使えます。
それぞれ意図やスコープは違いますが、共通して言えるのは、これらは文脈や作業方針を与えるためのファイルであって、セキュリティ上の制限を強制する場所ではないということです。

たとえば、次のような制御を instruction 系ファイルだけに頼って実現しようとするのは危険です。

- ネットワーク禁止
- 特定ディレクトリ以外への書き込み禁止
- 破壊的コマンドの抑止
- 承認必須の運用

こうした制御は `config.toml` の `sandbox` / `approval` / `network` 側で担保し、instruction 系ファイルは補助説明に留める方が安全です。

また、instruction 系ファイルは大きくしすぎない方がよいです。
OpenAI の公式情報では、`AGENTS.md` などの user instructions は 32 KiB の上限の対象とされています。
そのため、巨大な運用マニュアルを 1 ファイルに詰め込むより、短く保守しやすい指針に絞る方が実務的です。

「起きてほしくないこと」に対しては、お願いではなく強制で止める——ではどうしたら良いのか、というところがこのドキュメントの出発点です。

## セキュア設計の基本原則

### Human-In-The-Loop

影響の大きい操作を完全自動にしない、という考え方です。
AI は技術的には正しいが意図を超えた操作をすることがあります——「整理しておきます」とファイルを消したり、「最新にしておきました」と force-push したり。こうした「善意のやりすぎ」は、人間の目が入るポイントがあれば止められます。

Codex CLI では、まず `approval_policy = "on-request"` を共通デフォルトにするのが分かりやすい実装です。
これにより、次のような操作で人間の判断を挟みやすくなります。

- ワークスペース境界を超える変更
- ネットワーク利用を伴う処理
- 破壊的なコマンド実行
- 想定外の権限要求

デフォルトでは、Codex CLI はあなたのユーザーアカウントでできることは何でもできます。たった一度の承認で、破壊的なコマンドがそのまま通ることもあります。`on-request` はその「一度」に確認を入れるための仕組みです。

### 最小権限の原則

最初から強い権限を与えず、必要な時だけ最小限を追加する、という原則です。
「あとで絞る」は大抵うまくいきません。広い権限で始めると、どこまでが本当に必要だったのか分からなくなるからです。逆に、狭いところから始めて「これが足りない」と気づいた時だけ足す方が、結果的に何が必要かも明確になります。

Codex CLI では次の設定がこの考え方に対応します。

- `sandbox_mode = "workspace-write"`
- `writable_roots = []`
- `network_access = false`
- `allow_login_shell = false`

必要な時だけ `--add-dir` や `remote_enabled` profile で例外を与える方が、安全性と実用性の両立がしやすくなります。

### 多層防御

1つの設定に頼らず、複数の制御を重ねる考え方です。
セキュリティの世界では「単一障害点を作らない」とよく言いますが、設定も同じです。sandbox だけに頼ると、sandbox の設定ミス一つで全部が崩れます。

前述の sandbox / approval / network / history の 4 軸を重ねるのは、まさにこの考え方の実装です。sandbox の設定が甘くても、approval で止まる。approval をうっかり通しても、network が閉じていれば外部への影響は出ない——そういう重なりです。

### 明示的な運用境界

設定ファイルだけでなく、profile 名や運用ルールも安全設計の一部です。
名前と実態がずれると、設定そのものより先に運用ミスが起きます。承認が必要な profile を `full_auto` と呼んでいたら、人間がそのつもりで使います。設定が正しくても、名前が間違っていれば事故は起きます。

そのため、このチートシートでは次のような方針を取ります。

- `readonly_quiet` のように用途が分かる名前を使う
- `full_auto` のような誤解を招きやすい名前を避ける
- 共通テンプレートは単純に保ち、例外は profile や一時オプションで与える

## Quick Start

以下を `~/.codex/config.toml` に置いてください。共通デフォルトの出発点としてはこれが扱いやすいです。ワークスペース内の通常編集は進めやすく、ネットワークなど権限の拡大は指示が必要となり、ゴーサインを明示した場合に限定されます。

```toml
model = "gpt-5.4"
approval_policy = "on-request"
sandbox_mode = "workspace-write"
allow_login_shell = false

[history]
persistence = "save-all"

[sandbox_workspace_write]
network_access = false
exclude_slash_tmp = true
exclude_tmpdir_env_var = true
writable_roots = []

[profiles.readonly_quiet]
approval_policy = "never"
sandbox_mode = "read-only"

[profiles.local_write]
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[profiles.remote_enabled]
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[profiles.remote_enabled.sandbox_workspace_write]
network_access = true
exclude_slash_tmp = true
exclude_tmpdir_env_var = true
writable_roots = []
```

## 一般的なハードニングのコツ

### 1. デフォルトは狭く、必要時だけ広げる

- 常用デフォルトは `workspace-write` に留める
- 例外時だけ `--add-dir` や profile で権限を広げる
- `danger-full-access` は共有デフォルトにしない——これを常用にするのは、全員に root パスワードを渡すようなものです

### 2. ネットワークは閉じておく

- 外部通信は誤実行時の影響が大きい——ローカルの事故はローカルで済むが、ネットワークが開いているとデータが外に出る可能性がある
- 依存取得、Web 参照、外部 API 利用が必要な時だけ明示的に有効化する
- 外部コンテンツは indirect prompt injection の入口になりうる。`npm install` で取得したパッケージの README、postinstall スクリプト、参照先ドキュメント、Issue テキスト——こうした場所に悪意ある指示が混ざっていた場合、ネットワークが開いているほど行動半径が広がります
- data exfiltration（情報の持ち出し）の観点でも、既定で閉じておく方が扱いやすい

### 3. 履歴は利便性と情報残留のトレードオフで考える

- セッション履歴には、指示文、コマンド意図、作業文脈が含まれうる
- 日常利用では `history.persistence = "save-all"` の方が扱いやすいことが多い
- 機密性を強く優先するなら `history.persistence = "none"` に切り替える
- agent 的な運用では、履歴や文脈の残留自体が漏えい面になりうる
- たとえば API キー入りの貼り付け、DB 接続文字列、社内 URL、チケット番号、障害調査の断片などは、後から見ると想像以上に情報量があります

### 4. 名前と実態をずらさない

- 承認が必要なのに `full_auto` と呼ぶと誤解を招く
- `readonly_quiet`, `local_write`, `remote_enabled` のように用途で命名する

### 5. まずは個人設定で固め、例外は一時的に与える

- 共通の基準は `~/.codex/config.toml`
- 一時的な例外は `--profile`, `--add-dir`, `--config` で足す

## 項目別の考え方

### sandbox

Codex CLI の代表的な sandbox は次の 3 つです。

- `read-only`: 一切書き込めない。調査・確認専用
- `workspace-write`: ワークスペース内だけ書ける。日常の基準
- `danger-full-access`: 名前の通り、ほぼ何でもできる。ホームディレクトリの `.gitconfig`、`.bashrc`、`~/.ssh/config` すら書き換え可能

安全寄りの普段使いなら `workspace-write` が基準です。ローカル編集は許しつつ、フルアクセスまでは広げません。

推奨例:

```toml
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
exclude_slash_tmp = true
exclude_tmpdir_env_var = true
writable_roots = []
```

ポイント:

- `writable_roots = []` は追加書き込み先を増やさない安全な初期値。ここにディレクトリを追加するほど、AI が書き込める範囲が広がります
- `/tmp` や `$TMPDIR` を writable root に含めると、一時ファイル経由で sandbox 外にデータを受け渡す経路ができてしまいます
- 追加ディレクトリが必要な時は `--add-dir /path/to/dir` で一時的に足す方が、設定を恒久的に広げるより安全です

### approval

Codex CLI の主要な approval policy は次の 3 つです。

- `untrusted`: すべての操作に承認が必要。最も堅いが、使い続けるには重い
- `on-request`: sandbox 境界を超える操作で承認を求める。日常運用の現実的な落としどころ
- `never`: 承認なし。`read-only` sandbox と組み合わせる調査用途には合うが、書き込み権限と組み合わせると危険

対話型の通常運用では `on-request` が一番説明しやすく、実務上も無理がありません。

推奨例:

```toml
approval_policy = "on-request"
```

ポイント:

- sandbox 境界を超える操作で止めやすい
- 毎回フル手動レビューほど重くない
- 共通テンプレートに granular 設定まで入れなくてよい
- high-risk action に human-in-the-loop を入れるという意味で、[OWASP の AI Agent Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html) が推奨する方針とも一致します

### network

ネットワークは sandbox とは別に管理します。`workspace-write` でも、必要がなければ閉じておくのが基本です。

推奨例:

```toml
[sandbox_workspace_write]
network_access = false
```

必要時だけ profile で有効化:

```toml
[profiles.remote_enabled]
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[profiles.remote_enabled.sandbox_workspace_write]
network_access = true
exclude_slash_tmp = true
exclude_tmpdir_env_var = true
writable_roots = []
```

ポイント:

- リモートのコンテンツ取得は、indirect prompt injection の入口になりうる
- まず閉じておき、本当に必要な時だけ profile で開ける方が安全
- 具体例として、外部 README や Issue を参照した結果、モデルが「まず依存を追加しよう」「このスクリプトを実行しよう」と誘導されると、network が開いているほど行動半径が広がります

### history

履歴保存は便利ですが、何が残るかを意識しておく必要があります。
セッション履歴には、あなたが入力したプロンプト、AI の応答、実行されたコマンドとその結果が含まれます。つまり、作業中に触れた情報——API キー断片、接続先ホスト名、障害調査メモ、内部 URL、顧客固有の識別子——がそのまま残ります。

このチートシートでは、実用性を重視して `save-all` を基本にしつつ、
機密性を優先したい場合に `none` を選べる方針にしています。

推奨例:

```toml
[history]
persistence = "save-all"
```

ポイント:

- 日常の振り返りや継続作業には履歴保存が有用
- 共有端末や業務環境では、誰がその履歴を読めるかを確認しておくこと
- agent 的な運用（自動実行の繰り返し）では、履歴の蓄積自体が情報漏えい面になりうる
- 長期的には persistence attack（過去の文脈を利用した攻撃）の扱いも意識したい

### その他の実務寄りの設定

sandbox / approval / network / history の 4 軸以外にも、Codex CLI ならではの調整ポイントがあります。

- **`trust_level = "trusted"`** は便利だが、安全性とトレードオフになる。信頼したコンテキストから injection が来た場合、そのまま通ります
- **`allow_login_shell = false`** は、シェル初期化依存を減らして再現性を上げる。`.bashrc` や `.zshrc` に書かれたエイリアスや PATH 変更が意図せず AI の行動に影響するのを防ぎます

## 有効な実装例

### 1. 読み取り専用で静かに調査する

```toml
[profiles.readonly_quiet]
approval_policy = "never"
sandbox_mode = "read-only"
```

向いている用途:

- 初見リポジトリの確認
- 構造把握
- 変更前レビュー

### 2. 通常のローカル編集

```toml
[profiles.local_write]
approval_policy = "on-request"
sandbox_mode = "workspace-write"
```

向いている用途:

- ローカル修正
- テスト実行
- 小規模なリファクタリング

### 3. ネットワークが必要な時だけ切り替える

```toml
[profiles.remote_enabled]
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[profiles.remote_enabled.sandbox_workspace_write]
network_access = true
exclude_slash_tmp = true
exclude_tmpdir_env_var = true
writable_roots = []
```

向いている用途:

- 依存関係の取得
- 公式ドキュメントの参照
- 外部 API を伴う最小限の作業

## 避けたい設定

### 1. 常用の `danger-full-access`

```toml
sandbox_mode = "danger-full-access"
approval_policy = "never"
```

これはほぼフルアクセスです。隔離済みの外部環境でない限り、共通デフォルトにすべきではありません。

問題は「悪意あるコマンド」だけではありません。AI は技術的に正しいことを、あなたの意図を超えてやります。「整理しておきますね」とワークスペース外のファイルを消す。「設定を直しておきました」と `.gitconfig` を書き換える。「古いキャッシュを削除しておきました」と `/tmp` 配下を一掃する。どれも技術的には間違っていないかもしれませんが、あなたが頼んだことではありません。

`danger-full-access` + `approval_policy = "never"` は、この「善意だが雑な自動化」をすべて素通しにします。

### 2. 常時ネットワークオン

```toml
[sandbox_workspace_write]
network_access = true
```

必要性の割に権限が広く、共有テンプレートには向きません。ネットワークが常時開いていると、indirect prompt injection で誘導された操作がそのまま外部に到達します。また、意図せずトークンや内部情報が外部リクエストに乗ることもあります。「閉じておいて、必要な時だけ開ける」方が事故の範囲を限定できます。

### 3. 履歴保存を無条件で正当化すること

```toml
[history]
persistence = "save-all"
```

便利ですが、機密性の高い環境では `none` の方が適切なことがあります。セッション履歴にはプロンプト、差分説明、接続先、作業文脈が丸ごと残ります。共有端末や退職者のアカウント引き継ぎ時に、想像以上の情報が読めてしまうことがあります。

## 導入の仕方

### 個人の標準設定

- `~/.codex/config.toml` に最小安全テンプレートを置く

### プロジェクト固有の調整

- `.codex/config.toml` に、必要な追加設定だけを書く
- 例: 特定リポジトリだけ追加の `writable_roots` が必要な場合

### 日常の使い分け

テンプレートに profile を定義しておけば、用途に応じて切り替えられます。

```bash
# 調査だけしたい
codex --profile readonly_quiet

# 通常のローカル編集
codex --profile local_write

# ネットワークが必要な時だけ
codex --profile remote_enabled
```

### 一時的な例外

profile で足りない場合は、一時オプションで補えます。

```bash
# 追加ディレクトリだけ書けるようにする
codex --profile local_write --add-dir /path/to/output

# 一時的に network_access を有効化
codex --config sandbox_workspace_write.network_access=true
```

### profile 運用上の注意

- profile は「その起動時の実行姿勢」を決めるものとして扱い、必要なものを毎回明示する方が安全です
- 特に `remote_enabled` を使ったあとに次も同じ気分で作業すると、想定より広い権限で進めてしまうことがあります。権限は「上げた時」より「戻し忘れた時」に事故になります
- CI や自動実行では、profile を明示的に固定し、暗黙のデフォルトに頼らない方が事故を減らせます。スクリプトの中で profile 指定を省略すると、デフォルト（＝最も広い設定）が使われることがあります
- 人間向けの運用でも、「通常は `local_write`、必要時だけ `remote_enabled`」を明文化しておくと迷いにくくなります

## 別添ファイル

コメント付きテンプレート:

- [codex_config_min_safe_template.toml](./codex_config_min_safe_template.toml)
- [codex-config.hardened.template.toml](./codex-config.hardened.template.toml)

## OpenAI 公式資料

- [Sandboxing](https://developers.openai.com/codex/concepts/sandboxing)
- [Agent approvals & security](https://developers.openai.com/codex/agent-approvals-security)
- [Command line options](https://developers.openai.com/codex/cli/reference)
- [Config basics](https://developers.openai.com/codex/config-basic)
- [Configuration reference](https://developers.openai.com/codex/config-reference)
- [Rules / execpolicy](https://developers.openai.com/codex/rules)

## 参考資料

- [OWASP GenAI Security Project](https://genai.owasp.org/)
- [OWASP Prompt Injection](https://owasp.org/www-community/attacks/PromptInjection)
- [OWASP LLM Prompt Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/LLM_Prompt_Injection_Prevention_Cheat_Sheet.html)
- [Claude Code Hardening Cheat Sheet.ja](https://github.com/okdt/claude-code-hardening-cheatsheet/blob/main/Claude_Code_Hardening_Cheat_Sheet.ja.md)
