---
name: agent-notification-sounds
description: Diagnose, restore, or preserve local approval-request and task-completion sounds for Codex, Claude Code, and other AI terminal or desktop assistants. Use when lifecycle hooks, notification wrappers, sound files, or a third-party terminal app may have replaced, suppressed, or duplicated assistant notifications.
---

# Agent Notification Sounds

Restore only the requested lifecycle cues. Treat an approval request and a completed turn as separate events.

## Stable assets

Use the bundled files, not another assistant's configuration directory:

- `assets/approval-needed.wav` — approval or input required
- `assets/task-complete.wav` — completed or stopped

Keep these files here. Do not delete, rename, or replace them while editing another assistant's configuration. Use absolute paths in hook commands.

## Diagnose first

1. Identify how the assistant was launched: inspect `command -v`, `type -a`, wrapper scripts earlier on `PATH`, and the application's launch configuration.
2. Find the active hook configuration and any wrapper references with `rg` before editing. Distinguish a persistent config-file rewrite from process-scoped command-line overrides.
3. Check the assistant's lifecycle vocabulary. Map the desired behaviors to its actual event names; common names include `PermissionRequest`, `Notification`, `Stop`, `SessionEnd`, or a completion callback.
4. Confirm the player and assets work directly before blaming the hook. On macOS, run `/usr/bin/afplay -v 0.6 <asset>`.

## Implement safely

1. Preserve unrelated hooks and existing user changes. Make the smallest config change that binds the approval event to `approval-needed.wav` and the completion event to `task-complete.wav`.
2. Prefer one small dispatch script over duplicated inline shell. It should select only the two known event kinds, use absolute paths, return success if the optional desktop-notification utility is absent, and start playback asynchronously.
3. If desktop notifications are wanted, make them supplemental; the sound must not depend on `terminal-notifier` or a third-party app notification permission.
4. Let the host assistant record trust for a newly changed hook. Do not invent or hand-edit hook trust hashes. Start a short local session, review the changed hooks, and choose the product's trust action.
5. Validate both paths: direct playback, then a real completion event. Trigger an approval event too when the active approval policy permits it.

## Wrapper and desktop-app pitfalls

- A wrapper can send its own notifications without overwriting the assistant's hook file. Inspect its final command: a flag such as `-c notify=...` is generally process-scoped, whereas an installer that writes `hooks.json` or a settings file is persistent.
- Disabled macOS notification permission normally suppresses the wrapper application's visual/audio notification, but it does not stop the assistant's independently configured hook from calling `afplay`.
- A desktop app may cache configuration for an already-open task. Open a new task or restart the app after changing global hook configuration.
- Both a wrapper and the assistant may report completion. If the wrapper's notification is enabled, duplicate alerts are possible even when the hook file is unchanged.

## Current Superset check

In this environment, `/Users/admin/.superset/bin/codex` does not write `~/.codex/hooks.json`. It launches Codex with a process-scoped `-c notify=["bash","/Users/admin/.superset/hooks/notify.sh"]` override and a per-session watcher. Therefore enabling Superset will not overwrite the restored `PermissionRequest` or `Stop` hooks. Its own notification callback can still be invoked during a Superset-launched session; disabled Superset notification permission prevents that app's notification, while the independent Codex sound hooks remain active.

## Handoff

Report the active config path, the two event mappings, whether a wrapper was persistent or scoped, whether hook trust was accepted, and which real lifecycle events were verified.
