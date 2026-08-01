# Codex CLI Hardening Cheatsheet

Codex CLI を安全寄りに運用するための、日本語チートシートと設定テンプレート集です。

English: [README.en.md](./README.en.md)

このリポジトリの目的は次の 2 点です。

- 一般的なハードニングのコツを、Codex CLI の日常運用に落とし込んで整理する
- `sandbox` / `approval` / `network` / `history` / シークレットの扱いなど、Codex CLI で実際に効く設定例をすぐ使える形で提供する

**とくにシークレットの扱いは、設定を間違えても後から直せない唯一の領域です。** API キーを `config.toml` に書かず、OS キーチェーンやシークレットマネージャから起動時に渡す方法を、チートシート §5 にまとめています。

これは OpenAI 公式ドキュメントではありません。実運用前に、利用中の Codex CLI バージョンと公式情報を必ず確認してください。

## Included Files

- [Codex_CLI_Hardening_Cheat_Sheet.ja.md](./Codex_CLI_Hardening_Cheat_Sheet.ja.md)
  一般的なハードニングの考え方、Codex CLI の推奨設定、運用上の注意点をまとめた本体
- [Codex_CLI_Hardening_Cheat_Sheet.en.md](./Codex_CLI_Hardening_Cheat_Sheet.en.md)
  英語版
- [Codex_CLI_Hardening_Audit_Prompt.ja.md](./Codex_CLI_Hardening_Audit_Prompt.ja.md)
  **Codex CLI に貼り付けて使う監査プロンプト。** チートシートを読ませ、承認を取りながら一つずつ設定を見直させる（[英語版](./Codex_CLI_Hardening_Audit_Prompt.en.md)）
- [codex-config.hardened.template.toml](./codex-config.hardened.template.toml)
  コメント付きの `config.toml` テンプレート
- [codex_config_min_safe_template.toml](./codex_config_min_safe_template.toml)
  最小限の安全設定だけを抜き出した軽量テンプレート
- [CHANGELOG.md](./CHANGELOG.md)
  版ごとの変更点。v1.0 から更新する場合は、profile の書き方が変わっている点を確認してください

## How To Use

このドキュメントは、まず安全な共通デフォルトを知りたい初学者から、
自分の利用実態やプロジェクトの目的に合わせて設定を調整したい上級者まで、
段階的に使えるように構成しています。

最初は推奨される共通設定とテンプレートを確認し、
その後で、あなたの Codex にこのドキュメントを読ませて、
自分の用途に照らしてどこを調整すべきか相談してみるとよいでしょう。

たとえば、次のように段階的に相談できます。

- まず何を変えれば安全性が上がるか、初学者向けに整理して
- 個人開発向けなら、どこまで安全側に倒すべきか提案して
- `readonly_quiet` / `local_write` / `remote_enabled` をどう使い分けるべきか整理して
- このプロジェクトでは `network_access` を既定で無効にすべきか検討して
- チーム運用を前提に、共通テンプレートと例外ルールの分け方を提案して

## Scope

このリポジトリは、次のような観点を扱います。

- Codex CLI の `sandbox` 設定
- `approval_policy` の基本方針（`granular` を含む）
- ネットワーク有効化の切り分けと、Web 検索による外部コンテンツの取り込み
- シークレットと認証情報の置き場（OS キーチェーン、環境変数経由の受け渡し）
- MCP サーバのツール許可リスト
- ローカル履歴・メモリ保持のリスク
- 共通テンプレートとプロファイル運用
- 組織配布（`requirements.toml`）と、beta の権限プロファイル

次のものは主目的ではありません。

- 企業固有の DLP / SIEM / EDR 設計
- OpenAI 公式仕様の代替
- すべての環境でそのまま使える万能設定

## Notes

- Cheatsheet 本体では、OWASP の GenAI / prompt injection 関連資料を、どの観点で読むと Codex CLI の設定判断に役立つか分かるように整理しています
- また、`Human-In-The-Loop`、最小権限の原則、多層防御といったセキュア設計の基本原則もあわせて解説しています
- 共通テンプレートは、まず単純で説明しやすいことを優先しています
- `granular` のような細分化設定は、運用要件が固まってから追加する方が安全です
- 設定キーや挙動は Codex CLI のバージョンによって変わります。本文は **Codex CLI 0.146.0（2026-08-01 時点）**の公式ドキュメントと実機の双方で確認しています。両者が食い違う箇所は本文に明記しました

## References

- Codex Configuration reference: https://developers.openai.com/codex/config-reference
- Codex Permissions（beta）: https://developers.openai.com/codex/permissions
- Codex Admin-enforced requirements: https://developers.openai.com/codex/enterprise/managed-configuration
- OpenAI Codex config example discussion: https://github.com/openai/codex/issues/2760
- Claude Code hardening cheatsheet by okdt: https://github.com/okdt/claude-code-hardening-cheatsheet

## Author

Riotaro OKADA

## Acknowledgements

このリポジトリの構成と公開方針は、okdt による Claude Code hardening cheatsheet を参考にしています。

## License

CC BY-SA 4.0. See [LICENSE](./LICENSE).
