# Changelog

## v1.3 — 2026-08-03

記述を公開ソース（タグ `rust-v0.146.0`）と実機の挙動で確かめ直した改訂。**公式ドキュメントとの一致は根拠にしていません。** 実装と食い違う箇所は、食い違いとして書いています。

### Web 検索の推奨を変更

- **開発でコードを触らせるなら `web_search = "indexed"` を勧めます。** デフォルトの `cached` はライブ取得をしないため調べ物の途中で行き止まりに当たり、作業中に `live` へ緩めることになります。Quick Start と `codex_config_min_safe_template.toml` も `indexed` に合わせました
- **`cached` はインジェクション対策ではありません。** ライブページを取りに行かないだけで、検索結果に仕込まれたテキストは索引経由でそのまま届きます。索引は検査ではありません
- **`indexed` は取得先を索引済み URL に限ります。** 内部の情報をパラメータに載せて外へ持ち出す形を妨げますが、塞ぎ切る設定ではありません。照合しているのは OpenAI 側で、パス・フラグメント・符号化・リダイレクトは未検証です。MCP・hooks・承認を通ったコマンドは、この設定の管轄外です
- 締める表から `indexed` を落としました。`cached` から `indexed` へ動かすのは締める操作ではありません
- **`network_access = false` と `web_search = "cached"` はどちらもデフォルトです。** 両方とも触っていない状態では、自分で取りに行く経路がありません。索引の更新の速さは公開されていないので、返ってきたものがどれだけ新しいかを見積もる手立てもありません

### デフォルト値の記述を訂正

- **環境変数のデフォルトは安全側ではありません。** `inherit` のデフォルトは `all`、`ignore_default_excludes` のデフォルトは `true`（＝名前による除外を適用しない）です。名前による除外の対象は `*KEY*` / `*SECRET*` / `*TOKEN*` の 3 パターンだけで、`PGPASSWORD` や `DATABASE_URL` は含まれません
- **CLI の資格情報ストアのデフォルトは `file` です。** `auto` がデフォルトなのは MCP OAuth 側です。設定を書くだけでは移行しないので、再ログインまでの手順を載せました
- ユーザ向けの指示ファイルは `~/.codex/AGENTS.md`（一時的な上書きは `AGENTS.override.md`）です
- `default_permissions` と `sandbox_mode` の食い違いを、上流 issue [openai/codex#36448](https://github.com/openai/codex/issues/36448) として報告しました

### 設定の効く範囲を訂正

- **`network_access` が制御するのは、サンドボックス内のコマンドとその子プロセスです。** Web 検索・MCP・hooks は別の経路なので、別に締める必要があります
- **`history.persistence = "none"` はセッション全文を止めません。** 止まるのは `history.jsonl` だけで、作業内容は `sessions/` に残ります。対話 CLI に保存を止める設定・容量上限・期間による自動削除はありません。`codex exec --ephemeral` と `codex delete` を載せ、`offline_strict` プロファイルにこの限界を明記しました
- **`shell_environment_policy` には迂回路があります。** login shell が許可されていると、シェル環境のスナップショット経由で、フィルタの掛かっていない環境が戻ります。`allow_login_shell = false` はシークレット防御の設定として扱っています。スナップショットが `$CODEX_HOME/shell_snapshots/` に平文で作られること、hooks と `notify` はこのポリシーの外で動くことも書きました
- **`trust_level` は設定の優先権に関わります。** プロジェクト層はユーザ層より優先されるので、trusted なリポジトリの `.codex/config.toml` は手元の `sandbox_mode` や `approval_policy` を上書きできます

### 追加した節

- 承認ダイアログで永続化を選ぶと、コマンドの許可ルールとネットワークルールが同じ `rules/default.rules` に追記される点。監査プロンプトにも前後の差分確認を入れました
- `analytics.enabled` は書かなければ有効として扱われ、メトリクスの送信先には認証状態を参照する経路がありません
- `$CODEX_HOME` をまるごと機密ディレクトリとして扱う節。セッションやスナップショットは固定の厳しい権限では作られません
- 設定が効いているかを確かめる節。`codex exec --strict-config` を使わないと、知らないキーは警告なく無視されます。この確認自体がセッション記録を残さないよう `--ephemeral` を併せています

### 検証方法の訂正

- **`codex sandbox` は `sandbox_mode` や `network_access` を反映しません。** `workspace-write` かつ `network_access = true` を書いた設定でも、書き込みと外向き通信のどちらも拒否されました。同じ操作はサンドボックスを通さなければ成功します。設定が何であれ拒否されるので、ここで拒否されたことを「設定が効いている」と読むことはできません。反映されるのは `-P` で権限プロファイルを指定したときで、`-P :workspace` なら書き込みが通り、`-P :read-only` では拒否されます
- 「`network_access = false` が効いているかを `codex sandbox` で確かめる」という従来の案内を差し替えました
- 表とコード内コメントに残っていた「完全遮断」「外部を一切読ませない」「外に出さない」を、実際に閉じる範囲へ限定しました
- 監査プロンプトを日英で同梱しました（`Codex_CLI_Hardening_Audit_Prompt.{ja,en}.md`）

### リポジトリの扱い

- `AGENTS.md` / `CLAUDE.md` / `.claude/` / `.codex/` を `.gitignore` に加えました。これらは clone した読者のセッションに自動で読み込まれ、リポジトリを trust すればプロジェクト設定が読者自身の設定より優先されます

---

## v1.2 — 2026-08-01

Codex CLI 0.146.0 に合わせた改訂。記述は公式ドキュメントと実機の双方で確認しています。

### 設定方法の変更に追従

- **profile は 1 つにつき 1 ファイル**（`$CODEX_HOME/<名前>.config.toml`）になりました。profile の名前と役割は v1.0 のものを引き継いでいます。`config.toml` の中に定義が残っている場合は、そちらへ移します
- profile ファイルは設定ファイルそのものなので、`[sandbox_workspace_write]` や `[history]` も指定できます
- `approval_policy` に `granular` が加わりました（テーブル形式）。`on-failure` は非推奨です
- `codex sandbox` の呼び出し方を実機に合わせました
- OpenTelemetry の例に、必須キーの `protocol` を追記しました

### 挙動の説明を更新

- **`network_access = false` は関門です。** `on-request` と組み合わせた場合、通信を要するコマンドは承認を求め、承認すれば通ります。承認ごと止めるには `never` と組み合わせます
- **閉じても Web 検索は動きます。** `network_access` が制御するのはコマンドとその子プロセスの通信で、Web 検索は別の層を通ります
- **`default_permissions` は `sandbox_mode` より優先されます。** 公式ドキュメントは逆に説明していますが、0.146.0 の実機では `default_permissions` が勝ちました。beta の権限プロファイルを試したあとは、消し忘れにご注意ください
- `trust_level` は承認の省略だけでなく、プロジェクト配下の `.codex/` 層（config・hooks・rules）の読み込みも左右します

### 扱う範囲を拡張

- **シークレットと認証情報**を独立した軸に。OS キーチェーン、環境変数経由の受け渡し、`shell_environment_policy`
- Web 検索の 4 段階（デフォルトは `cached`）
- MCP サーバのツール許可リストと承認
- Memories による情報の残留
- ドメイン単位のネットワーク制限（experimental）。承認を発生させずに行き先を絞れます
- 組織配布（`requirements.toml`）と、beta の権限プロファイル
- 設定が効いているかを確かめる節

---

## v1.0 — 2026-04-02

初版。チートシート（日英）、設定テンプレート 2 種、`AGENTS.md`。

---

# Changelog (English)

## v1.3 — 2026-08-03

A revision checked against the published source (tag `rust-v0.146.0`) and the running binary. **Agreement with the official documentation was not used as the standard.** Where the implementation and the documentation disagree, the disagreement is stated.

### The web search recommendation changed

- **For development work, `web_search = "indexed"` is now the recommendation.** The default `cached` fetches no pages, so a lookup dead-ends partway and you reach for `live` in the middle of the job. The Quick Start and `codex_config_min_safe_template.toml` move to `indexed` as well.
- **`cached` is not a prompt-injection control.** It only declines to fetch live pages; text planted in the search results still arrives through the index, and an index is not an inspection.
- **`indexed` limits fetches to URLs already in the index.** That gets in the way of carrying internal data out in a parameter, but it does not close exfiltration. The matching happens at OpenAI's end, and paths, fragments, encodings and redirects are unexamined. MCP, hooks and approved commands sit outside the setting entirely.
- `indexed` was removed from the tightening table. Moving there from `cached` is not a tightening step.
- **`network_access = false` and `web_search = "cached"` are both defaults.** With neither touched there is no route for fetching anything yourself, and since the index refresh rate is not published, there is no way to judge how current an answer is.

### Corrected default values

- **The environment-variable defaults are not the safe ones.** `inherit` defaults to `all` and `ignore_default_excludes` defaults to `true`, meaning no name-based filtering. The name filter, when enabled, covers `*KEY*`, `*SECRET*`, and `*TOKEN*` only; `PGPASSWORD` and `DATABASE_URL` are not among them.
- **The CLI credentials store defaults to `file`.** `auto` is the default for MCP OAuth. Setting the key does not migrate anything, so the steps through to a fresh login are given.
- The user-level instruction file is `~/.codex/AGENTS.md` (`AGENTS.override.md` for a temporary override).
- The disagreement between `default_permissions` and `sandbox_mode` was reported upstream as [openai/codex#36448](https://github.com/openai/codex/issues/36448).

### Corrected how far each setting reaches

- **`network_access` governs commands run inside the sandbox and their subprocesses.** Web search, MCP, and hooks travel by other routes and need closing separately.
- **`history.persistence = "none"` does not stop the session transcript.** It governs `history.jsonl`; the work itself stays in `sessions/`, which the interactive CLI cannot disable, cap, or expire. `codex exec --ephemeral` and `codex delete` are covered, and the `offline_strict` profile states the limit.
- **`shell_environment_policy` can be bypassed.** With login shells allowed, a snapshot of the unfiltered environment is sourced back in. `allow_login_shell = false` is presented as a secrets control, along with the plaintext snapshot under `$CODEX_HOME/shell_snapshots/` and the fact that hooks and `notify` run outside the policy.
- **`trust_level` decides configuration precedence.** Project config sits above user config, so a trusted repository's `.codex/config.toml` can override your `sandbox_mode` or `approval_policy`.

### Added sections

- The allow rules an approval dialog can append to `rules/default.rules`, covering both commands and network rules; the audit prompt now asks for a before-and-after diff
- `analytics.enabled` is treated as enabled when absent, and the metrics destination has no path that consults your login state
- A section on treating `$CODEX_HOME` as one confidential directory; sessions and snapshots are not created with a fixed restrictive mode
- A section on verifying that settings took effect, including `codex exec --strict-config`, since an unknown key is otherwise ignored without warning. `--ephemeral` is paired with it so the check itself leaves no session record

### Corrected the verification method

- **`codex sandbox` does not reflect `sandbox_mode` or `network_access`.** With a configuration setting `workspace-write` and `network_access = true`, both a write and an outbound request were still refused; the same operations succeed outside it. Because it refuses whatever the configuration says, a refusal there cannot be read as confirmation. What it does reflect is a permission profile passed with `-P`: `:workspace` let a write through, `:read-only` refused it.
- The former advice, to confirm `network_access = false` with `codex sandbox`, was replaced.
- Absolute phrasing left in tables and code comments — "nothing gets through", "nothing external read at all", "nothing goes out" — was narrowed to what each setting actually closes.
- The hardening audit prompt is now shipped in both languages (`Codex_CLI_Hardening_Audit_Prompt.{ja,en}.md`)

### Repository handling

- `AGENTS.md`, `CLAUDE.md`, `.claude/` and `.codex/` are now ignored. They load automatically into the session of anyone who clones the repository, and once the repository is trusted, project configuration takes precedence over the reader's own.

---

## v1.2 — 2026-08-01

A revision for Codex CLI 0.146.0. The text was checked against both the official documentation and the running binary.

### Following configuration changes

- **A profile is now one file each** (`$CODEX_HOME/<name>.config.toml`). The profile names and roles from v1.0 are carried over; move any definitions still sitting in `config.toml` across.
- A profile file is a configuration file in its own right, so it can also carry `[sandbox_workspace_write]` and `[history]`.
- `approval_policy` gained `granular` (written as a table). `on-failure` is deprecated.
- `codex sandbox` invocation now matches the binary.
- The OpenTelemetry example now includes the required `protocol` key.

### Updated behaviour notes

- **`network_access = false` is a checkpoint.** With `on-request`, commands that need the network ask for approval and run once approved. Pair it with `never` to remove the approval path.
- **Web search keeps working when it is closed.** `network_access` governs command traffic; web search travels by another layer.
- **`default_permissions` takes precedence over `sandbox_mode`.** The official documentation describes the opposite, but on 0.146.0 `default_permissions` won. Worth remembering after experimenting with the beta permission profiles.
- `trust_level` decides not only whether approvals are skipped but whether a project's `.codex/` layer (config, hooks, rules) is loaded.

### Wider coverage

- **Secrets and credentials** as a first-class axis: OS keychain, environment-variable indirection, `shell_environment_policy`
- The four web search modes (`cached` by default)
- MCP server tool allow lists and approvals
- Retention through Memories
- Domain-level network rules (experimental), which constrain destinations without raising approval prompts
- Organization rollout via `requirements.toml`, and the beta permission profiles
- A section on verifying that settings took effect

---

## v1.0 — 2026-04-02

Initial release: the bilingual cheat sheet, two configuration templates, and `AGENTS.md`.
