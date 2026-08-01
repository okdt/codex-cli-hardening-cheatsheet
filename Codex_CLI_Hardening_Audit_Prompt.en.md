# Codex hardening audit prompt by okdt

Help me apply the following guide to my own usage and my actual environment.

https://github.com/okdt/codex-cli-hardening-cheatsheet/blob/main/Codex_CLI_Hardening_Cheat_Sheet.en.md

Start by cross-checking the guide, the current official OpenAI documentation, and the behavior of the installed Codex, reading only. Establish the facts you need in order to judge what applies here. Ask me only about things you cannot determine from the environment and that would change the decision.

Once you have shown me the overall picture and a prioritized list of improvement candidates, take exactly one at a time and decide it with me. For each candidate, briefly explain the current state, the risk, the benefit and the inconvenience, the concrete change, how to confirm it, and how to undo it, then give me your recommendation. Do not change anything until I have decided to apply, adjust, defer, or decline.

After I approve, apply and verify that change alone, confirm the result, and only then move on. Record the original state before the first change, and track this session's changes so they can be reverted individually or all together. **Keep that record to the structure and the key names; never include secret values in it.** Removing plaintext secrets from the configuration is part of the point of this exercise, so the record must not become another copy of them. If I ask to restore, explain the scope, the impact, and the steps, then restore after my approval and confirm the result. Say so before applying anything that cannot be fully reverted. **Treat removing or rotating a secret as one of those non-reversible changes.**

Never display or store secret values, and never make unapproved or destructive changes. Where something is already sufficiently safe, say that no change is needed.

Finish with a summary that separates what was already configured safely from what needed hardening, along with what you did.

If you find anything that would improve the guide, mention that the author, okdt, welcomes feedback with confidential details removed.
