# Codex CLI Hardening Cheatsheet

This repository provides a practical hardening cheatsheet and configuration templates for running Codex CLI with a security-conscious baseline.

It has two main goals:

- translate general hardening practices into day-to-day Codex CLI operations
- provide ready-to-use examples for `sandbox`, `approval`, `network`, and `history`

This is not official OpenAI documentation. Before applying any settings in production, verify them against the Codex CLI version you are actually using and the latest official references.

## Included Files

- [Codex_CLI_Hardening_Cheat_Sheet.ja.md](./Codex_CLI_Hardening_Cheat_Sheet.ja.md)
  Main Japanese cheatsheet covering hardening principles, recommended settings, and operational notes
- [Codex_CLI_Hardening_Cheat_Sheet.en.md](./Codex_CLI_Hardening_Cheat_Sheet.en.md)
  English version
- [codex-config.hardened.template.toml](./codex-config.hardened.template.toml)
  Commented `config.toml` template
- [codex_config_min_safe_template.toml](./codex_config_min_safe_template.toml)
  Smaller template with only the core safety-oriented settings

## How To Use

This repository is meant to work for a range of readers, from people who want a safer shared default to people who want to tune Codex CLI around a specific project or operational model.

Start by reviewing the baseline and templates. After that, it is often useful to feed the docs back into your own Codex setup and ask what should be adjusted for your actual workflow and risk profile.

For example:

- explain which changes would improve safety first for a beginner
- suggest how far to tighten the defaults for solo development
- explain how `readonly_quiet`, `local_write`, and `net_enabled` should be used in practice
- evaluate whether `network_access` should stay disabled by default for this project
- propose how to separate shared defaults from explicit exceptions for team use

## Scope

This repository focuses on:

- Codex CLI `sandbox` settings
- baseline `approval_policy` choices
- how and when to enable network access
- the tradeoffs around local history retention
- shared templates and profile-based workflows

It is not mainly about:

- organization-specific DLP / SIEM / EDR architecture
- replacing official OpenAI documentation
- providing a universal config that is automatically right for every environment

## Notes

- Config keys and behavior may change across Codex CLI versions
- The shared templates prioritize clarity and operational simplicity first
- Fine-grained approval setups should usually come after the team has a concrete need for them
- The main cheatsheet also explains how OWASP GenAI and prompt-injection references can inform Codex CLI hardening decisions
- It also covers secure design principles such as human-in-the-loop, least privilege, and defense in depth

## References

- OpenAI Help: https://help.openai.com/en/articles/11369540/
- OpenAI Codex config example discussion: https://github.com/openai/codex/issues/2760
- Claude Code hardening cheatsheet by okdt: https://github.com/okdt/claude-code-hardening-cheatsheet/blob/main/Claude_Code_Hardening_Cheat_Sheet.ja.md

## Author

Riotaro OKADA

## Acknowledgements

The structure and publishing direction of this repository were informed by okdt's Claude Code hardening cheatsheet.

## License

CC BY-SA 4.0. See [LICENSE](./LICENSE).
