# Codex CLI Hardening Cheatsheet

This repository provides a practical hardening cheatsheet and configuration templates for running Codex CLI with a security-conscious baseline.

It has two main goals:

- translate general hardening practices into day-to-day Codex CLI operations
- provide ready-to-use examples for `sandbox`, `approval`, `network`, `history`, and secrets handling

**Secrets handling in particular is the one area where a mistake cannot be undone by fixing a setting.** Section 5 of the cheat sheet covers how to keep API keys out of `config.toml` and pass them at launch from the OS keychain or a secrets manager.

This is not official OpenAI documentation. Before applying any settings in production, verify them against the Codex CLI version you are actually using and the latest official references.

## Included Files

- [Codex_CLI_Hardening_Cheat_Sheet.ja.md](./Codex_CLI_Hardening_Cheat_Sheet.ja.md)
  Main Japanese cheat sheet covering hardening principles, recommended settings, and operational notes
- [Codex_CLI_Hardening_Cheat_Sheet.en.md](./Codex_CLI_Hardening_Cheat_Sheet.en.md)
  English version
- [codex-config.hardened.template.toml](./codex-config.hardened.template.toml)
  Commented `config.toml` template
- [codex_config_min_safe_template.toml](./codex_config_min_safe_template.toml)
  Smaller template with only the core safety-oriented settings
- [CHANGELOG.md](./CHANGELOG.md)
  Changes per release. If you are updating from v1.0, note that the profile format has changed.

## How To Use

This repository is meant to work for a range of readers, from those who want a safer shared default to those who want to tune Codex CLI for a specific project or operational model.

Start with the cheatsheet and the templates. After that, it is often useful to feed the docs back into your own Codex setup and ask what should be adjusted for your actual workflow and risk profile.

For example:

- explain which changes would improve safety first for a beginner
- suggest how far to tighten the defaults for solo development
- explain how `readonly_quiet`, `local_write`, and `remote_enabled` should be used in practice
- evaluate whether `network_access` should stay disabled by default for this project
- propose how to separate shared defaults from explicit exceptions for team use

## Scope

This repository focuses on:

- Codex CLI `sandbox` settings
- baseline `approval_policy` choices, including `granular`
- how and when to enable network access, and how web search pulls in external content
- where secrets and credentials belong (OS keychain, environment-variable indirection)
- MCP server tool allow lists
- the tradeoffs around local history and memory retention
- shared templates and profile-based workflows
- organization-wide rollout via `requirements.toml`, and the beta permission profiles

It is not mainly about:

- organization-specific DLP / SIEM / EDR architecture
- replacing official OpenAI documentation
- providing a universal config that is automatically right for every environment

## Notes

- The main cheatsheet also explains how OWASP GenAI and prompt-injection references can inform Codex CLI hardening decisions
- It also covers secure design principles such as human-in-the-loop, least privilege, and defense in depth
- The shared templates prioritize clarity and operational simplicity first
- Fine-grained approval setups should usually come after the team has a concrete need for them
- Config keys and behavior change across Codex CLI versions. The current text was verified against **Codex CLI 0.146.0 as of 2026-08-01**, using both the official documentation and the running binary; where the two disagree, the text says so

## References

- Codex Configuration reference: https://developers.openai.com/codex/config-reference
- Codex Permissions (beta): https://developers.openai.com/codex/permissions
- Codex admin-enforced requirements: https://developers.openai.com/codex/enterprise/managed-configuration
- OpenAI Codex config example discussion: https://github.com/openai/codex/issues/2760
- Claude Code hardening cheatsheet by okdt: https://github.com/okdt/claude-code-hardening-cheatsheet

## Author

Riotaro OKADA

## Acknowledgements

The structure and publishing direction of this repository were informed by okdt's Claude Code hardening cheatsheet.

## License

CC BY-SA 4.0. See [LICENSE](./LICENSE).
