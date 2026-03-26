# Codex CLI Hardening Cheatsheet

Codex CLI を安全寄りに運用するための実務向けチートシートです。一般的なハードニングの考え方と、Codex CLI における `sandbox / approval / network / history` の有効な実装例をまとめています。

Claude Code 向けチートシートの「現場でそのまま使える構成」を参考にしつつ、内容は Codex CLI の公式ドキュメントに合わせて整理しています。

## まず意識したいリスク

Codex CLI は、ファイルの読み書き、シェルコマンド実行、設定変更、場合によっては外部通信まで扱えます。
便利な反面、設定が緩いと次のようなリスクが生じます。

- **間接的な prompt injection で意図しない行動を取る**
  取得した README、Issue、ドキュメント、生成物の中に埋め込まれた指示に引っ張られ、想定外のコマンドや参照を始めることがあります（[OWASP LLM01:2025 Prompt Injection](https://genai.owasp.org/llm-top-10/)）。
- **権限が広すぎて、被害がそのまま拡大する**
  `danger-full-access` や広い writable root を与えると、誤った判断がそのまま強い操作になります（[OWASP LLM06:2025 Excessive Agency](https://genai.owasp.org/llm-top-10/)）。
- **履歴や文脈に機密情報が残る**
  プロンプト、差分説明、接続先、内部 URL、トークン断片などが端末ローカルに残る可能性があります（[OWASP LLM02:2025 Sensitive Information Disclosure](https://genai.owasp.org/llm-top-10/)）。
- **もっともらしい提案を過信する**
  モデルの提案や設定解釈が正しそうに見えても、検証なしで適用すると危険です（[OWASP LLM09:2025 Misinformation](https://genai.owasp.org/llm-top-10/)）。
- **profile 名や承認方針が曖昧で、運用事故が起きる**
  これは構成管理上の問題ですが、実際には権限の誤用や review 漏れにつながります。

このチートシートは、こうしたリスクを抑えながら、普段の開発では過度に重くしすぎない設定を考えるためのものです。

関連する外部資料:

- OWASP GenAI Security Project: https://genai.owasp.org/
- OWASP Prompt Injection: https://owasp.org/www-community/attacks/PromptInjection
- OWASP LLM Prompt Injection Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/LLM_Prompt_Injection_Prevention_Cheat_Sheet.html

## 目的

- 一般的な環境ハードニングのコツを、Codex CLI の運用に落とし込む
- `sandbox / approval / network / history` を中心に、安全寄りの設定例をすぐ使える形で示す
- コメント付き `config.toml` テンプレートへの導線を用意する

## セキュア設計の基本原則

### Human-In-The-Loop

影響の大きい操作を完全自動にしない、という考え方です。
Codex CLI では、まず `approval_policy = "on-request"` を共通デフォルトにするのが分かりやすい実装です。

これにより、次のような操作で人間の判断を挟みやすくなります。

- ワークスペース境界を超える変更
- ネットワーク利用を伴う処理
- 破壊的なコマンド実行
- 想定外の権限要求

### 最小権限の原則

最初から強い権限を与えず、必要な時だけ最小限を追加する、という原則です。
Codex CLI では次の設定がこの考え方に対応します。

- `sandbox_mode = "workspace-write"`
- `writable_roots = []`
- `network_access = false`
- `allow_login_shell = false`

必要な時だけ `--add-dir` や `net_enabled` profile で例外を与える方が、安全性と実用性の両立がしやすくなります。

### 多層防御

1つの設定に頼らず、複数の制御を重ねる考え方です。
たとえば Codex CLI では、次のような組み合わせが多層防御になります。

- `sandbox` で書き込み範囲を絞る
- `approval` で高リスク操作に確認を入れる
- `network` を既定で無効にする
- `history` を抑えて情報残留を減らす

どれか 1 つが不十分でも、他の層が事故や誤実行の影響を小さくします。

### 明示的な運用境界

設定ファイルだけでなく、profile 名や運用ルールも安全設計の一部です。
名前と実態がずれると、設定そのものより先に運用ミスが起きます。

そのため、このチートシートでは次のような方針を取ります。

- `readonly_quiet` のように用途が分かる名前を使う
- `full_auto` のような誤解を招きやすい名前を避ける
- 共通テンプレートは単純に保ち、例外は profile や一時オプションで与える

## Codex CLI 特有のポイント

Claude Code など他の coding agent と共通する原則もありますが、Codex CLI には設定モデル上の特徴があります。
この違いを押さえておくと、なぜこのチートシートがこの形になっているか理解しやすくなります。

### 1. `sandbox_mode` と `approval_policy` が中心になる

Codex CLI では、何をどこまで自動で実行させるかを、
主に `sandbox_mode` と `approval_policy` の組み合わせで考えます。

- `sandbox_mode`: どこまで書き込みや実行を許すか
- `approval_policy`: どこで人間の承認を挟むか

この 2 つが安全設計の土台です。

### 2. `workspace-write` を基準にしやすい

Codex CLI では `read-only` / `workspace-write` / `danger-full-access` の差が大きく、
普段使いの基準として `workspace-write` が取りやすいのが特徴です。

- `read-only`: 調査や確認向き
- `workspace-write`: 日常の編集向き
- `danger-full-access`: 例外的な用途向き

そのため、共通デフォルトを `workspace-write` にし、必要に応じて前後へ振る設計が分かりやすくなります。

### 3. ネットワーク設定が sandbox 配下にぶら下がる

Codex CLI では `network_access` が `sandbox_workspace_write` 配下で管理されます。
このため、単に「ネットワークをオンにする」ではなく、
どの sandbox 前提でネットワークを許すかを意識した設計になります。

このチートシートで `net_enabled` profile を分けているのも、そのためです。

### 4. profile による切り替えが運用しやすい

Codex CLI は profile 単位で設定差分を持たせやすく、
共通デフォルトを堅めにしつつ、用途ごとに切り替える運用と相性が良いです。

たとえば:

- `readonly_quiet`
- `local_write`
- `net_enabled`

のように分けると、権限差と用途が一致しやすくなります。

### 5. TOML の構造制約を踏まえる必要がある

Codex CLI の設定は `config.toml` なので、TOML の構造制約そのものが設定ミス要因になります。

たとえば、次のような形はエラーになります。

```toml
approval_policy = "on-request"

[approval_policy.granular]
rules = true
```

これは `approval_policy` を文字列として定義したあとに、
同じキーをテーブルとして拡張しているためです。

つまり、Codex の hardening では「安全な値を選ぶ」だけでなく、
「壊れない TOML にする」ことも重要です。

### 6. `trusted` や login shell など、実務寄りの論点がある

Codex CLI には、単純な allow/deny だけではない運用論点があります。

- `trust_level = "trusted"` は便利だが、安全性とトレードオフになる
- `allow_login_shell = false` は、シェル初期化依存を減らして再現性を上げる
- `history.persistence` は、利便性と情報残留のバランスが問われる

こうした点は、他ツールの一般論だけでは見えにくい、Codex CLI ならではの調整ポイントです。

### 7. `instructions.md` はセキュリティポリシーの置き場ではない

Codex CLI では、バージョンやドキュメント系統によって `~/.codex/instructions.md`、`AGENTS.md`、`CODEX.md` などの案内が揺れている時期があります。
ただし共通して言えるのは、これらは「文脈や作業方針を与えるためのファイル」であり、強制力のあるセキュリティ境界ではないということです。

つまり:

- 命名規約
- コーディング規約
- レビュー方針
- このプロジェクトで注意したい背景事情

を書くのには向いていますが、次のようなものを **ここだけに依存して** 守ろうとするのは危険です。

- ネットワーク禁止
- 特定ディレクトリ以外への書き込み禁止
- 破壊的コマンドの抑止
- 承認必須の運用

これらは `config.toml` の `sandbox` / `approval` / `network` 側で制御し、`instructions.md` 側は補助説明に留める方が安全です。

## 結論

共通デフォルトの出発点としては、次の方針が扱いやすいです。

- `sandbox_mode = "workspace-write"`
- `approval_policy = "on-request"`
- `network_access = false`
- `history.persistence = "save-all"` を基本にし、必要なら `none` へ切り替える
- `allow_login_shell = false`
- `writable_roots = []`

この組み合わせなら、ワークスペース内の通常編集は進めやすく、ネットワークや権限拡大は明示時だけにできます。

## Quick Start

### 最小安全テンプレート

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

[profiles.net_enabled]
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[profiles.net_enabled.sandbox_workspace_write]
network_access = true
exclude_slash_tmp = true
exclude_tmpdir_env_var = true
writable_roots = []
```

### 日常の使い分け

```bash
# 調査だけしたい
codex --profile readonly_quiet

# 通常のローカル編集
codex --profile local_write

# ネットワークが必要な時だけ
codex --profile net_enabled
```

## 一般的なハードニングのコツ

### 1. デフォルトは狭く、必要時だけ広げる

- 常用デフォルトは `workspace-write` に留める
- 例外時だけ `--add-dir` や profile で権限を広げる
- `danger-full-access` は共有デフォルトにしない

### 2. ネットワークは閉じておく

- 外部通信は誤実行時の影響が大きい
- 依存取得、Web 参照、外部 API 利用が必要な時だけ明示的に有効化する
- 外部コンテンツ経由の prompt injection や data exfiltration を考えると、既定で閉じる方が扱いやすい
- たとえば `npm install` で取得したパッケージの README や postinstall 周辺の文脈、参照先ドキュメント、Issue テキストに悪意ある指示が混ざっていた場合、ネットワークが開いているほど影響範囲が広がります

### 3. 履歴は利便性と情報残留のトレードオフで考える

- セッション履歴には、指示文、コマンド意図、作業文脈が含まれうる
- 日常利用では `history.persistence = "save-all"` の方が扱いやすいことが多い
- 機密性を強く優先するなら `history.persistence = "none"` に切り替える
- agent 的な運用では、履歴や文脈の残留自体が漏えい面になりうる
- たとえば API キー入りの貼り付け、DB 接続文字列、社内 URL、チケット番号、障害調査の断片などは、後から見ると想像以上に情報量があります

### 4. 名前と実態をずらさない

- 承認が必要なのに `full_auto` と呼ぶと誤解を招く
- `readonly_quiet`, `local_write`, `net_enabled` のように用途で命名する

### 5. まずは個人設定で固め、例外は一時的に与える

- 共通の基準は `~/.codex/config.toml`
- 一時的な例外は `--profile`, `--add-dir`, `--config` で足す

## 項目別の考え方

### sandbox

Codex CLI の代表的な sandbox は次の 3 つです。

- `read-only`
- `workspace-write`
- `danger-full-access`

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

- `writable_roots = []` は追加書き込み先を増やさない安全な初期値
- `/tmp` や `$TMPDIR` を writable root に含めないと、書き込み面が広がりにくい
- 追加ディレクトリが必要な時は `--add-dir /path/to/dir` を優先する

### approval

Codex CLI の主要な approval policy は次の 3 つです。

- `untrusted`
- `on-request`
- `never`

対話型の通常運用では `on-request` が一番説明しやすく、実務上も無理がありません。

推奨例:

```toml
approval_policy = "on-request"
```

ポイント:

- sandbox 境界を超える操作で止めやすい
- 毎回フル手動レビューほど重くない
- 共通テンプレートに granular 設定まで入れなくてよい
- high-risk action に human-in-the-loop を入れるという意味で、OWASP 的にも説明しやすい

### network

ネットワークは sandbox とは別に管理します。`workspace-write` でも、必要がなければ閉じておくのが基本です。

推奨例:

```toml
[sandbox_workspace_write]
network_access = false
```

必要時だけ profile で有効化:

```toml
[profiles.net_enabled]
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[profiles.net_enabled.sandbox_workspace_write]
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

履歴保存は便利ですが、共通デフォルトとしては強すぎることがあります。
このチートシートでは、実用性を重視して `save-all` を基本にしつつ、
機密性を優先したい場合に `none` を選べる方針にしています。

推奨例:

```toml
[history]
persistence = "save-all"
```

ポイント:

- 日常の振り返りや継続作業には履歴保存が有用
- ただし履歴には指示文や作業文脈が残りうる
- その中には API キー断片、接続先ホスト名、障害調査メモ、内部 URL、顧客固有の識別子が含まれうる
- 共有端末や業務環境では特に有効
- 長期的には persistence attack や文脈残留の扱いも意識したい

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
[profiles.net_enabled]
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[profiles.net_enabled.sandbox_workspace_write]
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
たとえば、誤って `rm -rf` 系の削除、広すぎる `chmod`、意図しない生成物の上書き、ホームディレクトリ配下の設定変更などが、そのままローカル環境へ届きます。
問題は「悪意あるコマンド」だけではなく、「善意だが雑な自動化」がそのまま通ってしまうことです。

### 2. 常時ネットワークオン

```toml
[sandbox_workspace_write]
network_access = true
```

必要性の割に権限が広く、共有テンプレートには向きません。

### 3. 履歴保存を無条件で正当化すること

```toml
[history]
persistence = "save-all"
```

便利ですが、機密性の高い環境では `none` の方が適切なことがあります。

### 4. `approval_policy` の TOML 衝突

次はエラーになります。

```toml
approval_policy = "on-request"

[approval_policy.granular]
rules = true
```

理由は、`approval_policy` を文字列で定義したあとに、同じキーをテーブルとして拡張しているためです。最小安全テンプレートでは granular 設定を外す方が無難です。

## 導入の仕方

### 個人の標準設定

- `~/.codex/config.toml` に最小安全テンプレートを置く

### プロジェクト固有の調整

- `.codex/config.toml` に、必要な追加設定だけを書く
- 例: 特定リポジトリだけ追加の `writable_roots` が必要な場合

### 一時的な例外

```bash
# 追加ディレクトリだけ書けるようにする
codex --profile local_write --add-dir /path/to/output

# 一時的に network_access を有効化
codex --config sandbox_workspace_write.network_access=true
```

### profile 運用上の注意

- profile は「その起動時の実行姿勢」を決めるものとして扱い、必要なものを毎回明示する方が安全です
- 特に `net_enabled` を使ったあとに次も同じ気分で作業すると、想定より広い権限で進めてしまうことがあります
- CI や自動実行では、profile を明示的に固定し、暗黙のデフォルトに頼らない方が事故を減らせます
- 人間向けの運用でも、「通常は `local_write`、必要時だけ `net_enabled`」を明文化しておくと迷いにくくなります

## 別添ファイル

コメント付きテンプレート:

- [codex_config_min_safe_template.toml](./codex_config_min_safe_template.toml)
- [codex-config.hardened.template.toml](./codex-config.hardened.template.toml)

## 参考リンク

- [Sandboxing](https://developers.openai.com/codex/concepts/sandboxing)
- [Agent approvals & security](https://developers.openai.com/codex/agent-approvals-security)
- [Command line options](https://developers.openai.com/codex/cli/reference)
- [Config basics](https://developers.openai.com/codex/config-basic)
- [Configuration reference](https://developers.openai.com/codex/config-reference)
- [Rules / execpolicy](https://developers.openai.com/codex/rules)
- [OWASP GenAI Security Project](https://genai.owasp.org/)
- [OWASP Prompt Injection](https://owasp.org/www-community/attacks/PromptInjection)
- [OWASP LLM Prompt Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/LLM_Prompt_Injection_Prevention_Cheat_Sheet.html)
- [Claude Code Hardening Cheat Sheet.ja](https://github.com/okdt/claude-code-hardening-cheatsheet/blob/main/Claude_Code_Hardening_Cheat_Sheet.ja.md)
