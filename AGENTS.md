# Workspace Notes

This repository contains the Codex CLI hardening cheatsheet and related templates.

## Scope

- Primary audience: Codex CLI users who want practical hardening guidance.
- The Japanese documents are the primary source of truth.
- The English documents should stay aligned in structure and meaning, but should read naturally in English.

## Content Priorities

- Prefer practical, operational guidance over abstract security theory.
- When discussing security controls, explain both the reason and the tradeoff.
- Keep README files concise. Put deeper guidance in the cheatsheet documents.
- Treat instruction files as contextual guidance, not as the place to enforce security policy.

## File Intent

- `README.md`: Japanese top-level overview for the repository.
- `README.en.md`: English companion overview.
- `Codex_CLI_Hardening_Cheat_Sheet.ja.md`: Main Japanese cheatsheet.
- `Codex_CLI_Hardening_Cheat_Sheet.en.md`: Main English cheatsheet.
- `codex-config.hardened.template.toml`: Commented template.
- `codex_config_min_safe_template.toml`: Smaller baseline template.

## Editing Notes

- Preserve the repository's practical tone.
- Avoid overstating guarantees; describe limits and tradeoffs clearly.
- Keep examples realistic for day-to-day CLI use.

## Git Safety

- Do not run `git add`, `git commit`, `git push`, or open pull requests unless explicitly requested.
- Treat staging changes as a user-controlled step.
