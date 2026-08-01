# Changelog

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
- Web 検索の 4 段階（既定は `cached`）
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

## v1.2 — 2026-08-01

A revision for Codex CLI 0.146.0. The text was checked against both the official documentation and the running binary.

### Following configuration changes

- **A profile is now one file each** (`$CODEX_HOME/<name>.config.toml`). The profile names and roles from v1.0 are carried over; move any definitions still sitting in `config.toml` across.
- A profile file is a configuration file in its own right, so it can also carry `[sandbox_workspace_write]` and `[history]`.
- `approval_policy` gained `granular` (written as a table). `on-failure` is deprecated.
- `codex sandbox` invocation now matches the binary.
- The OpenTelemetry example now includes the required `protocol` key.

### Updated descriptions of behavior

- **`network_access = false` is a checkpoint.** With `on-request`, commands that need the network ask for approval and run once approved. Pair it with `never` to remove the approval path.
- **Web search keeps working when it is closed.** `network_access` governs commands and their subprocesses; web search goes through a separate layer.
- **`default_permissions` takes precedence over `sandbox_mode`.** The official documentation describes the opposite, but on 0.146.0 `default_permissions` won. Worth remembering after experimenting with the beta permission profiles.
- `trust_level` affects more than skipped approvals: it also decides whether a project's `.codex/` layer (config, hooks, rules) is loaded.

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
