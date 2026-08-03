# Codex CLI hardening audit prompt by okdt

Help me apply the following guide to my own usage and my actual environment.

https://github.com/okdt/codex-cli-hardening-cheatsheet/blob/main/Codex_CLI_Hardening_Cheat_Sheet.en.md

(If you have downloaded this cheat sheet locally, read `Codex_CLI_Hardening_Cheat_Sheet.en.md` instead of fetching it.)

Start by cross-checking the guide, the current official OpenAI documentation, and the behavior of the installed Codex CLI — read-only, changing nothing. Establish the facts you need in order to judge what applies here. Ask me only about things you cannot determine from the environment and that would change the decision.

Two things to observe while you do that:

- **Check `codex --version`.** The configuration keys and default values the guide relies on were verified against one specific version. If mine differs, say so up front, and before writing any key, confirm that the key exists in my version. Do not write keys that do not exist; note the key name and its purpose in your report instead.
- **The guide and its templates deliberately spell out some values that match the current defaults.** Defaults change between versions; an explicit line does not. Do not propose removing or omitting those lines on the grounds that they are redundant.

Once you have shown me the overall picture and a prioritized list of improvement candidates, take exactly one at a time and decide it with me. For each candidate, briefly explain the current state, the risk, the benefit and the inconvenience, the concrete change, how to confirm it, and how to undo it, then give me your recommendation. Do not change anything until I have decided to apply, adjust, defer, or decline.

After I approve, apply and verify that change alone, confirm the result, and only then move on. Each time you apply a change, also confirm that Codex still starts without a configuration error. Record the original state before the first change, and track this session's changes so they can be reverted individually or all together. **Record non-secret values as the original value or an exact diff, and keep only secret values out of that record.** Removing plaintext secrets from the configuration is part of the point of this exercise, so the record must not become another copy of them. If I ask to restore, explain the scope, the impact, and the steps, then restore after my approval and confirm the result. Say so before applying anything that cannot be fully reverted. **Treat removing or rotating a secret as one of those non-reversible changes.**

Never display or store secret values, and never make unapproved or destructive changes. Where something is already sufficiently safe, say that no change is needed.

Compare `$CODEX_HOME/rules/default.rules` before and after the session. Choosing a persistent option in an approval dialog appends an allow rule to that file. Check whether a temporary approval made for this exercise has been left behind as a permanent one, and unless it was persisted on purpose, remove it once you have verified the change.

Finish with a summary that separates what was already configured safely from what needed hardening, along with what you did.

If you find anything that would improve the guide, mention that the author, okdt, welcomes feedback with confidential details removed.
