# Codex CLI Hardening Cheatsheet

Codex CLI はファイルの読み書き、シェルコマンド実行、設定変更、場合によっては外部通信まで扱えます。強力です。そのぶんリスクもあります。

このチートシートは、そのリスクをコントロールするための設定ガイドです。サンドボックス(sandbox)・承認(approval)・ネットワーク(network)・履歴(history)・**シークレット(secrets)** を中心に、安全寄りの構成を Codex CLI の公式ドキュメントに合わせて整理しました。

急ぐ方は、「導入の仕方」の Quick Start をそのまま `~/.codex/config.toml` に置いてください。それだけでも効きます。なぜその値なのかを知りたい方は、通して読んでください。

**この文書ごと Codex に読ませて、手元の `config.toml` を調整させる**のも有効な使い方です。そのために、設定キーとデフォルト値、そして「なぜその値か」の根拠を、いずれも省略せずに書いてあります。用途を伝えたうえで相談すれば、あなたの環境に合わせた形に落としてくれます。

そのまま貼り付けて使えるプロンプトを同梱しています → [Codex_CLI_Hardening_Audit_Prompt.ja.md](./Codex_CLI_Hardening_Audit_Prompt.ja.md)

> **検証環境:** Codex CLI 0.146.0（2026-08-03 時点）。設定キーと挙動は版によって変わります。本文の記述は、公式ドキュメント・公開ソース・実機の挙動を突き合わせて確認しています。公式ドキュメントと実機が食い違う箇所があったため、公式の記述だけを根拠にはしていません。食い違いはその旨を明記しました。「0.146.0 で確認」と書いた箇所は、この版の実装で裏を取ったものです。版が上がったら、まずそこを確かめ直してください。

## リスク — なぜハードニング（セキュリティ堅牢化）設定が必要なのか

便利な反面、設定が緩いと次のようなリスクが生じます。

- **間接的な prompt injection で意図しない行動を取る**
  たとえば取得した README、Issue、ドキュメント、生成物の中に埋め込まれた指示に引っ張られ、「このパッケージを追加しておきましょう」「先にこのスクリプトを実行しますね」と想定外の行動を始めることがあります。（[OWASP LLM01:2025 Prompt Injection](https://genai.owasp.org/llm-top-10/)）
- **権限が広すぎて、被害がそのまま拡大する**
  `danger-full-access` や広い writable root を与えると、誤った判断がそのまま強い操作になります。「整理しておきますね」とワークスペース外のファイルまで手を出したり、「修正しておきました」と `.gitconfig` を書き換えたり——技術的には正しくても、あなたの意図は超えています。（[OWASP LLM06:2025 Excessive Agency](https://genai.owasp.org/llm-top-10/)）
- **シークレットが設定ファイルに転記され、そのまま広がる**
  API キーやトークンを `config.toml` に直接書いてしまうと、平文のままディスクに残ります。ホームディレクトリを同期していれば、書いた瞬間にクラウドと全端末へ複製されます。設定ファイルはバックアップにも、うっかりの共有にも乗りやすい。**シークレットは、置いた場所ではなく広がった先で漏れます。**（[OWASP LLM02:2025 Sensitive Information Disclosure](https://genai.owasp.org/llm-top-10/)）
- **履歴や文脈に機密情報が残る**
  プロンプト、差分説明、接続先、内部 URL、トークン断片などが端末ローカルに残る可能性があります。後から見返すと、想像以上に情報量があるものです。（[OWASP LLM02:2025 Sensitive Information Disclosure](https://genai.owasp.org/llm-top-10/)）
- **もっともらしい提案を過信する**
  モデルの提案や設定解釈が正しそうに見えても、検証なしで適用すると危険です。自信たっぷりに間違えるのは人間も AI も同じですが、AI は迷った顔を見せてくれません。（[OWASP LLM09:2025 Misinformation](https://genai.owasp.org/llm-top-10/)）
- **profile 名や承認方針が曖昧で、運用事故が起きる**
  これは構成管理上の問題ですが、実際には権限の誤用や review 漏れにつながります。名前が曖昧だと、設定が壊れるより先に人間が間違えます。

これらは仮定の話ではありません。問題はいずれ起きます。分かれ目は、起きたときに封じ込められるかどうかです。

かといって、普段の開発が重くなっては続きません。その両立点を探すのがこのチートシートです。

## 基本的なアプローチ

ではどうするか。Codex CLI のハードニングは、`config.toml` の次の軸で考えていきます。

1. **サンドボックス化** — `sandbox_mode` で、AI がファイルシステムのどこまで書き込めるかを制御します。`workspace-write` ならワークスペース内だけ、`read-only` なら一切書けません。これは OS レベルの制約なので、AI 側から迂回できません。もっとも硬い防御層です。
2. **承認ポリシー** — `approval_policy` で、sandbox 境界を超える操作に人間の承認を挟むかどうかを決めます。`on-request` にしておけば、想定外の操作で止まります。Human-In-The-Loop の実装です。
3. **ネットワーク制限** — `network_access` で、サンドボックス内で動くコマンドとその子プロセスの外向き通信を許すかどうかを決めます。sandbox 配下の設定ですが、独立した防御層として機能します。閉じておけば、間接的プロンプトインジェクションで誘導されたコマンドは、そこから外へ出られません。Codex 全体の通信を止める設定ではないので、Web 検索・MCP・hooks は別に評価してください（§3・§6・§10）。
4. **シークレットの扱い** — API キーやトークンを、設定ファイルに書かずに渡します。ここだけは「設定を締める」話ではなく「値をどこに置くか」の話で、他の軸とは種類が違います。そして漏れたときの取り返しがつかない度合いも違います。
5. **履歴保存の制限** — `history.persistence` で、セッション履歴をどこまで残すかを決めます。利便性と情報残留のトレードオフです。
6. **ログ履歴** — ここはエンタープライズ現場や、この仕組み自体のデバッグで必要になるかもしれません。Codex CLI は OpenTelemetry 統合とセッション rollout を持っており、公式ドキュメントで目立つ扱いではないものの、実は相当充実しています。詳細は後述します。

> **ポイント:** これら複数の防御を重ねるアプローチを多層防御(defense in depth)といいます。どれか 1 つが不十分でも、他の層が事故の影響を小さくしてくれます。

この 5 + 1 軸が土台です。加えて、Codex CLI に機能が増えたぶん、締める対象も増えました。外部コンテンツの取り込み（Web 検索）と、MCP サーバ。この 2 つも後半で扱います。

### instruction系ファイルに書いておけば？ NO.

Codex CLI でコンテキストを記録するには、`AGENTS.md` のようなプロジェクト向けコンテキストファイルや、`~/.codex/AGENTS.md`（一時的に上書きするなら `~/.codex/AGENTS.override.md`）のようなユーザ向け指示ファイル、`SKILL.md` のような補助的な定義ファイルがあります。ここではこれらをまとめて **instruction 系ファイル** と呼びます。それぞれ意図やスコープは異なりますが、共通してコンテキストや作業方針を与えるためのものです。

ただ、これらは結局のところユーザプロンプトにすぎません。次のような制御を instruction 系ファイルだけに頼るのは危険です。

- ネットワーク禁止
- 特定ディレクトリ以外への書き込み禁止
- 破壊的コマンドの抑止
- 承認(approval)必須の運用

こうした制御は `config.toml` の sandbox / approval / network 側で担保し、instruction 系ファイルは補助説明に留める。この分担が安全です。

サイズの面でも向いていません。instruction 系ファイルには上限があり、OpenAI の公式情報ではデフォルトで 32 KB とされています（設定で変更はできます）。運用マニュアルのように育てる置き場所ではない、ということです。

**「起きてほしくないこと」は、お願いではなくポリシーとして強制する。** ではどうすればいいのか。そこがこのドキュメントの出発点です。

## セキュア設計の基本原則

### Human-In-The-Loop

影響の大きい操作を完全自動にしない。人間の判断(Human-In-The-Loop)を挟む、という考え方です。

AI は、技術的には正しいけれど意図を超えた操作をすることがあります。「整理しておきます」とファイルを消したり、「最新にしておきました」と force-push したり。こうした善意のやりすぎは、人間の目が入るポイントさえあれば止まります。

Codex CLI では、`approval_policy = "on-request"` を共通デフォルトにするのが分かりやすい実装です。次のような操作で人間の判断を挟めます。

- ワークスペース境界を超える変更
- ネットワーク利用を伴う処理
- 破壊的なコマンド実行
- 想定外の権限要求

デフォルトでは、Codex CLI はあなたのユーザーアカウントでできることは何でもできます。たった一度の承認で、破壊的なコマンドがそのまま通ることもあります。`on-request` はその「一度」に確認を入れるための仕組みです。

### 最小権限の原則

最初から強い権限を与えず、必要な時だけ最小限を追加する。最小権限の原則(principle of least privilege)です。

「あとで絞ればいい」は、たいていうまくいきません。広い権限で始めると、どこまでが本当に必要だったのか分からなくなるからです。狭いところから始めて、足りないと気づいた時だけ足す。この順序なら、何が必要かもはっきりします。

Codex CLI では、次の設定がこれに対応します。

- `sandbox_mode = "workspace-write"`
- `writable_roots = []`
- `network_access = false`
- `allow_login_shell = false`
- `inherit = "core"`（`shell_environment_policy`。デフォルトは全継承です）

必要な時だけ `--add-dir` や profile で例外を与える方が、安全性と実用性の両立がしやすくなります。

### 多層防御

1 つの設定に頼らず、複数の制御を重ねる考え方です。

セキュリティの世界では「単一障害点(single point of failure)を作らない」とよく言います。設定も同じで、サンドボックスだけに頼っていると、その設定を一つ間違えただけで全部が崩れます。

sandbox / approval / network / history の 4 軸を重ねるのは、この考え方の実装です。サンドボックスの設定が甘くても、承認で止まる。承認をうっかり通しても、ネットワークが閉じていればそのコマンドは外へ出られない。そういう重なりを作ります。

### 承認は、少ないほど効く

多層防御と表裏の関係にある注意点です。

**毎回尋ねられる設定は、いずれ機械的に承認される設定になります。** プロンプトが多すぎれば、人は中身を読まなくなる。読まれない承認は、防御として数えられません。

だから承認は「頻度」ではなく「置き場所」で設計します。このチートシートが `untrusted` ではなく `on-request` を推すのは、そのためです。ワークスペースの中は自動で進み、外に出るとき——ファイルでもネットワークでも——にだけ止まる。尋ねられる回数が少ないからこそ、尋ねられた時に人は読みます。

裏を返せば、承認を増やさずに範囲を狭められる設定は使い得です。`writable_roots = []` は書き込み先を増やさないだけで、普段の編集を一切妨げません。摩擦ゼロで範囲が狭まる。迷う理由がありません。

### 明示的な運用境界

安全設計の一部は、設定ファイルの外にあります。profile の名前と、運用ルールです。

名前と実態がずれると、設定そのものより先に運用ミスが起きます。承認が必要な profile を `full_auto` と呼んでいたら、人間はそのつもりで使います。**設定が正しくても、名前が間違っていれば事故は起きる。**

そのため、このチートシートでは次の方針を取ります。

- `readonly_quiet` のように用途が分かる名前を使う
- `full_auto` のような誤解を招きやすい名前を避ける
- 共通テンプレートは単純に保ち、例外は profile や一時オプションで与える

## ハードニングしよう

### 1. サンドボックス化

まずはサンドボックスからです。`sandbox_mode` は 3 段階あります。

- `read-only` (default): 一切書き込めない。調査・確認専用
- `workspace-write`: ワークスペース内だけ書ける。日常の基準
- `danger-full-access`: 名前の通り、ほぼ何でもできる。ホームディレクトリの `.gitconfig`、`.bashrc`、`~/.ssh/config` すら書き換え可能

```toml
# ~/.codex/config.toml
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
exclude_slash_tmp = true
exclude_tmpdir_env_var = true
writable_roots = []
```

- `writable_roots = []` は追加書き込み先を増やさない安全な初期値。ここにディレクトリを追加するほど、AI が書き込める範囲が広がります
- `/tmp` や `$TMPDIR` はワークスペースの外にある共有の場所です。サンドボックスの外で動くプログラムが読みにくるもの——ソケット、ロックファイル、シンボリックリンク——が集まり、書いた内容はセッションやプロジェクトを越えて残ります。外しても普段の作業はほとんど妨げられません（`TMPDIR` / `TEMP` / `TMP` は後述の `inherit = "core"` でも子プロセスへ渡ります）。詰まるのは `$TMPDIR` 配下へ実際に書くツールだけで、書き込み拒否のエラーとして現れるので、その起動だけ `--add-dir` で足せます。摩擦がほとんどないなら、締めておく側に倒します
- 信頼境界の異なるプログラムが同居する環境——複数ユーザの共有ホスト、CI ランナー、コンテナ、別のエージェントが動いている端末——では、`/tmp` に置いたものが誰から見えるかを確認してください
- 追加ディレクトリが必要な時は `--add-dir /path/to/dir` で一時的に足す方が、設定を恒久的に広げるより安全です
- `allow_login_shell = false` にしておくと、`.bashrc` や `.zshrc` に書かれたエイリアスや PATH 変更が意図せず AI の行動に影響するのを防げます。シェル初期化への依存を減らすことで再現性も上がります

**注意:**

```toml
# ~/.codex/config.toml — これは避ける
sandbox_mode = "danger-full-access"
approval_policy = "never"
```

これはほぼフルアクセスです。隔離済みの外部環境でない限り、共通デフォルトにすべきではありません。

問題は「悪意あるコマンド」だけではありません。AI は技術的に正しいことを、あなたの意図を超えてやります。「整理しておきますね」とワークスペース外のファイルを消す。「設定を直しておきました」と `.gitconfig` を書き換える。「古いキャッシュを削除しておきました」と `/tmp` 配下を一掃する。

どれも技術的には間違っていないかもしれません。が、あなたが頼んだことではない。`danger-full-access` と `approval_policy = "never"` の組み合わせは、この善意だが雑な自動化をすべて素通しにします。

### 2. 承認ポリシー

Codex CLI の approval policy は 4 つです。

- `untrusted`: 「既知の安全な読み取り専用コマンド」だけが自動承認され、それ以外はすべて承認を求める。最も堅いが、使い続けるには重い
- `on-request` (default): sandbox 境界を超える操作で承認を求める。日常運用の現実的な落としどころ
- `granular`: 承認の種類ごとに「人間に尋ねる / 自動的に拒否する」を決める。これだけは文字列ではなくテーブル形式で書きます（後述）
- `never`: 承認なし。`read-only` sandbox と組み合わせる調査用途には合うが、書き込み権限と組み合わせると危険

```toml
# ~/.codex/config.toml
approval_policy = "on-request"
```

なお `on-failure` は非推奨です。対話的に使うなら `on-request`、非対話で回すなら `never` を選びます。

これは好みの問題ではありません。**非対話実行（`codex exec`）には承認フラグがそもそも無く、`approval_policy` に何を書いていても承認は行われません。** CI やスクリプトの中で、承認を防御として数えることはできない。頼れるのはサンドボックスの側だけです。

- sandbox 境界を超える操作で止めやすい
- 毎回フル手動レビューほど重くない
- 共通テンプレートに granular 設定まで入れなくてよい
- high-risk action に human-in-the-loop を入れるという意味で、[OWASP の AI Agent Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html) が推奨する方針とも一致します
- `[projects."<パス>"]` の `trust_level` は、承認を省略するだけの設定ではありません。`"untrusted"` にすると、そのプロジェクト配下の `.codex/` 層——プロジェクト固有の config、hooks、rules——がまるごと読み込まれなくなります。裏を返せば `"trusted"` は「このリポジトリが持ち込む設定・フック・ルールを実行してよい」と宣言することです。他人のリポジトリを開くときに効いてくるのは、承認の省略よりむしろこちらの側面です
- しかも、プロジェクト層のほうが優先されます。設定レイヤーの優先度は user より project が上なので（0.146.0 で確認）、trusted にしたリポジトリの `.codex/config.toml` は、あなたが `~/.codex/config.toml` に書いた `sandbox_mode` や `approval_policy` や `allow_login_shell` を上書きできます。trust は「承認を省く」宣言ではなく、**そのリポジトリに設定の優先権を渡す**宣言です。trusted にする前に、そのリポジトリの `.codex/config.toml` を読んでください。組織として上限を決めたいなら、ユーザー設定より強い層である `requirements.toml`（後述）で縛ります

**granular — 承認の種類ごとに決める:**

`granular` は、承認プロンプトのカテゴリごとに真偽値を与える形式です。`true` なら人間に尋ね、**`false` ならそのカテゴリの要求を自動的に拒否**します。`false` は「黙って通す」ではなく「拒否する」。ここを取り違えると設計が逆になります。

```toml
# ~/.codex/config.toml
[approval_policy.granular]
sandbox_approval = true      # sandbox 境界を超えるコマンド実行
rules = true                 # execpolicy の prompt ルール由来
mcp_elicitations = true      # MCP サーバからの問い合わせ
request_permissions = false  # request_permissions ツール由来
skill_approval = false       # skill のスクリプト実行由来
```

`approval_policy` の**値そのもの**が granular になるので、`approval_policy = "on-request"` と `[approval_policy.granular]` を同じファイルに併記することはできません。どちらか一方です。

「尋ねられても判断できないものは最初から拒否しておく」という設計ができるので、非対話の自動実行とは相性が良い形式です。ただし `false` にした分だけ、静かに失敗が増えます。**どのカテゴリを閉じたかは運用メモに残してください。** 後から原因を探すとき、これがあるかないかで大きく違います。

**コマンド単位のルール（execpolicy / preview）:**

`approval_policy` はセッション全体の姿勢を決めますが、コマンド単位で `allow` / `prompt` / `forbidden` を制御する仕組みも現在プレビューとして提供されています。Claude Code の deny/ask/allow ルールに相当するものです。`~/.codex/rules/*.rules` に Starlark（Python サブセット）で記述します。

```starlark
# ~/.codex/rules/default.rules
prefix_rule(
    pattern = ["git", "reset", "--hard"],
    decision = "forbidden",
    justification = "destructive operation",
)
```

**このファイルには、あなたが承認した内容が書き込まれることがあります。** 承認ダイアログで永続化にあたる選択——コマンドの許可ルールの追加、ネットワークルールの永続化——をすると、`$CODEX_HOME/rules/default.rules` に追記されます（0.146.0 で確認）。素の承認とセッション限りの承認は追記されません。つまり、その場をしのぐつもりの一度の承認が、恒久的な許可として残ることがあります。一時的に許可を出した前後で、このファイルの差分を確認してください。意図して永続化を選んだのでなければ、検証後に削除します。

現時点ではシェルコマンドのプレフィックスマッチが対象で、Claude Code の `Read(**/.env)` のようなファイル操作単位のルールはありません。`config.toml` で `approval_policy` を granular にし `rules = true` にすると `prompt` ルールが有効になります。現時点では[プレビュー扱い](https://github.com/openai/codex/blob/main/codex-rs/execpolicy/README.md)のため、API に破壊的変更が入る可能性があります。詳細は [Rules / execpolicy](https://developers.openai.com/codex/rules) を参照してください。

### 3. ネットワーク制限

ネットワークは sandbox 配下の設定ですが、他の sandbox 設定とは独立して判断・設定することができます。**`network_access` のデフォルト値は `false`** で、公式も「`workspace-write` のデフォルトはネットワークを無効のままにする」と明記しています。つまり何も書かなければ、外向き通信は閉じた状態から始まります。

```toml
# ~/.codex/config.toml
sandbox_mode = "workspace-write"   # この前提で ↓ が効く

[sandbox_workspace_write]
network_access = false   # デフォルト値。閉じた状態を出発点にする
```

閉じたままを共通デフォルトにする。これが本チートシートの推奨です。

ただしその前に、「何が止まって、何が止まらないか」を正確に知っておいてください。ここは誤解されやすく、必要のない設定変更を招きがちなところです。

**閉じても、「調べ物」は止まりません。**

`network_access` が制御するのは、**Codex が起動するコマンドとその子プロセス**の通信です。公式の記述はこうです。

> Network access is controlled through destination rules that apply to scripts, programs, and subprocesses spawned by commands.

ですから、よく利用されるはずのWeb 検索（`web_search`、次節）はコマンドではなくツール経由の別の層なので、この設定の管轄外です。公式も「Web 検索ツールは、コマンドにネットワークを与えることなく制御できる」と明記しています。それでWeb検索をするからといってこの部分をtrueにしなければならないと考えないでください。

| 作業 | `network_access = false` で |
|---|---|
| Web 検索・調査・裏取り | **動く** |
| `npm install` / `pip install` | 承認を求められる |
| `git fetch` / `git push` | 承認を求められる |
| スクリプト内の `curl`、API 呼び出し | 承認を求められる |

もう一点。**止まるのではなく、承認を挟みます。** `approval_policy = "on-request"` と組み合わせた場合、通信を要するコマンドで Codex は承認を求めます。承認すれば実行されます。公式にもこう書かれています。

> Codex asks for approval to edit files outside the workspace or to run commands that require network access.

つまり `false` は「絶対の遮断」ではなく「**関門**」です。完全な遮断が必要なら、承認そのものを発生させない `approval_policy = "never"` と組み合わせてください。

なぜ「閉じた状態」を推すのかですが、それは外部コンテンツは間接的プロンプトインジェクション(indirect prompt injection) の主要な経路だからです。例えば、`npm install` で取得したパッケージの README、postinstall スクリプト、参照先ドキュメント、Issue テキスト——こうした場所に悪意ある指示が混ざっていた場合、ネットワークが開いているほど行動半径が広がります。data exfiltration（情報の持ち出し）の観点でも同様です。

デフォルトでそこを閉じておけば、人間の判断(Human-In-the-Loop)が一度入れられます。

そして調べ物が止まらない以上、**この関門が発生するのはパッケージ取得とリポジトリ操作の時だけ**です。頻度は限られ、しかも「いま依存を取りに行った」という文脈がはっきりしている。承認が意味を持つのは、まさにこういう時だけです。

必要な作業が決まっているなら、開ける側を profile ファイルとして用意しておくのが楽です（書き方は「日常の使い分け」を参照）:

```toml
# ~/.codex/remote_enabled.config.toml
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
network_access = true
writable_roots = []
```

**開けて使う場合に知っておくこと:** ネットワークが開いていれば、indirect prompt injection で誘導された操作はそのまま外部に到達します。意図せずトークンや内部情報が外部リクエストに乗ることもあります。承認も書き込み範囲の制限も、この経路の代わりにはなりません——`writable_roots` が絞るのは書き込み先であって、読み取れる範囲ではないからです。開けたうえで持ち出しを抑えたいなら、次のドメイン単位の制限が本筋になります。

**ドメイン単位で絞る（experimental）:**

`features.network_proxy` を有効にすると、外向き通信をプロキシ経由にして、ドメイン単位の allow / deny を書けます。現時点では experimental で、デフォルトは無効です。

実務上の要点はここです。**ドメインルールは承認を発生させません。** 許可した宛先は素通り、拒否した宛先はその場で失敗する。人間に尋ねるのではなく、ポリシーとして評価されます。

「通信のたびに尋ねられて、結局は中身を見ずに承認してしまう」を避けたいなら、選ぶべきはこちらです。

```toml
# ~/.codex/config.toml
[sandbox_workspace_write]
network_access = true          # プロキシは「開いた通信」を絞る仕組み

[features.network_proxy]
enabled = true

[features.network_proxy.domains]
"registry.npmjs.org" = "allow"
"**.github.com" = "allow"
"*" = "deny"
```

パターンは、完全一致のホスト名、`*.example.com`（サブドメインのみ）、`**.example.com`（apex ＋サブドメイン）、そして全体を指す `*` が使えます。**`deny` が `allow` に優先します。**

とはいえ、許可リスト(allowlist)を最初から完全に書ききるのは現実的ではありません。そこで `*` の出番です。逆向きから始められます。

```toml
[features.network_proxy.domains]
"*" = "allow"                          # 通常の作業は妨げない
"169.254.169.254" = "deny"             # クラウドのメタデータサービス
"metadata.google.internal" = "deny"
"**.internal.example.com" = "deny"     # 社内ネットワーク
```

行かせたくない先だけを塞ぎ、運用しながら締めていく形です。どちらの向きでも同じ機構で書けます。

`network_access` との関係は次のとおりです。

| `network_access` | `features.network_proxy` | 結果 |
|---|---|---|
| `false` | 有効 | 閉じたまま。プロキシは何もしない |
| `true` | 無効 | **無制限の直接外向き通信** |
| `true` | 有効 | 通信は開くが、ポリシーで制約される |

ドメイン制限は `network_access = true` を前提に絞り込む仕組みであって、`false` の代わりにはなりません。**遮断の強さでは `false` が上、承認の少なさではドメインルールが上。** 使い分けはこうなります。

| 構成 | 承認の発生 | 遮断 |
|---|---|---|
| `false` ＋ `on-request`（**本チートシートのデフォルト**） | npm / git のたびに出る | 承認すれば通る |
| `false` ＋ `never` | 出ない | 通らない（完全遮断） |
| `true` ＋ ドメインルール | 出ない | 許可した宛先だけ通る |

なお、これは experimental の機能で、デフォルトは無効です。安定性を要する環境では、まず `false` ＋ `on-request` で運用してください。

### 4. Web 検索と外部コンテンツ

前節のとおり、`network_access` を閉じても Web 検索は動きます。モデルが外部由来のテキストを読む経路は、コマンドの通信とは別に存在するということです。`web_search` は 4 段階です。

- `disabled`: 検索しない
- `cached` (default): OpenAI が保持する索引済みの結果を返す。ライブページを取りに行かない
- `indexed`: 外部アクセスを検索インデックス経由に限定する
- `live`: その場で実ページを取りに行く（`--search` と同じ）

**デフォルトの `cached` は、そのまま使ってよい設定です。** 事前に索引された結果を返すので、任意のライブページを読み込む場合に比べてプロンプトインジェクションの露出が小さくなります。公式も同じ説明をしています。調べ物ができることと露出の小ささが両立している、今のところの落としどころです。

意識すべきなのは次の 2 点です。

- **`live` は意図して選ぶもの。** その場で任意のページを読むので、`cached` の緩和は効きません
- **`--yolo` などフルアクセス系の設定を使うと、web 検索は自動的に `live` に変わります。** サンドボックスを緩めたつもりが、外部コンテンツの取り込みまで一段緩んでいる、という組み合わせ事故が起きます

```toml
# ~/.codex/config.toml
web_search = "cached"      # デフォルト。ふつうはこのまま
# web_search = "indexed"   # 外部アクセスを索引経由に絞りたいとき
# web_search = "disabled"  # 外部を一切読ませたくないとき
```

いずれの段階でも、検索結果は**信頼できない入力**として扱ってください。`cached` は露出を減らしますが、無くすわけではありません。

`features.web_search` / `features.web_search_cached` / `features.web_search_request` は旧式の切り替えで、非推奨です。トップレベルの `web_search` を使ってください。

### 5. シークレットと認証情報

> **この節だけは、設定を間違えても後から直せません。** サンドボックスや承認は、間違えたら設定を直せば済みます。漏れたシークレットは直せません。ローテーションするしかない。ここは他の節より慎重に読んでください。

ここは二つの別の話が混ざりやすいところです。「キーチェーンに入れたから安心」で止まりがちですが、それでは片方しか解決していません。分けて考えます。

**(1) Codex 自身の資格情報をどこに置くか**

Codex CLI のログイン情報は、**デフォルトではファイル（`auth.json`）に保存されます。** `keyring` を選べば OS キーチェーンへ寄せられます。MCP の OAuth 情報だけはデフォルトが `auto`（キーチェーンが使えればそちら、駄目ならファイル）で、CLI 側とは別のキーです。

```toml
# ~/.codex/config.toml
cli_auth_credentials_store = "keyring"    # file（デフォルト）| keyring | auto
mcp_oauth_credentials_store = "keyring"   # auto（デフォルト）| file | keyring
```

道具を増やさずに、平文でディスクに残る資格情報を一つ減らせます。個人の端末ならまずこれで十分です。ホームディレクトリをクラウド同期している場合はとくに効きます——平文の `auth.json` は、置いた瞬間に同期先へ複製されるからです。

**この行を書いただけでは移行しません。** 保存先が変わるだけなので、既存の `auth.json` はそのまま残り、キーチェーンには何も入っていない状態になります。再ログインまで済ませてください。

1. `cli_auth_credentials_store = "keyring"` を書く
2. `codex login` を実行する。ログインは開始時に既存の資格情報を先に消すので、**途中でやめると未ログイン状態になります。** 最後まで進めてください
3. `codex login status` で認証できることを確認する
4. `auth.json` が残っていないことを、値を表示せずに確認する

キーチェーンへの保存に成功した時点で `auth.json` は削除されますが、削除に失敗しても警告だけでログインは成功します（0.146.0 で確認）。だから 4 の確認は自分で行ってください。

**(2) あなたの API キーをどう渡すか**

こちらはキーチェーンだけでは片付きません。原則は一つです。

> **`config.toml` に値を書かない。**

例外はありません。「あとで消すから」「ローカルだけだから」で書いた行が、そのまま同期・バックアップ・スクリーンショット・画面共有に乗ります。

Codex 側には、値ではなく「どの環境変数から取るか」を書く受け口が用意されています。

```toml
# ~/.codex/config.toml
[mcp_servers.example]
url = "https://mcp.example.com"
bearer_token_env_var = "EXAMPLE_MCP_TOKEN"   # 値ではなく変数名を書く

[mcp_servers.example.env_http_headers]
X-Api-Key = "EXAMPLE_API_KEY"                # ヘッダの中身も環境変数から
```

あとは、その環境変数を**起動時にだけ**注入します。対話シェルへ `export` して放置したり、`.env` に書き戻したりすると、せっかくの間接参照が台無しになります。

| 手段 | 向いている場面 |
|---|---|
| OS キーチェーン（macOS `security`、Linux libsecret、Windows 資格情報マネージャー） | 個人の端末。追加の契約も常駐プロセスも不要で、まずここから |
| [1Password CLI](https://developer.1password.com/docs/cli/)（`op run` / `op read`） | 個人〜小チーム。設定ファイルに「参照」を書ける |
| [Bitwarden Secrets Manager](https://bitwarden.com/products/secrets-manager/)（`bws run`） | チーム共有。マシンアカウント単位でスコープを絞れる |
| [HashiCorp Vault](https://developer.hashicorp.com/vault) | 動的な資格情報や短命トークンが要る規模 |
| AWS Secrets Manager / Google Secret Manager / Azure Key Vault | すでにそのクラウドで IAM を運用している場合 |
| [`pass`](https://www.passwordstore.org/) / [`gopass`](https://www.gopass.pw/) | GPG 一本で完結させたい場合 |
| [SOPS](https://github.com/getsops/sops) + [age](https://github.com/FiloSottile/age) | 設定ごとリポジトリに置きたいが、平文は避けたい場合 |

どれを選んでも、狙いは同じです。**`config.toml` にもシェル履歴にも平文の値を残さず、必要なときにだけ取り出す。** どの道具も値そのものはどこかに保管しますが（キーチェーンは OS の保管庫に、SOPS は暗号化してリポジトリに）、平文が設定ファイルに転記され、そのまま同期やバックアップに乗っていく事態は避けられます。

**(3) 環境変数がそのまま子プロセスへ流れないようにする**

Codex はシェルコマンドを子プロセスとして起動します。あなたの環境変数がどこまで引き継がれるかは `shell_environment_policy` で決まります。

**このデフォルトは安全側ではありません。** 0.146.0 では `inherit` のデフォルトが `all`、`ignore_default_excludes` のデフォルトが `true`（＝名前による除外を適用しない）です。つまり何も書かなければ、あなたの環境変数はすべて子プロセスへ渡ります。

```toml
# ~/.codex/config.toml
allow_login_shell = false             # 下記の再注入を止める

[shell_environment_policy]
inherit = "core"                      # all（デフォルト）| core | none
ignore_default_excludes = false       # デフォルトは true ＝除外なし
```

`inherit = "core"` は、渡す変数を `PATH` / `HOME` / `SHELL` / `TMPDIR` などの固定リストに絞ります。`ignore_default_excludes = false` は名前による除外を有効にしますが、**対象は `*KEY*` / `*SECRET*` / `*TOKEN*` の 3 パターンだけ**です。`PGPASSWORD` や `DATABASE_URL` は素通りするので、これを万能の保護と考えないでください。`core` にしている限りこの行は効きませんが、あとで `inherit` を緩めたときに残る保険として書いておきます。

**`allow_login_shell = false` が要るのは、迂回路があるからです。** login shell が許可されていると、Codex はコマンドを login shell として起動し、その前にシェル環境のスナップショットを読み込みます。このスナップショットは Codex 自身の環境を丸ごと写したもので、`shell_environment_policy` のフィルタが掛かりません（0.146.0 で確認）。`inherit = "core"` で絞っても、この経路で元に戻ります。`false` にすると login shell にならないので、読み込み自体が起きません。

スナップショットのファイルは `$CODEX_HOME/shell_snapshots/` に平文で作られ、正常に終了すれば消えます。クラッシュや強制終了では残るので、機微な環境で異常終了したあとは中身を確認してください。生成そのものを止めるなら `[features] shell_snapshot = false` を指定します（0.146.0 で存在を確認したキーです。シェル関数やエイリアスの引き継ぎを失う代償があります）。

**そして、この設定で塞げない経路が二つあります。** hooks と `notify` は、`shell_environment_policy` の外で、Codex の環境をそのまま受け取って実行されます（0.146.0 で確認）。

だから、いちばん確実なのは設定ではなく運び方です。**Codex を起動するシェルに、不要なシークレットを `export` しない。** MCP など Codex 自身が読む必要のある変数だけを、起動時に注入してください（§5 (2) の受け口）。継承を広く保つ必要がある場合だけ、`filters` で名前を落とします。

```toml
[shell_environment_policy.filters]
"*PASSWORD*" = "exclude"
"*CREDENTIAL*" = "exclude"
```

`filters` は旧来の `exclude` / `include_only` と併記できません（併記するとエラーになります）。パターンは正規表現ではなく `*` / `?` のワイルドカードで、大文字小文字を区別しません。`include` を一つでも書くと許可リストとして動き、`exclude` で落ちた変数は復活しません。

### 6. MCP サーバ

MCP サーバは、Codex にできることを外から増やす仕組みです。便利ですが、増えるのは能力だけではありません。攻撃面(attack surface)も一緒に増えます。

見落とされがちなのは、**サーバが返してくるテキストがモデルへの入力になる**という点です。§4 の外部コンテンツと同じ警戒が要ります。

```toml
# ~/.codex/config.toml
[mcp_servers.example]
command = "example-mcp-server"
enabled_tools = ["search", "fetch"]   # 使うものだけを許可
disabled_tools = ["delete"]           # enabled_tools の後に適用される拒否リスト
default_tools_approval_mode = "prompt"
```

- `enabled_tools` で許可リストを作り、`disabled_tools` で個別に落とすのが基本形です。サーバが提供するツールを全部有効にする必要はありません
- ツール単位でも `[mcp_servers.<id>.tools.<ツール名>]` の `approval_mode = "prompt"` で承認を要求できます
- `approval_policy` を `granular` にしている場合、`mcp_elicitations = false` にすると MCP からの問い合わせは自動的に拒否されます
- `enabled = false` にすれば、設定を消さずにサーバだけ止められます
- 認証情報は §5 のとおり、`bearer_token_env_var` などで環境変数から取ります

### 7. 履歴保存の制限

履歴保存は便利です。ただ、何が残るのかは意識しておいてください。

端末に残るのは、あなたが入力したプロンプト、AI の応答、実行されたコマンドとその結果。つまり、作業中に触れた情報がそのまま残ります。API キーの断片、接続先のホスト名、障害調査のメモ、内部 URL、顧客固有の識別子。後から見返すと、思っていたより多くのことが書かれているものです。

**残り方は二系統に分かれています。** この節で扱う入力履歴（`history.jsonl`）と、§8 で扱うセッション全文（`sessions/`）です。設定で止められるのは前者だけなので、両方を読んでください。

```toml
# ~/.codex/config.toml
[history]
persistence = "save-all"   # デフォルト
```

- 日常の振り返りや継続作業には履歴保存が有用
- 機密性を優先したい場合は `persistence = "none"` に切り替える
- **ただし `persistence` が止めるのは入力履歴（`history.jsonl`）だけです。** セッション全文は別の仕組みで `~/.codex/sessions/` に保存され、この設定では止まりません。§8 を必ず読んでください
- `max_bytes` を指定すると、履歴ファイルの上限を決めて古いものから落とせます
- 共有端末や業務環境では、誰がその履歴を読めるかを確認しておくこと
- agent 的な運用（自動実行の繰り返し）では、履歴の蓄積自体が情報漏えい面になりうる

**注意:** `save-all` を無条件で正当化しないこと。機密性の高い環境では `none` の方が適切なことがあります。共有端末や退職者のアカウント引き継ぎ時に、想像以上の情報が読めてしまうことがあります。

**メモリ（Memories）:**

セッションを跨いで内容を持ち越す Memories は、機能ゲート（`features.memories`）がデフォルトで無効です。ただし**有効にした時点で、内側の利用・生成はどちらもデフォルトでオン**になります。つまり「使う／使わない」は一度の判断で決まり、細かい調整はその後です。有効にすると、履歴とは別の残留面が増えます。

```toml
# ~/.codex/config.toml
[memories]
use_memories = false                  # 既存メモリを新しいセッションへ注入しない
generate_memories = false             # 新しいスレッドをメモリ生成の材料にしない
disable_on_external_context = true    # MCP・Web 検索など外部由来を含むスレッドを材料から外す
```

`disable_on_external_context` は、外から来たテキストが長期記憶に定着する経路を断つという意味で、情報残留だけでなく injection 対策としても効きます。Memories を使うなら、ここは有効にしておくのが無難です。

### 8. ログ記録

Codex CLI は、公式ドキュメントで目立つ扱いではないものの、本格的な監査・テレメトリ基盤を持っています。

**セッション rollout（自動保存）:**

- `~/.codex/sessions/` 以下に JSONL 形式で自動保存される
- `history.persistence` とは独立した仕組みで、セッション全体（ツール実行結果を含む）が記録される
- 振り返りやデバッグに有用
- **止める設定はありません。** 対話 CLI の `config.toml` には、この保存を無効にするキーも、容量上限も、期間による自動削除もありません（0.146.0 で確認）。7 日を過ぎたものは `.jsonl.zst` に圧縮されますが、圧縮は削除ではありません

つまり `[history] persistence = "none"` にしても、作業内容そのものは残り続けます。機密性を優先する場面では、次のどちらかで対処してください。

```bash
# 非対話で済む作業は、そもそも保存させない
codex exec --ephemeral "..."

# 対話で作業したあとは、そのセッションを削除する
codex delete <セッション ID>
```

`codex archive` は `archived_sessions/` へ移すだけで、削除ではありません。機微な作業を続ける端末では、`sessions/` と `archived_sessions/` を定期的に棚卸ししてください。

**OpenTelemetry 統合:**

`config.toml` の `[otel]` セクションで、ログ・トレース・メトリクスを外部の OTLP コレクター（Jaeger、Grafana、Datadog 等）へ送信できます。

```toml
# ~/.codex/config.toml
[otel]
environment = "dev"

[otel.trace_exporter.otlp-http]
endpoint = "https://otlp.example.com"
protocol = "binary"        # 必須。"binary" または "json"
```

`protocol` は必須のキーです（`binary` または `json`）。

記録される主なイベント:

- `codex.tool_decision` — ツールの承認/拒否と、その判断元（config / user / automated_reviewer）
- `codex.tool_result` — ツール実行結果（成功/失敗、引数、出力）
- `codex.api_request` — API 呼び出しの記録

エンタープライズ環境での監査証跡や、hardening 設定自体のデバッグに使えます。

**アプリケーションログ:**

- `~/.codex/log/` にファイルログが書き出される
- `config.toml` の `log_dir` で変更可能

**利用データの送信:**

`[otel]` を自分で設定しなくても、利用メトリクスは OpenAI 側の収集先へ送られます。`analytics.enabled` は書かなければ有効として扱われ、メトリクスはログインしていなくても送信されます（0.146.0 の release ビルドで確認）。操作イベントの送信はログイン状態と認証方式によります。止めるなら次の 1 行です。

```toml
# ~/.codex/config.toml
[analytics]
enabled = false
```

メトリクス側もこの行に連動して止まります。将来その連動が変わっても閉じたままにしたい場合は、`[otel] metrics_exporter = "none"` も併記してください。

**`$CODEX_HOME` はまるごと機密ディレクトリです:**

ここまで見てきたとおり、`~/.codex` には資格情報、入力履歴、セッション全文、シェル環境のスナップショット、永続化した許可ルールが集まります。Codex はこれらのファイルを必ず厳しい権限で作るわけではなく（`auth.json` は 0600 ですが、セッションやスナップショットは作成時の umask 任せです）、ディレクトリ自体の権限も検査しません。入れ物の側で守るのが確実です。

```bash
chmod 700 ~/.codex
```

Unix 系での一般的な目安です。運用形態によって適否は変わるので、共有ホストでは所有者と権限を併せて確認してください。

### 9. 設定が効いているか確かめる

設定は、書いただけでは信用できません。一度、効いているかを確かめてください。

Codex CLI には、サンドボックスの中で任意のコマンドを試せるサブコマンドがあります。

```bash
codex sandbox -- curl -sS https://example.com

# macOS では、拒否された操作をあわせて表示できる
codex sandbox --log-denials -- curl -sS https://example.com
```

`--` の後ろが、サンドボックス内で実行されるコマンドです。`--log-denials`（macOS）を付けると、実行中に発生したサンドボックス拒否が終了後にまとめて表示されます。

`network_access = false` が本当に効いているか。`writable_roots` の外に書けないか。この形で一度確かめておく。**ハードニングで最も多い事故は「設定したつもり」です。**

profile を切り替えて確かめるなら `--profile <名前>`、beta の権限プロファイルを対象にするなら `-P` / `--permission-profile <名前>` を付けます。

ただし、このコマンドで確かめられる範囲には限りがあります。`codex sandbox` が適用するのはファイルシステムとネットワークのサンドボックスまでで、`shell_environment_policy` は適用されません。`MY_API_KEY` を export した状態で `codex sandbox -- env` を実行すると、その値はそのまま表示されます。環境変数の除外が効いているかは、この方法では確かめられません。

**設定キーそのものが有効かどうかも、別に確かめてください。** 綴りを間違えたキーや、その版に存在しないキーは、通常の起動では警告なく無視されます。書いたつもりの設定が一つも効いていなくても、Codex は普通に動きます。

```bash
codex exec --strict-config --skip-git-repo-check "ok"
```

`--strict-config` を付けると、`config.toml` 全体がスキーマと照合され、知らないキーがあればその場で行番号つきのエラーになります。版を上げたあとや、この文書を読ませて設定を書かせたあとに、一度通しておくと確実です。

## 導入の仕方

### Quick Start

以下を `~/.codex/config.toml` に置いてください。目指しているのは「便利だが安全」です。

ワークスペース内の編集も調べ物も普段どおりに進みます。人間の判断が入るのは、ワークスペースの外に出る操作と、外部への通信を伴う操作だけです。

```toml
# ~/.codex/config.toml
approval_policy = "on-request"
sandbox_mode = "workspace-write"
allow_login_shell = false
web_search = "cached"          # 索引済みの結果を返すので live より露出が小さい

# 資格情報はファイル（平文の auth.json）ではなく OS キーチェーンへ
# 書いただけでは移行しません。再ログインまで済ませてください（§5）
# API キーは絶対にこのファイルに書かない（§5 を参照）
cli_auth_credentials_store = "keyring"
mcp_oauth_credentials_store = "keyring"

# モデル名はここで固定しない方が無難です。世代が上がるたびに古くなります

[analytics]
enabled = false                # 書かないと利用メトリクスが送信されます（§8）

[history]
persistence = "save-all"       # 機微な作業では "none"。ただしセッション全文は別に残ります（§8）

[sandbox_workspace_write]
network_access = false         # Web 検索は止まらない。npm / git で承認を挟む
exclude_slash_tmp = true
exclude_tmpdir_env_var = true
writable_roots = []            # 追加の書き込み先を増やさない

[shell_environment_policy]
inherit = "core"               # デフォルトは all（全部渡す）
ignore_default_excludes = false  # デフォルトは true（＝名前による除外なし）
```

**profile は 1 つにつき 1 ファイルです**（詳しくは「日常の使い分け」）。切り替え用に次を置いておきます。

```toml
# ~/.codex/readonly_quiet.config.toml — 調査だけしたいとき
approval_policy = "never"
sandbox_mode = "read-only"
```

```toml
# ~/.codex/local_write.config.toml — 通常のローカル編集
approval_policy = "on-request"
sandbox_mode = "workspace-write"
```

```toml
# ~/.codex/remote_enabled.config.toml — ネットワークが必要な時だけ
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
network_access = true
exclude_slash_tmp = true
exclude_tmpdir_env_var = true
writable_roots = []
```

```toml
# ~/.codex/offline_strict.config.toml — 機微な作業用。外に出さない
approval_policy = "untrusted"
sandbox_mode = "workspace-write"
web_search = "disabled"

[sandbox_workspace_write]
network_access = false
exclude_slash_tmp = true
exclude_tmpdir_env_var = true
writable_roots = []

[history]
# 入力履歴（history.jsonl）だけを止めます。セッション全文は sessions/ に残ります
persistence = "none"
```

**この profile の限界:** ネットワークと入力履歴は閉じますが、セッション全文の保存までは止まりません。「端末に何も残らない profile」ではないので、機微な作業を終えたら `codex delete <セッション ID>` で当該セッションを削除し、`sessions/` と `archived_sessions/` の残りを確認してください。非対話で済む作業なら `codex exec --ephemeral` を選ぶほうが確実です（§8）。

この構成の要は 2 点です。書き込みがワークスペース内に限られること。外に出る操作で人間に来ること。中では自動で進み、外に出るとき——ファイルでもネットワークでも——に判断が挟まります。普段は静かで、来た時には読む価値がある。狙っているのはその塩梅です。

ただし過信は禁物です。この構成でも、**ワークスペース内の破壊的な操作は承認なしに進みます**。`workspace-write` とは、そこを自動で通すための設定だからです。git で戻せる状態にしておくことが、設定と同じくらい効きます。

### 厳格に締めたいとき — 触る場所

用途によっては、上のデフォルトより締める必要が出てきます。顧客データを扱う、機微なコードを読ませる、法規制のある環境で使う——そうした場面です。

**どこを触るかは決まっています。** 次の表から必要なものを選んでください。ただし代償の欄も併せて見ること。何も失わずに締まる設定はありません。

| 締めたいもの | 設定 | 代償 |
|---|---|---|
| サンドボックス内のコマンドの外向き通信を断つ | `network_access = false` ＋ `approval_policy = "never"` | 承認による例外も効かなくなる。npm / git は失敗する。Web 検索・MCP・hooks は別に締める必要がある |
| 承認を出さずに行き先だけ絞る | ドメインルール（`features.network_proxy`） | experimental。設定量が増える |
| 外部テキストの取り込み | `web_search = "indexed"` または `"disabled"` | 調査の質が落ちる |
| ワークスペース内の操作も含めて止める | `approval_policy = "untrusted"` | 承認が頻発する。日常運用には重い |
| 特定カテゴリの操作 | `approval_policy` を granular にして該当を `false` | 静かに失敗が増える。何を閉じたか記録が要る |
| 破壊的コマンド | execpolicy の `forbidden` ルール | プレビュー機能。ルールの保守が要る |
| 環境変数からの漏れ | `inherit = "core"` ＋ `allow_login_shell = false`。継承を広く保つなら `filters` で許可リスト化 | 通す変数を把握しておく必要がある。login shell の PATH やエイリアスに頼れなくなる |
| 入力履歴の残留 | `persistence = "none"` | 継続作業がしづらくなる。セッション全文は残る |
| セッション全文の残留 | 非対話は `codex exec --ephemeral`、対話は作業後に `codex delete <ID>` | `codex resume` で再開できなくなる。設定では止められないので運用が要る |
| メモリの残留 | `use_memories = false` / `generate_memories = false` | セッションを跨ぐ利点が消える |
| MCP 経由の操作 | `enabled_tools` で許可リスト化 | サーバ更新のたびに見直しが要る |
| 組織全体で強制 | `requirements.toml` | 配布と維持の運用が要る |

これらを全部入れる必要はありません。むしろ全部入れた構成は、使われなくなるか、承認が形骸化するかのどちらかに終わります。**締めすぎた設定は、たいてい誰かに外されます。**

守りたいものを一つ決めて、そこに効く行だけを足す。順序はこれです。

一時的に厳格側へ寄せたいだけなら、profile の切り替えで足ります。

```bash
codex --profile offline_strict
```

### プロジェクト固有の調整

- `.codex/config.toml`（プロジェクトルート）に、必要な追加設定だけを書く
- 例: 特定リポジトリだけ追加の `writable_roots` が必要な場合

### 日常の使い分け

profile は **1 つにつき 1 ファイル**です。`$CODEX_HOME/<名前>.config.toml`（デフォルトでは `~/.codex/<名前>.config.toml`）を置き、`--profile <名前>` で選びます。ベースの `config.toml` の上に、そのファイルが重なります。

profile ファイルは**設定ファイルそのもの**なので、`[sandbox_workspace_write]` や `[history]` を含め、`config.toml` に書けるものはひととおり書けます。ネットワークや履歴を profile 単位で切り替えられるのはこのためです。

```bash
# 調査だけしたい
codex --profile readonly_quiet

# 通常のローカル編集
codex --profile local_write

# ネットワークが必要な時だけ
codex --profile remote_enabled

# 機微な作業。ネットワークと入力履歴を閉じる（セッション全文は別途削除が要る）
codex --profile offline_strict
```

なお公式ドキュメントのサンプルには `full_auto` という名前のファイルが登場しますが、このチートシートの立場は変わりません——実態が「承認あり」なら、その名前は付けないでください。

古い設定から移す場合は、`config.toml` に `[profiles.<名前>]` や `profile = "<名前>"` が残っていないか見てください。中身を `<名前>.config.toml` に移せば、そのまま同じ `--profile <名前>` で使えます。

### 一時的な例外

profile を作るほどでもない場合は、起動時のオプションで補えます。

```bash
# 追加ディレクトリだけ書けるようにする
codex --add-dir /path/to/output

# その起動だけネットワークを開ける
codex --config sandbox_workspace_write.network_access=true

# その起動だけ Web 検索を止める
codex --config 'web_search="disabled"'
```

### profile 運用上の注意

- profile は「その起動時の実行姿勢」を決めるものとして扱い、必要なものを毎回明示する方が安全です
- 権限は「上げた時」より「戻し忘れた時」に事故になります。広い方の profile を使った次の作業は、意識して戻してください
- CI や自動実行では profile を明示的に固定し、暗黙のデフォルトに頼らない方が事故を減らせます
- 人間向けの運用でも、「通常はベース設定、機微な作業は `offline_strict`」のように明文化しておくと迷いにくくなります
- profile ファイルが増えてきたら、名前だけで用途と危険度が伝わるかを定期的に見直してください

## 新しい権限プロファイル（beta）

Codex には、`sandbox_mode` / `[sandbox_workspace_write]` とは別系統の権限モデルが入りました。ファイルシステムとネットワークの境界を、名前付きプロファイルとして定義できます。

将来はこちらが主流になるかもしれません。ただ、飛びつく前に注意点が二つあります。

1. **beta です。** 公式が「under active development and may change」と明記しています
2. **併用できません。** `default_permissions` と `[permissions.*]` を使うか、`sandbox_mode` / `[sandbox_workspace_write]` を使うか、どちらか一方です

> **注意 — 公式ドキュメントの説明と、実機の挙動が食い違っています（0.146.0 で確認）。**
>
> [公式ドキュメント](https://developers.openai.com/codex/permissions)にはこう書かれています。
>
> > If `sandbox_mode` appears in any loaded config file, you pass `--sandbox`, or the selected config profile sets `sandbox_mode`, Codex uses those older sandbox settings instead of `default_permissions`.
>
> ところが実際は**逆で、`default_permissions` が勝ちます**。
>
> - `sandbox_mode = "workspace-write"` ＋ `default_permissions = ":read-only"` → 書き込みは**拒否された**
> - `sandbox_mode = "read-only"` ＋ `default_permissions = ":workspace"` → **書き込めてしまった**
>
> 危ないのは後者です。安全のために `read-only` にしたつもりでも、どこかの層に `default_permissions` が 1 行残っていれば、警告もなく `workspace-write` で動きます。**beta を試したあとは、`default_permissions` と `[permissions.*]` を消したことを確認してください。**
>
> この食い違いは上流に報告済みです（[openai/codex#36448](https://github.com/openai/codex/issues/36448)）。コードとドキュメントのどちらが意図された挙動かは、そちらの回答待ちです。

このチートシートがここまで説明してきた設定は、今も現役の本流です。ただし「beta に触れなければ関係ない」とは言い切れません。上のとおり、消し忘れた 1 行がその本流を静かに上書きします。

組み込みプロファイルは `:read-only` / `:workspace` / `:danger-full-access` の 3 つ。独自に定義するときは `[permissions.<名前>]` を書きます。

```toml
# ~/.codex/config.toml
default_permissions = ":workspace"

[permissions.reviewed_fetch]
description = "レビュー用。読み取りと、限定したドメインへの通信だけ"
extends = ":read-only"

[permissions.reviewed_fetch.network]
enabled = true

[permissions.reviewed_fetch.network.domains]
"github.com" = "allow"
"registry.npmjs.org" = "allow"
```

- `domains` は **allow エントリが 1 つも無ければ全ドメインが拒否**され、`deny` は `allow` に優先します。何も書かなければ拒否から始まるので、許可リストとして素直に書けます
- ファイルシステム側はパスごとに `read` / `write` / `deny` を指定します
- `extends` で他のプロファイルを継承できます
- `dangerously_allow_non_loopback_proxy` / `dangerously_allow_all_unix_sockets` は名前のとおりの抜け道です。通常の開発では触らないでください

## 組織で配る（requirements.toml）

ここまでは、一人の端末の `config.toml` の話でした。組織で配るなら、ユーザーが上書きできない層が要ります。

それが `requirements.toml` です。管理者が「ここは動かせない」と決めた設定を強制できます。

```toml
# requirements.toml（管理者が配置する）
allowed_approval_policies = ["untrusted", "on-request", "granular"]
allowed_sandbox_modes = ["read-only", "workspace-write"]
allow_login_shell = false
allow_managed_hooks_only = true
```

- `allowed_*` は許可値の列挙です。省略したキーは制約されないままなので、締めたいものは明示します
- `allow_managed_hooks_only = true` にすると、ユーザー・プロジェクト・セッション・プラグイン由来のフックを読み込まず、管理側のフックだけを通します。リポジトリが持ち込むフックを実行させたくない場合の要です
- `[mcp_servers]` は許可リストとして書けます。サーバ名だけでなく **identity（起動コマンドと引数、または URL）まで一致**しないと有効になりません。名前を保ったまま中身を差し替える手口を防ぐ設計です
- `[experimental_network]` で、管理側からサンドボックスのネットワークポリシーを構成できます。`managed_allowed_domains_only = true` にすると、ユーザーが足した許可は無視されます
- ChatGPT Business / Enterprise では、クラウドから取得した requirements も適用されます
- 権限プロファイル（beta）を管理側で配るなら `allowed_permission_profiles` を使います。ただし全クライアントが **Codex 0.138.0 以降**である必要があり、移行期は `allowed_sandbox_modes` を暫定の互換制約として残せます

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
- [Permissions（beta の権限プロファイル）](https://developers.openai.com/codex/permissions)
- [Admin-enforced requirements（requirements.toml）](https://developers.openai.com/codex/enterprise/managed-configuration#admin-enforced-requirements-requirementstoml)

## 参考資料

- [OWASP GenAI Security Project](https://genai.owasp.org/)
- [OWASP Prompt Injection](https://owasp.org/www-community/attacks/PromptInjection)
- [OWASP LLM Prompt Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/LLM_Prompt_Injection_Prevention_Cheat_Sheet.html)
- [Claude Code Hardening Cheatsheet](https://github.com/okdt/claude-code-hardening-cheatsheet)
