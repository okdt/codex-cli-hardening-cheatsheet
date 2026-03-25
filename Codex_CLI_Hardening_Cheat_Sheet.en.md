# Codex CLI Hardening Cheatsheet

This is a practical cheatsheet for running Codex CLI with a security-conscious baseline. It combines general hardening guidance with concrete Codex CLI examples for `sandbox`, `approval`, `network`, and `history`.

It follows the same "usable in the real world" structure as the Claude Code hardening cheatsheet, while aligning the content to Codex CLI concepts and documentation.

## Risks To Keep In Mind First

Codex CLI can read and write files, run shell commands, change settings, and in some cases access the network. That is powerful, but loose defaults can create avoidable risk.

- Accidentally modifying files outside the intended workspace
- Running unintended commands that alter or pollute the local environment
- Fetching dependencies or calling external APIs when that was not the original intent
- Leaving sensitive context in local history or session traces
- Creating operational confusion through vague profile names or approval rules

This cheatsheet is meant to reduce those risks without making everyday development unnecessarily painful.

Relevant external references:

- OWASP GenAI Security Project: https://genai.owasp.org/
- OWASP Prompt Injection: https://owasp.org/www-community/attacks/PromptInjection
- OWASP LLM Prompt Injection Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/LLM_Prompt_Injection_Prevention_Cheat_Sheet.html

## Goals

- Translate general hardening practices into Codex CLI operations
- Show practical, security-conscious examples centered on `sandbox`, `approval`, `network`, and `history`
- Provide a clear path to commented `config.toml` templates

## Core Secure Design Principles

### Human-In-The-Loop

High-impact actions should not be fully automated by default. In Codex CLI, the clearest baseline for that is:

```toml
approval_policy = "on-request"
```

This makes it easier to keep human judgment in the loop for actions such as:

- Changes that cross workspace boundaries
- Tasks that require network access
- Destructive commands
- Unexpected permission expansion

### Principle Of Least Privilege

Do not grant strong permissions up front. Start narrow, and only widen them when there is a real reason.

In Codex CLI, that maps naturally to:

- `sandbox_mode = "workspace-write"`
- `writable_roots = []`
- `network_access = false`
- `allow_login_shell = false`

When extra access is needed, it is usually safer to add it temporarily through `--add-dir` or a dedicated profile such as `net_enabled`.

### Defense In Depth

Do not rely on a single safeguard. Layer multiple controls so one weak point does not become a full compromise.

In Codex CLI, that usually means combining:

- `sandbox` to constrain write scope
- `approval` to pause high-risk actions
- `network` defaults that stay closed unless explicitly enabled
- `history` choices that limit unnecessary context retention

Even when one layer is imperfect, the others can still reduce impact.

### Explicit Operational Boundaries

Secure design is not just about config keys. Profile names and operational rules matter too.

If names and behavior drift apart, operators make mistakes before the config itself even fails.

That is why this cheatsheet favors:

- Clear profile names such as `readonly_quiet`
- Avoiding misleading names such as `full_auto`
- Keeping the shared baseline simple and expressing exceptions through profiles or one-off flags

## What Is Specific To Codex CLI

Some security principles are common across coding agents, but Codex CLI has a few characteristics that shape how hardening works in practice.

### 1. `sandbox_mode` And `approval_policy` Are The Main Levers

In Codex CLI, the core question is how much the agent is allowed to do automatically, and when a human needs to approve it.

- `sandbox_mode`: how much file access and execution latitude is allowed
- `approval_policy`: where human review is introduced

Those two settings form the backbone of a sane baseline.

### 2. `workspace-write` Is A Natural Default

The gap between `read-only`, `workspace-write`, and `danger-full-access` is significant. That makes `workspace-write` a strong everyday default.

- `read-only`: good for inspection and reconnaissance
- `workspace-write`: good for normal editing
- `danger-full-access`: only for exceptional cases

That is why this cheatsheet treats `workspace-write` as the main baseline and moves stricter or broader only when needed.

### 3. Network Access Hangs Off The Sandbox Configuration

In Codex CLI, `network_access` lives under `sandbox_workspace_write`. In practice, that means network policy is not just "on or off" in the abstract. It is tied to a specific execution posture.

That is also why this cheatsheet separates `net_enabled` into its own profile instead of treating network access as a casual toggle.

### 4. Profiles Fit The Operational Model Well

Codex CLI works well with profile-based deltas. That makes it easy to keep the shared default conservative while still giving operators a clean way to switch posture by use case.

Typical examples:

- `readonly_quiet`
- `local_write`
- `net_enabled`

When permissions and purpose line up cleanly, mistakes become less likely.

### 5. TOML Structure Matters

Because Codex CLI uses `config.toml`, TOML itself becomes part of the operational risk surface.

For example, this will fail:

```toml
approval_policy = "on-request"

[approval_policy.granular]
rules = true
```

That is invalid because `approval_policy` was already defined as a string and is then reused as a table.

So Codex hardening is not only about choosing safer values. It is also about producing valid, maintainable TOML.

### 6. Real-World Concerns Like `trusted`, Login Shells, And History Matter

Codex CLI also has practical tuning points that do not fit into a simple allow/deny story.

- `trust_level = "trusted"` is convenient, but trades away guardrails
- `allow_login_shell = false` reduces dependence on shell startup state
- `history.persistence` is a real usability vs. retention tradeoff

Those are not generic AI-agent talking points. They are Codex CLI operational decisions.

## Bottom Line

A good starting baseline is:

- `sandbox_mode = "workspace-write"`
- `approval_policy = "on-request"`
- `network_access = false`
- `history.persistence = "save-all"` by default, with `none` as an option when confidentiality matters more
- `allow_login_shell = false`
- `writable_roots = []`

This combination keeps normal workspace edits practical, while making broader permissions and network access explicit decisions.

## Quick Start

### Minimal Safer Template

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

### Everyday Usage Split

```bash
# Inspection only
codex --profile readonly_quiet

# Normal local editing
codex --profile local_write

# Only when network access is actually needed
codex --profile net_enabled
```

## General Hardening Tips

### 1. Keep The Default Narrow, Expand Only When Needed

- Keep the shared baseline at `workspace-write`
- Widen access only through a profile or `--add-dir` when the workflow really needs it
- Do not use `danger-full-access` as a shared default

### 2. Keep Network Closed By Default

- Outbound communication increases the impact of unintended actions
- Only enable it when dependency fetches, web access, or external APIs are actually required
- Remote content is also one of the entry points for prompt injection and data exfiltration

### 3. Treat History As A Usability vs. Retention Tradeoff

- Session history can include prompts, intent, and operational context
- For day-to-day work, `history.persistence = "save-all"` is often more practical
- When confidentiality matters more, switch to `history.persistence = "none"`
- In agentic workflows, retained context is itself part of the risk surface

### 4. Keep Names Aligned With Behavior

- If approval is still required, do not call the profile `full_auto`
- Prefer names such as `readonly_quiet`, `local_write`, and `net_enabled` that reflect purpose clearly

### 5. Keep The Shared Baseline Simple, Use Temporary Exceptions

- Put your default policy in `~/.codex/config.toml`
- Add temporary exceptions via `--profile`, `--add-dir`, or `--config`

## How To Think About Each Setting Area

### sandbox

The main sandbox modes in Codex CLI are:

- `read-only`
- `workspace-write`
- `danger-full-access`

For security-conscious everyday use, `workspace-write` is usually the right baseline. It still allows local editing without jumping straight to full access.

Recommended example:

```toml
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
exclude_slash_tmp = true
exclude_tmpdir_env_var = true
writable_roots = []
```

Key points:

- `writable_roots = []` avoids silently broadening writable scope
- Excluding `/tmp` and `$TMPDIR` keeps temporary areas from becoming implicit write channels
- If extra writable space is needed, prefer `--add-dir /path/to/dir`

### approval

The main approval policies in Codex CLI are:

- `untrusted`
- `on-request`
- `never`

For normal interactive use, `on-request` is the easiest shared default to explain and operate.

Recommended example:

```toml
approval_policy = "on-request"
```

Key points:

- It helps stop actions that cross trust boundaries
- It is lighter than forcing full manual review every time
- There is no need to push granular policy design into the shared baseline too early
- From an OWASP perspective, it maps cleanly to a human-in-the-loop control for high-risk actions

### network

Network policy should be managed explicitly. Even under `workspace-write`, it is usually better to keep network access off unless there is a concrete reason to allow it.

Recommended example:

```toml
[sandbox_workspace_write]
network_access = false
```

Enable it only through a profile when needed:

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

Key points:

- Fetching remote content can become an entry point for indirect prompt injection
- It is safer to keep network access closed by default and open it deliberately

### history

History is useful, but it is also part of the retention story. This cheatsheet treats `save-all` as the practical default while keeping `none` available for stricter environments.

Recommended example:

```toml
[history]
persistence = "save-all"
```

Key points:

- Persisted history helps with continuity and review in day-to-day work
- It can also retain prompts and sensitive context
- This matters even more on shared machines or in regulated environments
- Over time, context retention and persistence attacks become part of the model

## Effective Profile Examples

### 1. Quiet Read-Only Inspection

```toml
[profiles.readonly_quiet]
approval_policy = "never"
sandbox_mode = "read-only"
```

Good for:

- First look at an unfamiliar repository
- Structural inspection
- Review before making changes

### 2. Normal Local Editing

```toml
[profiles.local_write]
approval_policy = "on-request"
sandbox_mode = "workspace-write"
```

Good for:

- Local fixes
- Test runs
- Small refactors

### 3. Turn Network On Only When Needed

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

Good for:

- Fetching dependencies
- Looking up official documentation
- Minimal work that really requires external APIs

## Settings To Avoid

### 1. Everyday `danger-full-access`

```toml
sandbox_mode = "danger-full-access"
approval_policy = "never"
```

This is effectively near-full access. Unless you are already inside a deliberately isolated environment, it should not be the shared default.

### 2. Always-On Network Access

```toml
[sandbox_workspace_write]
network_access = true
```

That is too broad for a shared baseline unless there is a very specific reason.

### 3. Treating History Retention As Automatically Fine

```toml
[history]
persistence = "save-all"
```

This is practical, but it is not always the right answer. In more sensitive environments, `none` may be the better choice.

### 4. TOML Collisions Around `approval_policy`

The following will error:

```toml
approval_policy = "on-request"

[approval_policy.granular]
rules = true
```

That happens because `approval_policy` was first defined as a string and then extended as a table. For a safer baseline, it is usually better to keep granular approval policy out of the shared template until the team has a clear need for it.

## How To Roll This Out

### Personal Default

- Put the safer baseline in `~/.codex/config.toml`

### Project-Specific Adjustments

- Add only the settings you really need in `.codex/config.toml`
- Example: a specific repository may need additional `writable_roots`

### Temporary Exceptions

```bash
# Allow writes only to one extra directory
codex --profile local_write --add-dir /path/to/output

# Enable network access temporarily
codex --config sandbox_workspace_write.network_access=true
```

## Included Template Files

Commented templates:

- [codex_config_min_safe_template.toml](./codex_config_min_safe_template.toml)
- [codex-config.hardened.template.toml](./codex-config.hardened.template.toml)

## References

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
