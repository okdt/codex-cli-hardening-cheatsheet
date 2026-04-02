# Codex CLI Hardening Cheatsheet

Codex CLI can read and write files, run shell commands, change settings, and, in some cases, access the network. That is powerful, and it comes with real risk.

This cheat sheet is a practical guide to configuring Codex CLI to keep risk manageable. It focuses on `sandbox`, `approval`, `network`, and `history`, and organizes a safety-oriented setup around the official Codex CLI documentation. If you are new to this, start with the Quick Start template in the rollout section. If you want to understand the design choices behind it, read through the whole document.

## Risk: Why Hardening Matters

Loose defaults make tools feel convenient right up until something goes wrong. With Codex CLI, the main risks look like this:

- **Indirect prompt injection can steer the agent into unintended actions**
  A fetched README, issue, document, or generated artifact may contain instructions that nudge the model into doing something you did not ask for, such as "let me install that package first" or "I will run this helper script before continuing." ([OWASP LLM01:2025 Prompt Injection](https://genai.owasp.org/llm-top-10/))
- **Overly broad permissions turn mistakes into real damage**
  If you grant `danger-full-access` or overly broad writable roots, bad judgment no longer stays theoretical. The model may start cleaning up files outside the workspace or rewriting your `.gitconfig` in the name of being helpful. Technically plausible does not mean operationally acceptable. ([OWASP LLM06:2025 Excessive Agency](https://genai.owasp.org/llm-top-10/))
- **History and retained context can expose sensitive information**
  Prompts, diff explanations, internal URLs, connection targets, and token fragments can all end up stored locally. The amount of information left behind is often larger than people expect when they later review it. ([OWASP LLM02:2025 Sensitive Information Disclosure](https://genai.owasp.org/llm-top-10/))
- **Plausible-sounding advice is still easy to overtrust**
  A setting recommendation or operational explanation can sound confident and still be wrong. Humans do this too, but models rarely look uncertain when they are guessing. ([OWASP LLM09:2025 Misinformation](https://genai.owasp.org/llm-top-10/))
- **Vague profile names and fuzzy approval rules cause operational mistakes**
  This is partly a configuration management problem, but in practice it becomes permission misuse and missed review. Ambiguous names cause trouble before the config itself breaks.

These are not hypothetical concerns. At some point, something will go wrong. The goal of hardening is to keep the damage contained without making everyday development unbearably heavy.

## The Basic Approach

So what can you actually control? In Codex CLI, hardening mainly happens through these `config.toml` axes:

1. **Sandboxing**: `sandbox_mode` controls how far the agent can write. `workspace-write` keeps writes inside the workspace. `read-only` blocks writes entirely. This is the most fundamental control because it is enforced outside the model itself.
2. **Approval policy**: `approval_policy` decides whether a human has to approve actions that cross the sandbox boundary. `on-request` is the practical implementation of human-in-the-loop.
3. **Network restrictions**: `network_access` controls outbound communication. It lives under the sandbox configuration, but it functions as its own defensive layer. If it is closed, even an indirectly injected action cannot easily reach outside the machine.
4. **History retention**: `history.persistence` controls how much session history is kept. This is a usability versus information-retention tradeoff.
5. **Logging and Telemetry**: this matters more in enterprise environments and for debugging the hardening setup itself. Codex CLI quietly has a fairly capable story here, including OpenTelemetry integration and session rollout logs.

> **Point:** layering these controls is a classic **defense-in-depth** strategy. If one layer is imperfect, another can still reduce the blast radius.

### "Can I Just Put It In Instruction Files?" No.

Codex CLI can use project-context files such as `AGENTS.md`, user-level instruction files such as `~/.codex/instructions.md`, and supplemental definition files such as `SKILL.md`. In this document, I refer to these collectively as **instruction-family files**. Their purpose and scope differ, but they all exist to provide context and working guidance.

But they are not a true enforcement layer. They are still, fundamentally, user prompts. That makes them the wrong place to rely on for controls such as:

- disabling network access
- preventing writes outside specific directories
- blocking destructive commands
- requiring approval for risky actions

Those controls are better enforced through `config.toml`, specifically the `sandbox`, `approval`, and `network` settings. Instruction-family files are best treated as supplemental guidance.

They also have size limits. According to OpenAI's official guidance, user-instruction material is subject to a 32 KiB limit. That is another reason not to turn these files into an ever-growing operating manual.

For both reasons, instruction-family files are not the right place for security restrictions. If something genuinely must not happen, it should be enforced as policy, not merely requested as guidance. That is the starting point for the rest of this document.

## Core Secure Design Principles

### Human-In-The-Loop

Do not fully automate high-impact actions.

AI systems often do things that are technically reasonable but operationally out of bounds: deleting files while "cleaning up," force-pushing while "updating" a branch, or broadening permissions while "fixing" a workflow. These are not always malicious. Often they are just over-helpful. Human review is how you stop that.

In Codex CLI, the simplest practical implementation is:

```toml
approval_policy = "on-request"
```

That makes it easier to keep human judgment in the loop for actions such as:

- changes that cross workspace boundaries
- tasks that require network access
- destructive shell commands
- unexpected permission expansion

By default, Codex CLI can do whatever your user account is allowed to do. One approval is sometimes all that stands between a useful action and a damaging one. `on-request` exists to put a checkpoint there.

### Principle Of Least Privilege

Do not start with more access than you need.

"We can tighten it later" rarely works in practice. Once you begin with broad permissions, it becomes hard to tell what was actually necessary. Starting narrow and widening only when something proves necessary gives you a much clearer picture of the real requirements.

In Codex CLI, that usually maps to:

- `sandbox_mode = "workspace-write"`
- `writable_roots = []`
- `network_access = false`
- `allow_login_shell = false`

When something extra is required, it is safer to add it temporarily through `--add-dir` or a `remote_enabled` profile.

### Defense In Depth

Do not rely on a single setting.

Security people like to say "avoid a single point of failure." Configurations are no different. If everything depends on sandboxing alone, a sandbox mistake becomes a total failure.

The combination of sandboxing, approval, network restrictions, and history choices is a concrete defense-in-depth design. If sandboxing is broader than intended, approval may still stop the action. If approval is granted too easily, closed network access may still prevent external impact.

### Explicit Operational Boundaries

Secure design is not only about the file format. Profile names and operating rules matter too.

If the name of a profile and its actual behavior drift apart, human operators make mistakes before the configuration itself even fails. A profile that still requires approval should not be called `full_auto`, because people will use it as if it were full auto.

That is why this cheatsheet favors:

- names such as `readonly_quiet` that clearly communicate purpose
- avoiding misleading names such as `full_auto`
- keeping the shared baseline simple, then expressing exceptions through profiles or one-off options

## Let’s Harden It

### 1. Sandboxing

Codex CLI has three main sandbox modes:

- `read-only` (default): no writes at all; good for inspection and review
- `workspace-write`: writes allowed only inside the workspace; the normal everyday baseline
- `danger-full-access`: effectively almost anything; this can reach files like `.gitconfig`, `.bashrc`, or `~/.ssh/config`

```toml
# ~/.codex/config.toml
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
exclude_slash_tmp = true
exclude_tmpdir_env_var = true
writable_roots = []
```

- `writable_roots = []` is the safe default. Every additional path broadens what the agent can modify.
- Including `/tmp` or `$TMPDIR` creates extra routes for passing data around outside the intended workspace boundary.
- If you need an extra writable directory, adding it temporarily with `--add-dir /path/to/dir` is safer than widening the shared baseline permanently.
- `allow_login_shell = false` helps prevent shell startup customizations such as aliases or PATH changes from subtly changing the agent's behavior. That also improves reproducibility.

**Avoid this:**

```toml
# ~/.codex/config.toml — avoid this
sandbox_mode = "danger-full-access"
approval_policy = "never"
```

This is close to unrestricted local access. Unless you are already inside a deliberately isolated environment, it should not be the shared default.

The problem is not only overtly malicious commands. It is also well-intentioned but sloppy automation. "Let me clean that up" can delete files outside the workspace. "I fixed the configuration" can rewrite your `.gitconfig`. "I cleared stale caches" can wipe a temporary directory you were still using. `danger-full-access` plus `approval_policy = "never"` turns all of that into a path with no real checkpoint.

### 2. Approval Policy

Codex CLI has three approval policies:

- `untrusted`: every operation requires approval; strongest, but heavy in day-to-day use
- `on-request` (default): asks for approval when an action crosses the sandbox boundary; the practical middle ground
- `never`: no approval; acceptable for read-only investigation, risky when combined with write access

```toml
# ~/.codex/config.toml
approval_policy = "on-request"
```

- It helps stop actions that cross trust boundaries
- It is much lighter than full manual review on every step
- There is no need to push granular approval design into the shared template too early
- From an OWASP perspective, it lines up with the recommendation to keep human-in-the-loop around high-risk actions via the [AI Agent Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html)
- `trust_level = "trusted"` is convenient, but if a trusted context carries injected instructions, they may go through with fewer pauses; use it with that tradeoff in mind

**Command-level rules (`execpolicy`, preview):**

`approval_policy` sets the posture for the whole session, but Codex CLI also has a preview mechanism for controlling individual command families with `allow`, `prompt`, and `forbidden`. This is roughly comparable to deny/ask/allow rules in Claude Code. Rules are written in Starlark (a Python subset) under `~/.codex/rules/*.rules`.

```starlark
# ~/.codex/rules/default.rules
prefix_rule(
    pattern = ["git", "reset", "--hard"],
    decision = "forbidden",
    justification = "destructive operation",
)
```

At the moment, these rules operate on shell-command prefixes. They are not file-operation rules in the style of `Read(**/.env)`. If you switch `approval_policy` into a granular form and enable `rules = true`, `prompt` rules become active. This is still [in preview](https://github.com/openai/codex/blob/main/codex-rs/execpolicy/README.md), so breaking changes remain possible. See [Rules / execpolicy](https://developers.openai.com/codex/rules) for details.

### 3. Network Restrictions

Network access is configured under the sandbox, but it can be set independently from other sandbox settings. Setting `network_access = false` blocks all outbound communication at the OS level — this is a hard block that approval cannot override.

In practice, most users will run with `network_access = true`. But for tasks that do not need external access — internal refactoring, data cleanup, local code generation — there is no reason to leave the network open. Use `false` for those.

Remote content is also one of the main entry points for indirect prompt injection: package READMEs, post-install scripts, linked documentation, and issue threads can all contain instructions that influence the model. If network access is already open, those nudges have much more room to spread. The same applies to data exfiltration.

```toml
# ~/.codex/config.toml
sandbox_mode = "workspace-write"   # required for the following to matter

[sandbox_workspace_write]
network_access = false   # default
```

Enable it only when necessary, usually through a profile:

```toml
# ~/.codex/config.toml
[profiles.remote_enabled.sandbox_workspace_write]
network_access = true
```

**Avoid this as a shared default:** `network_access = true`. Always-on outbound access gives indirectly injected actions a direct path to the outside world, and it makes accidental leaks of tokens or internal details easier.

### 4. History Retention

Saved history is useful, but it helps to be clear-eyed about what it actually keeps.

Session history can include the prompts you typed, the model's responses, commands that were run, and their outputs. In practice, that means API-key fragments, hostnames, debugging notes, internal URLs, and customer-specific identifiers may all be preserved.

```toml
# ~/.codex/config.toml
[history]
persistence = "save-all"   # default
```

- Saved history is genuinely useful for continuity and review
- If confidentiality matters more, switch to `persistence = "none"`
- On shared machines or in workplace environments, check who can read the retained history
- In more agentic workflows, accumulated history becomes part of the retention risk itself

**Do not treat `save-all` as automatically harmless.** In more sensitive environments, `none` is often the better answer. Shared terminals, account handoffs, and offboarding scenarios can leave far more exposed than people expect.

### 5. Logging

Codex CLI has a more substantial logging and telemetry story than the official docs initially make obvious.

**Session rollout logs:**

- stored automatically under `~/.codex/sessions/` as JSONL
- separate from `history.persistence`
- useful for review and debugging because they retain the session, including tool execution results

**OpenTelemetry integration:**

The `[otel]` section in `config.toml` can send logs, traces, and metrics to an external OTLP collector such as Jaeger, Grafana, or Datadog.

```toml
# ~/.codex/config.toml
[otel]
environment = "dev"

[otel.trace_exporter.otlp-http]
endpoint = "https://otlp.example.com"
```

Typical events include:

- `codex.tool_decision`: approvals, denials, and what drove them
- `codex.tool_result`: tool results, including success or failure, arguments, and output
- `codex.api_request`: API request traces

This can be useful both for enterprise audit trails and for debugging the hardening setup itself.

**Application logs:**

- written to `~/.codex/log/`
- configurable via `log_dir` in `config.toml`

## How To Roll This Out

### Quick Start

If you want a workable shared baseline, start with this in `~/.codex/config.toml`. It keeps ordinary workspace editing usable, while making broader permissions and network expansion explicit choices.

```toml
# ~/.codex/config.toml
model = "gpt-5.4"
approval_policy = "on-request"
sandbox_mode = "workspace-write"
allow_login_shell = false

[history]
persistence = "save-all"

[sandbox_workspace_write]
exclude_slash_tmp = true
exclude_tmpdir_env_var = true
writable_roots = []

# If you do not need to access the network
# network_access = false

# Optional switching profiles
[profiles.readonly_quiet]
approval_policy = "never"
sandbox_mode = "read-only"

[profiles.local_write]
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[profiles.remote_enabled]
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[profiles.remote_enabled.sandbox_workspace_write]
network_access = true
exclude_slash_tmp = true
exclude_tmpdir_env_var = true
writable_roots = []
```

### Project-Specific Adjustments

- Put only the extra settings you really need into `.codex/config.toml` at the project root
- For example, a specific repository may need extra `writable_roots`

### Everyday Profile Use

If you define multiple `[profiles.name]` blocks in `config.toml`, you can switch them with `--profile`. The profile values override the base settings.

```bash
# Just inspecting
codex --profile readonly_quiet

# Normal local editing
codex --profile local_write

# Work that needs to touch resources beyond the local machine
codex --profile remote_enabled
```

### Temporary Exceptions

If profiles are not enough, temporary command-line overrides can fill the gap.

```bash
# Allow writes to one extra directory
codex --profile local_write --add-dir /path/to/output

# Turn on network access temporarily
codex --config sandbox_workspace_write.network_access=true
```

### Operational Notes For Profiles

- Treat profiles as a launch-time execution posture and pass the one you want explicitly
- After using `remote_enabled`, it is easy to keep working with a broader-than-intended mindset; many accidents happen on the "forgot to switch back" path
- In CI and automation, pin the profile explicitly rather than relying on implicit defaults
- For human workflows, it helps to make the rule explicit: `local_write` by default, `remote_enabled` only when the task clearly requires it

## Included Template Files

Commented templates:

- [codex_config_min_safe_template.toml](./codex_config_min_safe_template.toml)
- [codex-config.hardened.template.toml](./codex-config.hardened.template.toml)

## Official Documentation

- [Sandboxing](https://developers.openai.com/codex/concepts/sandboxing)
- [Agent approvals & security](https://developers.openai.com/codex/agent-approvals-security)
- [Command line options](https://developers.openai.com/codex/cli/reference)
- [Config basics](https://developers.openai.com/codex/config-basic)
- [Configuration reference](https://developers.openai.com/codex/config-reference)
- [Rules / execpolicy](https://developers.openai.com/codex/rules)

## Additional References

- [OWASP GenAI Security Project](https://genai.owasp.org/)
- [OWASP Prompt Injection](https://owasp.org/www-community/attacks/PromptInjection)
- [OWASP LLM Prompt Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/LLM_Prompt_Injection_Prevention_Cheat_Sheet.html)
- [Claude Code Hardening Cheatsheet](https://github.com/okdt/claude-code-hardening-cheatsheet)
