# Changelog

## v1.2 — 2026-08-01（2026-08-03 更新）

Codex CLI 0.146.0 に合わせた改訂。記述は公式ドキュメント・公開ソース・実機の挙動を突き合わせて確認しています。

### 設定方法の変更に追従

- **profile は 1 つにつき 1 ファイル**（`$CODEX_HOME/<名前>.config.toml`）になりました。profile の名前と役割は v1.0 のものを引き継いでいます。`config.toml` の中に定義が残っている場合は、そちらへ移します
- profile ファイルは設定ファイルそのものなので、`[sandbox_workspace_write]` や `[history]` も指定できます
- `approval_policy` に `granular` が加わりました（テーブル形式）。`on-failure` は非推奨です
- `codex sandbox` の呼び出し方を実機に合わせました
- OpenTelemetry の例に、必須キーの `protocol` を追記しました

### デフォルト値の記述

- **環境変数のデフォルトは安全側ではありません。** `inherit` のデフォルトは `all`、`ignore_default_excludes` のデフォルトは `true`（＝名前による除外を適用しない）です。名前による除外の対象は `*KEY*` / `*SECRET*` / `*TOKEN*` の 3 パターンだけで、`PGPASSWORD` や `DATABASE_URL` は含まれません
- **CLI の資格情報ストアのデフォルトは `file` です。** `auto` がデフォルトなのは MCP OAuth 側です。設定を書くだけでは移行しないので、再ログインまでの手順を載せました
- ユーザ向けの指示ファイルは `~/.codex/AGENTS.md`（一時的な上書きは `AGENTS.override.md`）です
- **`default_permissions` は `sandbox_mode` より優先されます。** 公式ドキュメントは逆に説明していますが、0.146.0 の実機では `default_permissions` が勝ちました。beta の権限プロファイルを試したあとは、消し忘れにご注意ください（上流に報告済み: [openai/codex#36448](https://github.com/openai/codex/issues/36448)）

### 設定の効く範囲

- **`network_access = false` は関門です。** `on-request` と組み合わせた場合、通信を要するコマンドは承認を求め、承認すれば通ります。承認ごと止めるには `never` と組み合わせます。制御の対象はサンドボックス内のコマンドとその子プロセスで、Web 検索・MCP・hooks は別の経路です
- **閉じても Web 検索は動きます。** 調べ物を止めずにパッケージ取得だけを関門にできます
- **`history.persistence = "none"` はセッション全文を止めません。** 止まるのは `history.jsonl` だけで、作業内容は `sessions/` に残ります。対話 CLI に保存を止める設定・容量上限・期間による自動削除はありません。`codex exec --ephemeral` と `codex delete` を載せ、`offline_strict` プロファイルにこの限界を明記しました
- **`shell_environment_policy` には迂回路があります。** login shell が許可されていると、シェル環境のスナップショット経由で、フィルタの掛かっていない環境が戻ります。`allow_login_shell = false` はシークレット防御の設定として扱っています。スナップショットが `$CODEX_HOME/shell_snapshots/` に平文で作られること、hooks と `notify` はこのポリシーの外で動くことも書きました
- **`trust_level` は設定の優先権に関わります。** 承認の省略やプロジェクト配下の `.codex/` 層（config・hooks・rules）の読み込みだけでなく、プロジェクト層はユーザ層より優先されるので、trusted なリポジトリの `.codex/config.toml` は手元の `sandbox_mode` や `approval_policy` を上書きできます

### 扱う範囲を拡張

- **シークレットと認証情報**を独立した軸に。OS キーチェーン、環境変数経由の受け渡し、`shell_environment_policy`
- Web 検索の 4 段階（デフォルトは `cached`）
- MCP サーバのツール許可リストと承認
- Memories による情報の残留
- ドメイン単位のネットワーク制限（experimental）。承認を発生させずに行き先を絞れます
- 組織配布（`requirements.toml`）と、beta の権限プロファイル
- 承認ダイアログで永続化を選ぶと `rules/default.rules` に許可ルールが追記される点。監査プロンプトにも前後の差分確認を入れました
- `analytics.enabled` は書かなければ有効として扱われ、利用メトリクスは未ログインでも送信されます
- `$CODEX_HOME` をまるごと機密ディレクトリとして扱う節。セッションやスナップショットは固定の厳しい権限では作られません
- 設定が効いているかを確かめる節。`codex exec --strict-config` を使わないと、知らないキーは警告なく無視されます

---

## v1.0 — 2026-04-02

初版。チートシート（日英）、設定テンプレート 2 種、`AGENTS.md`。

---

# Changelog (English)

## v1.2 — 2026-08-01 (updated 2026-08-03)

A revision for Codex CLI 0.146.0. The text was checked against the official documentation, the published source, and the running binary.

### Following configuration changes

- **A profile is now one file each** (`$CODEX_HOME/<name>.config.toml`). The profile names and roles from v1.0 are carried over; move any definitions still sitting in `config.toml` across.
- A profile file is a configuration file in its own right, so it can also carry `[sandbox_workspace_write]` and `[history]`.
- `approval_policy` gained `granular` (written as a table). `on-failure` is deprecated.
- `codex sandbox` invocation now matches the binary.
- The OpenTelemetry example now includes the required `protocol` key.

### Default values

- **The environment-variable defaults are not the safe ones.** `inherit` defaults to `all` and `ignore_default_excludes` defaults to `true`, meaning no name-based filtering. The name filter, when enabled, covers `*KEY*`, `*SECRET*`, and `*TOKEN*` only; `PGPASSWORD` and `DATABASE_URL` are not among them.
- **The CLI credentials store defaults to `file`.** `auto` is the default for MCP OAuth. Setting the key does not migrate anything, so the steps through to a fresh login are given.
- The user-level instruction file is `~/.codex/AGENTS.md` (`AGENTS.override.md` for a temporary override).
- **`default_permissions` takes precedence over `sandbox_mode`.** The official documentation describes the opposite, but on 0.146.0 `default_permissions` won. Worth remembering after experimenting with the beta permission profiles. Reported upstream as [openai/codex#36448](https://github.com/openai/codex/issues/36448).

### How far each setting reaches

- **`network_access = false` is a checkpoint.** With `on-request`, commands that need the network ask for approval and run once approved. Pair it with `never` to remove the approval path. It governs commands run inside the sandbox and their subprocesses; web search, MCP, and hooks are separate paths.
- **Web search keeps working when it is closed.** Research continues while package fetches remain the thing that stops for a human.
- **`history.persistence = "none"` does not stop the session transcript.** It governs `history.jsonl`; the work itself stays in `sessions/`, which the interactive CLI cannot disable, cap, or expire. `codex exec --ephemeral` and `codex delete` are covered, and the `offline_strict` profile states the limit.
- **`shell_environment_policy` can be bypassed.** With login shells allowed, a snapshot of the unfiltered environment is sourced back in. `allow_login_shell = false` is presented as a secrets control, along with the plaintext snapshot under `$CODEX_HOME/shell_snapshots/` and the fact that hooks and `notify` run outside the policy.
- **`trust_level` decides configuration precedence.** Beyond skipped approvals and whether a project's `.codex/` layer (config, hooks, rules) is loaded, project config sits above user config, so a trusted repository's `.codex/config.toml` can override your `sandbox_mode` or `approval_policy`.

### Wider coverage

- **Secrets and credentials** as a first-class axis: OS keychain, environment-variable indirection, `shell_environment_policy`
- The four web search modes (`cached` by default)
- MCP server tool allow lists and approvals
- Retention through Memories
- Domain-level network rules (experimental), which constrain destinations without raising approval prompts
- Organization rollout via `requirements.toml`, and the beta permission profiles
- The allow rules an approval dialog can append to `rules/default.rules`; the audit prompt now asks for a before-and-after diff
- `analytics.enabled` is treated as enabled when absent, and usage metrics are sent even when logged out
- A section on treating `$CODEX_HOME` as one confidential directory; sessions and snapshots are not created with a fixed restrictive mode
- A section on verifying that settings took effect, including `codex exec --strict-config`, since an unknown key is otherwise ignored without warning

---

## v1.0 — 2026-04-02

Initial release: the bilingual cheat sheet, two configuration templates, and `AGENTS.md`.
