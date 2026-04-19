# cmux-parity plan (revised after architectural discovery)

## Architecture discovery

Limux has **two control servers**:

1. **Standalone `limux-control-server` binary** — uses `limux_core::Dispatcher`
   + `ControlState` and supports the **full** command vocabulary. Used for
   tests and for CLI calls when the GUI isn't running.

2. **Embedded bridge inside `limux-host-linux`** — `control_bridge.rs` only
   routes a narrow subset of methods to the GTK main loop. Supports
   `system.ping`, `system.identify`, `workspace.{current,list,create,
   select,rename,close}`, `surface.send_text`. **Does NOT support**
   `pane.list`, `pane.surfaces`, `surface.list`, `surface.read_text`,
   `surface.send_key`, `notification.*`, or any browser commands.

When the GUI is running, the CLI targets the bridge via the runtime
socket. `list-panes`, `read-screen`, and most other commands currently
**error out** against the running host — this is the root blocker for
the Codex↔Claude workflow.

## Delivery strategy (revised)

### Phase 1 — Env auto-wiring ✅ (shipped in 1295d12)

### Phase 2 — Make the bridge a full proxy (CRITICAL)

The bridge should route unknown methods to a local `Dispatcher` instance
seeded with live GTK state, OR to dedicated per-method `ControlCommand`
variants that interrogate the live state. The cleanest path:

- Maintain a `Arc<Mutex<ControlState>>` owned by the GTK app, kept in
  sync with live workspace/pane/surface state.
- Bridge falls through unknown methods to `Dispatcher::dispatch` on that
  shared state.
- Specific methods that need GTK side-effects (send_text, create_surface,
  notification.create) remain as `ControlCommand` variants.

This unblocks `list-panes`, `pane.surfaces`, `surface.list`,
`surface.read_text`, `surface.send_key` against the live GUI — i.e.
everything needed for agents to discover each other and read each
other's screens.

### Phase 3 — `limux notify` + GUI toast/sidebar integration
Wire a new `ControlCommand::CreateNotification` variant in the bridge,
plumb into `mark_workspace_unread_with_message` + libadwaita toast.
Add CLI subcommand.

### Phase 4 — `limux claude-hook` / `opencode-hook` / `gemini-hook`
Read hook JSON from stdin, translate to `notify`/`send`. Provides a one-line
install into `~/.claude/settings.json`.

### Phase 5 — `limux agent-team` + `AGENTS.md` template
Spawns N agent surfaces, captures their surface IDs, writes
`./AGENTS.md` with the XML message protocol and the surface-id table,
launches each agent with the seed prompt.

### Phase 6 — (deferred) `limux progress`, `limux log`, `limux markdown`
Nice polish, not blockers.

## Why phase 2 first

Without a real bridge, every subsequent feature ends up routing around
the same hole: the GUI owns the ground truth about surfaces/panes but
the CLI can't query it. Fixing this once, properly, makes phases 3–5
small.
