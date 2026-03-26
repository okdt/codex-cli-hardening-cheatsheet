# Codex CLI Hardening Cheatsheet

Codex CLI を安全寄りに運用するための、日本語チートシートと設定テンプレート集です。

English: [README.en.md](./README.en.md)

このリポジトリの目的は次の 2 点です。

- 一般的なハードニングのコツを、Codex CLI の日常運用に落とし込んで整理する
- `sandbox` / `approval` / `network` / `history` など、Codex CLI で実際に効く設定例をすぐ使える形で提供する

これは OpenAI 公式ドキュメントではありません。実運用前に、利用中の Codex CLI バージョンと公式情報を必ず確認してください。

## Included Files

- [Codex_CLI_Hardening_Cheat_Sheet.ja.md](./Codex_CLI_Hardening_Cheat_Sheet.ja.md)
  一般的なハードニングの考え方、Codex CLI の推奨設定、運用上の注意点をまとめた本体
- [Codex_CLI_Hardening_Cheat_Sheet.en.md](./Codex_CLI_Hardening_Cheat_Sheet.en.md)
  英語版
- [codex-config.hardened.template.toml](./codex-config.hardened.template.toml)
  コメント付きの `config.toml` テンプレート
- [codex_config_min_safe_template.toml](./codex_config_min_safe_template.toml)
  最小限の安全設定だけを抜き出した軽量テンプレート

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
- `readonly_quiet` / `local_write` / `net_enabled` をどう使い分けるべきか整理して
- このプロジェクトでは `network_access` を既定で無効にすべきか検討して
- チーム運用を前提に、共通テンプレートと例外ルールの分け方を提案して

## Scope

このリポジトリは、次のような観点を扱います。

- Codex CLI の `sandbox` 設定
- `approval_policy` の基本方針
- ネットワーク有効化の切り分け
- ローカル履歴保持のリスク
- 共通テンプレートとプロファイル運用

次のものは主目的ではありません。

- 企業固有の DLP / SIEM / EDR 設計
- OpenAI 公式仕様の代替
- すべての環境でそのまま使える万能設定

## Notes

- 設定キーや挙動は Codex CLI のバージョンによって変わる可能性があります
- 共通テンプレートは、まず単純で説明しやすいことを優先しています
- `granular` のような細分化設定は、運用要件が固まってから追加する方が安全です
- Cheatsheet 本体では、OWASP の GenAI / prompt injection 関連資料を、どの観点で読むと Codex CLI の設定判断に役立つか分かるように整理しています
- また、`Human-In-The-Loop`、最小権限の原則、多層防御といったセキュア設計の基本原則もあわせて解説しています

## References

- OpenAI Help: https://help.openai.com/en/articles/11369540/
- OpenAI Codex config example discussion: https://github.com/openai/codex/issues/2760
- Claude Code hardening cheatsheet by okdt: https://github.com/okdt/claude-code-hardening-cheatsheet/blob/main/Claude_Code_Hardening_Cheat_Sheet.ja.md

## Author

Riotaro OKADA

## Acknowledgements

このリポジトリの構成と公開方針は、okdt による Claude Code hardening cheatsheet を参考にしています。

## License

CC BY-SA 4.0. See [LICENSE](./LICENSE).
