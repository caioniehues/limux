# Plan: Agent Self-Split Pane Spawning

**Generated**: 2026-05-03

## Overview

Add first-class live-GUI support for agents to create a new pane from inside
Limux and optionally launch another agent command in that pane. The primary
path is the existing CLI surface, `limux new-pane`, backed by a new embedded
GTK bridge route for `pane.create`. The implementation should extend the
canonical split/pane creation code in `window.rs`; it should not add a
parallel dispatcher shim or a second pane model.

Target user workflow:

```bash
limux new-pane --direction right --command claude
```

When run from an agent terminal, this should infer the caller's workspace and
source pane from `LIMUX_WORKSPACE_ID`, `LIMUX_SURFACE_ID`, and `LIMUX_PANE_ID`,
split that pane in the running GTK app, create a terminal pane, launch `claude`
in it, return the new pane/surface IDs in JSON mode, and make the pane
discoverable through `list-panes` / `list-panels`.

Context7 documentation checked:

- `/gtk-rs/gtk4-rs`: GTK widgets are not thread-safe and must be mutated on
  the main thread. Work from socket/background threads should be relayed to the
  GLib main loop, matching the existing `ControlCommand` bridge pattern.
- `/gnome/libadwaita`: Limux already uses libadwaita-style main application
  and toast patterns; no new libadwaita API is needed for pane creation.

## Prerequisites

- Existing branch: `feat/cmux-parity`
- Keep the current uncommitted bridge changes intact; do not revert them.
- Use the canonical quality gate: `./scripts/check.sh`
- If possible, run the GUI smoke test through `scripts/xvfb-smoke-test.sh` or
  a live Limux window. If headless GTK launch fails, record that explicitly.

## Dependency Graph

```text
T0 ──┬── T1 ──┬── T3 ── T5 ── T7 ── T8 ── T9 ── T10
     │        │              │
     └── T2 ──┘              │

T4 ───────────────┘
T6 ───────────────┘
```

## Tasks

### T0: Preflight the dirty tree and current bridge state

- **depends_on**: []
- **location**:
  - `rust/limux-cli/src/main.rs`
  - `rust/limux-host-linux/src/control_bridge.rs`
  - `rust/limux-host-linux/src/window.rs`
  - `rust/limux-host-linux/src/pane.rs`
  - `docs/cmux-parity-plan.md`
- **description**: Inspect the current uncommitted changes before editing.
  Preserve the existing agent-comms work, including `surface.send_key`,
  `pane.list`, `pane.surfaces`, and `surface.list` bridge changes. Record
  which files are already dirty so parallel agents do not overwrite each
  other's work.
- **validation**: `git status --short` and targeted `git diff -- <file>` have
  been reviewed; implementation tasks know which files are already modified.
- **status**: Completed
- **log**:
  - 2026-05-03: Preflight completed on branch `feat/cmux-parity`.
  - Dirty-tree check found no pre-existing uncommitted files:
    `git status --short` produced no output.
  - Targeted diff check found no uncommitted changes in
    `rust/limux-cli/src/main.rs`,
    `rust/limux-host-linux/src/control_bridge.rs`,
    `rust/limux-host-linux/src/window.rs`,
    `rust/limux-host-linux/src/pane.rs`, or
    `docs/cmux-parity-plan.md`: both `git diff --stat` and
    `git diff -- <target files>` produced no output.
  - Static bridge search verified current live-host route state:
    `control_bridge.rs` `METHODS` includes `pane.list`, `pane.surfaces`,
    `surface.list`, and `surface.send_key`; dispatch arms exist for those
    methods. `pane.create` is not present in host `control_bridge.rs` yet.
  - CLI/core path already references `pane.create` via `new-pane`
    serialization in `rust/limux-cli/src/main.rs`; T1/T3 should add the
    live GTK bridge contract/route without overwriting the existing
    agent-comms routes.
  - `reason_not_testable`: T0 is an inspection/preflight task with no
    production behavior change. Verification is static command evidence only.
- **files edited/created**:
  - `agent-self-split-plan.md`

### T1: Define the self-split `pane.create` contract

- **depends_on**: [T0]
- **location**:
  - `rust/limux-host-linux/src/control_bridge.rs`
  - `rust/limux-cli/src/main.rs`
  - `rust/limux-core/src/lib.rs`
- **description**: Align the live bridge request/response shape with the
  existing CLI and core dispatcher while making source-pane targeting
  explicit. Inputs should include `workspace_id` or workspace name/ref,
  `surface_id` and/or `pane_id`, `direction`, `type`, `url`, and new optional
  `command`. `new-pane` should default `workspace_id` from
  `LIMUX_WORKSPACE_ID`, `surface_id` from `LIMUX_SURFACE_ID`, and `pane_id`
  from `LIMUX_PANE_ID`, so agents split their own pane even if GTK focus has
  moved. Validate direction up front (`left|right|up|down`) and type
  (`terminal|browser`) before scheduling GTK work. Preserve existing JSON field
  names in responses: `pane_id`, `pane_ref`, `surface_id`, `surface_ref`.
- Accepted inbound IDs must include both raw values and `workspace:`,
  `pane:`, / `surface:` prefixed refs.
- For this delivery, the live GTK bridge should support `type=terminal` only.
  `type=browser`, `url`, or `command` combined with `type=browser` should fail
  fast with invalid params. Browser split support is a follow-up.
- Decide and document the standalone/core behavior for `command`: recommended
  path is to accept and validate the field in core for contract compatibility,
  but only the live GTK host actually injects it into a terminal. If core keeps
  ignoring host-only fields like source-pane targeting, direction, and command,
  tests and docs must state that those fields are live-host extensions.
- **validation**: A documented contract exists in code comments or tests; bad
  `direction` / `type` values return invalid params, not a silent fallback;
  source pane targeting and raw/ref ID parsing are specified before
  implementation starts.
- **status**: Completed
- **log**:
  - 2026-05-03: Added the shared `pane.create` contract in
    `control_bridge.rs`: workspace refs/raw IDs, source `pane_id`/`surface_id`,
    direction, type, and command parse shape are documented; invalid
    direction/type/browser-url combinations return invalid params before GTK
    work is scheduled.
  - 2026-05-03: Updated core `pane.create` to validate the same
    direction/type/source fields and accept raw/ref workspace/pane/surface IDs
    for compatibility. Core explicitly treats source pane, direction, and
    command as live-host extensions while preserving `pane_id`, `pane_ref`,
    `surface_id`, and `surface_ref` response fields.
  - 2026-05-03: Added tests for host parser contract, core invalid
    direction/type errors, ref/raw source IDs, and response field preservation.
  - 2026-05-03: `reason_not_testable`: a TDD subagent RED phase was requested
    but could not be spawned because the session agent thread limit was already
    reached; local contract tests were added instead.
  - 2026-05-03: Validation passed:
    `cargo test -p limux-core dispatcher_pane_create`;
    `cargo test -p limux-host-linux pane_create_contract`;
    `cargo check -p limux-cli`;
    `cargo check -p limux-host-linux`.
- **files edited/created**:
  - `agent-self-split-plan.md`
  - `rust/limux-cli/src/main.rs`
  - `rust/limux-core/src/lib.rs`
  - `rust/limux-host-linux/src/control_bridge.rs`

### T2: Identify the caller's source pane and split orientation

- **depends_on**: [T0]
- **location**:
  - `rust/limux-host-linux/src/window.rs`
  - `rust/limux-host-linux/src/pane.rs`
  - `rust/limux-host-linux/src/split_tree.rs`
- **description**: Reuse existing focused-pane discovery and split-tree
  helpers to decide which pane to split in the requested workspace. Resolve in
  this order: explicit `surface_id`, explicit `pane_id`, focused pane only when
  the target workspace is the active workspace, then first leaf pane fallback.
  Do not let a background agent split a random focused pane in another
  workspace. Map `left|right` to horizontal splits and `up|down` to vertical
  splits, using `new_pane_first=true` for `left|up`.
- **validation**: Completed with targeted host tests for direction-to-split
  placement, explicit `surface:`/`LIMUX_SURFACE_ID` style targeting, explicit
  pane targeting, active-workspace focused-pane selection, background-workspace
  first-pane fallback, invalid surface/pane errors, and no-pane errors. The
  live resolver returns `WorkspaceNotFound` before GTK work when the workspace
  target cannot be resolved. `cargo test -p limux-host-linux pane_create`
  passed; `cargo check -p limux-host-linux` passed. `cargo fmt --check` was not
  run globally because T1 has parallel uncommitted changes in
  `rust/limux-cli/src/main.rs`, `rust/limux-core/src/lib.rs`, and
  `rust/limux-host-linux/src/control_bridge.rs`; T2 files were formatted
  directly with `rustfmt rust/limux-host-linux/src/window.rs
  rust/limux-host-linux/src/pane.rs`.
- **status**: Completed
- **log**:
  - Added `resolve_pane_create_target` for later `ControlCommand::CreatePane`
    handling. It resolves source panes in order: explicit surface, explicit
    pane, focused pane only for the active workspace, then first pane fallback.
  - Added `PaneCreateDirection` split placement mapping: `left|right` use
    horizontal splits, `up|down` use vertical splits, and `left|up` place the
    new pane first.
  - Added a root-scoped `pane_widget_for_root` helper so later bridge code can
    split the resolved GTK pane without scanning unrelated workspaces.
- **files edited/created**:
  - `rust/limux-host-linux/src/window.rs`
  - `rust/limux-host-linux/src/pane.rs`
  - `agent-self-split-plan.md`

### T3: Add `ControlCommand::CreatePane` and parser routing

- **depends_on**: [T1, T2]
- **location**:
  - `rust/limux-host-linux/src/control_bridge.rs`
- **description**: Add `pane.create` / `new-pane` to `METHODS`, parse the
  request into a new `ControlCommand::CreatePane`, and include its reply sender
  in `ControlCommand::reply`. Parse workspace targets with `allow_name=true`
  so agent-team peers can address `--workspace claude` or their own
  `LIMUX_WORKSPACE_ID`. Normalize raw and prefixed pane/surface IDs at the
  parser boundary where possible.
- **validation**: Bridge parser unit coverage proves `pane.create` is no
  longer reported as unknown method; invalid direction/type/browser fields fail
  before reaching GTK; raw IDs and `pane:` / `surface:` refs are accepted.
- **status**: Completed
- **log**:
- 2026-05-03: Added `pane.create` to live bridge capabilities and routed
  `pane.create` / `new-pane` into `ControlCommand::CreatePane` with a reply
  sender.
- 2026-05-03: Parser now strips `pane:` and `surface:` prefixes from
  source-pane IDs while continuing to accept raw IDs. Workspace parsing uses
  `allow_name=true` via the shared `parse_create_pane_request` path.
- 2026-05-03: Added bridge route tests proving valid `pane.create` dispatches
  a `CreatePane` command, invalid direction rejects before dispatch, and
  existing parser contract tests cover invalid type/browser/url fields.
- 2026-05-03: Added a temporary GTK handler arm that returns
  `pane.create GTK handler not implemented` so the crate remains exhaustive
  and buildable until T5 wires the real split/create behavior.
- 2026-05-03: Validation passed:
  `cargo test -p limux-host-linux pane_create`;
  `cargo check -p limux-host-linux`.
- **files edited/created**:
  - `agent-self-split-plan.md`
  - `rust/limux-host-linux/src/control_bridge.rs`
  - `rust/limux-host-linux/src/window.rs`

### T4: Extend CLI `new-pane` with source targeting and `--command`

- **depends_on**: [T1]
- **location**:
  - `rust/limux-cli/src/main.rs`
  - `README.md`
- **description**: Add optional `--pane <id|ref>`, `--surface <id|ref>`, and
  `--command <text>` to `run_new_pane`, CLI help, and docs. Make `new-pane`
  agent-friendly by defaulting workspace/surface/pane from `LIMUX_WORKSPACE_ID`,
  `LIMUX_SURFACE_ID`, and `LIMUX_PANE_ID` when flags are absent. Preserve
  current behavior outside a Limux terminal by falling back to the active
  workspace. Do not make command parsing ambiguous with trailing positional
  text; use the existing `parse_opt(args, "--command")` style.
- **validation**: CLI unit test or dry serialization test shows env-derived
  workspace/surface/pane and `--command claude` reach the RPC params; raw and
  prefixed source IDs are serialized unchanged; help text includes the new
  flags. Non-JSON output may keep returning the surface handle, but JSON output
  must expose pane and surface IDs.
- **status**: Completed
- **log**:
  - 2026-05-03: Added `new-pane` source-target serialization for
    `--pane`, `--surface`, and `--command`, with `LIMUX_WORKSPACE_ID`,
    `LIMUX_SURFACE_ID`, and `LIMUX_PANE_ID` defaults when flags are absent.
  - 2026-05-03: Preserved active-workspace fallback outside Limux terminals by
    omitting `workspace_id` when neither `--workspace` nor
    `LIMUX_WORKSPACE_ID` is available.
  - 2026-05-03: Updated CLI help and README examples for agent self-splitting.
  - 2026-05-03: Validation passed:
    `cargo test -p limux-cli`;
    `cargo check -p limux-cli`;
    `cargo run -p limux-cli -- --help`.
- **files edited/created**:
  - `agent-self-split-plan.md`
  - `README.md`
  - `rust/limux-cli/src/main.rs`

### T5: Implement source-aware live GTK pane creation on the main thread

- **depends_on**: [T3]
- **location**:
  - `rust/limux-host-linux/src/window.rs`
  - `rust/limux-host-linux/src/pane.rs`
- **description**: Handle `ControlCommand::CreatePane` inside the existing
  main-loop command handler. Resolve the target pane by explicit source
  surface/pane first, then allowed focus/fallback rules from T2. Reuse
  `split_pane` / `create_pane_for_workspace` instead of adding a separate
  creation path. Return IDs from the new pane's active surface using existing
  surface summary helpers. Persist session state via the same path used by
  manual split actions. The returned `surface_id` is the canonical target for
  any follow-up command injection.
- **validation**: `pane.list` and `surface.list` include the new pane/surface
  immediately after `pane.create`; invalid workspace returns not found; invalid
  source pane/surface returns not found or invalid params consistently.
- **status**: Not Completed
- **log**:
- **files edited/created**:

### T6: Reject browser-pane fields for this terminal self-split feature

- **depends_on**: [T1]
- **location**:
  - `rust/limux-host-linux/src/window.rs`
  - `rust/limux-host-linux/src/pane.rs`
  - `rust/limux-cli/src/main.rs`
- **description**: Keep this feature focused on terminal agent launch. In the
  live GTK bridge, reject `type=browser`, `--url`, and browser+command
  combinations with invalid params. Leave browser split support as a separate
  follow-up so terminal self-spawn is not blocked by browser-tab wiring.
- **validation**: Parser tests show `type=browser`, `--url`, and
  `type=browser --command claude` fail fast in the live bridge contract.
- **status**: Not Completed
- **log**:
- **files edited/created**:

### T7: Launch optional command in the created terminal pane

- **depends_on**: [T5, T4, T6]
- **location**:
  - `rust/limux-host-linux/src/window.rs`
  - `rust/limux-host-linux/src/pane.rs`
- **description**: If `command` is present and the created pane has a terminal
  surface, inject the command plus newline into the exact `surface_id` returned
  by T5. Do not use a broad "first terminal in workspace" lookup. Use bounded
  readiness polling against `terminal_handle_for_surface(..., Some(new_surface_id))`
  or an equivalent exact-surface helper, and fail the RPC if the terminal never
  becomes writable.
- **validation**: Test or smoke confirms `new-pane --command 'printf
  split-ok > /tmp/limux-self-split-proof'` creates the proof file and that the
  returned `surface_id` belongs to the new pane, not the source pane. A second
  smoke command writes `$LIMUX_WORKSPACE_ID`, `$LIMUX_PANE_ID`, and
  `$LIMUX_SURFACE_ID` from inside the new pane and confirms they match the
  `pane.create` response rather than the source pane.
- **status**: Not Completed
- **log**:
- **files edited/created**:

### T8: Add host behavior and end-to-end regression tests

- **depends_on**: [T5, T7]
- **location**:
  - `rust/limux-cli/src/main.rs`
  - `rust/limux-host-linux/src/control_bridge.rs`
  - `rust/limux-host-linux/src/window.rs`
  - `rust/limux-core/src/lib.rs`
- **description**: Add host-level tests and smoke checks that require the
  implementation to exist: explicit source surface/pane targeting, newly
  returned surface ID lookup, command injection into that exact surface, and
  fresh `LIMUX_*` env for the newly spawned pane. Parser and CLI serialization
  tests belong in T3 and T4 respectively, so this task should focus on behavior
  that could not be tested before host implementation. If GTK widget tests are
  impractical, cover pure target-resolution helpers plus a live smoke script.
- **validation**: `cargo test -p limux-cli` and the relevant host/core tests
  pass; failure output would catch a missing bridge route.
- **status**: Not Completed
- **log**:
- **files edited/created**:

### T9: Update roadmap, generated agent instructions, and smoke workflow

- **depends_on**: [T8]
- **location**:
  - `docs/cmux-parity-plan.md`
  - `README.md`
  - `rust/limux-cli/src/main.rs`
  - `scripts/xvfb-smoke-test.sh`
- **description**: Mark `pane.create` and `surface.send_key` accurately in
  the parity plan, add the self-split command to the generated runtime
  `AGENTS.md` template source in `build_agents_md`, and update the smoke script
  to exercise `new-pane --command` if the host can launch. The repo-root
  `AGENTS.md` is contributor guidance and should only change if contributor
  instructions need updating. Keep docs honest that `surface.read_text` remains
  separate work unless it is also implemented. Do not bundle unrelated doc
  claims unless the behavior is validated in this branch.
- **validation**: Docs no longer claim `surface.send_key` is missing; README
  shows an agent self-spawn example; smoke script contains a deterministic
  `new-pane` check.
- **status**: Not Completed
- **log**:
- **files edited/created**:

### T10: Final verification and push decision

- **depends_on**: [T9]
- **location**:
  - repository root
- **description**: Run `cargo fmt`, `cargo check -p limux-host-linux`,
  `cargo test -p limux-cli`, and the canonical `./scripts/check.sh`. Then run
  a live or headless smoke test with exact local binaries, after verifying the
  host process and socket are live. Example:
  `./target/debug/limux-cli --json new-pane --direction right --command 'printf split-ok > /tmp/limux-self-split-proof; printf \"%s\\n%s\\n%s\\n\" \"$LIMUX_WORKSPACE_ID\" \"$LIMUX_PANE_ID\" \"$LIMUX_SURFACE_ID\" > /tmp/limux-self-split-env'`
  followed by `./target/debug/limux-cli list-panels` and a filesystem check for
  `/tmp/limux-self-split-proof` plus env-file values matching the JSON response.
- **validation**: Quality gate is green. Smoke either passes against a live
  host or records a clear environment blocker. Only after this is the feature
  ready to ask for push approval.
- **status**: Not Completed
- **log**:
- **files edited/created**:

## Parallel Execution Groups

| Wave | Tasks | Can Start When |
|------|-------|----------------|
| 1 | T0 | Immediately |
| 2 | T1, T2 | T0 complete |
| 3 | T3, T4, T6 | T1 and T2 complete for T3; T1 complete for T4 and T6 |
| 4 | T5 | T3 complete |
| 5 | T7 | T4, T5, and T6 complete |
| 6 | T8 | T5 and T7 complete |
| 7 | T9 | T8 complete |
| 8 | T10 | T9 complete |

## Testing Strategy

- Unit/contract tests:
  - CLI serialization for `new-pane --command`, `--surface`, `--pane`, and
    env-derived `LIMUX_*` defaults.
  - Bridge parser accepts `pane.create` and rejects invalid inputs.
  - Bridge parser accepts raw source IDs plus `pane:` / `surface:` refs.
  - Bridge parser rejects `type=browser` and `url` for this feature.
  - Existing `limux-core` `pane.create` tests remain green.
- Host behavior checks:
  - `pane.create` creates a real GTK pane via the same codepath as manual
    split buttons and shortcuts.
  - Explicit source surface/pane targeting splits the caller pane, not a
    random focused pane.
  - `list-panes` and `list-panels` expose the created pane/surface.
  - `send --surface <new-surface-id>` targets the new pane.
  - `send-key --surface <new-surface-id>` still works.
  - A command launched in the new pane observes fresh `LIMUX_*` values matching
    the new pane/surface response.
- End-to-end smoke:
  - Start a live Limux host.
  - Verify host/socket liveness before interpreting failures.
  - Run the exact `./target/debug/limux-cli --json new-pane ...` proof command
    from T10.
  - Confirm the new surface exists, the proof file was created, and the env
    file values match the returned JSON.

## Risks & Mitigations

- **GTK thread safety**: Do all widget mutation through the existing
  `ControlCommand` main-loop handler; do not access GTK widgets directly from
  the socket thread.
- **Wrong target pane**: Centralize focused/fallback pane selection and test
  explicit `LIMUX_SURFACE_ID` / `LIMUX_PANE_ID` targeting, workspace-not-found,
  invalid source, and no-focused-pane cases.
- **Command sent before terminal realization**: Use the existing delayed-send
  concept from `workspace.create`, but target only the returned surface ID with
  bounded readiness polling; fail the RPC if the surface never becomes writable.
- **Stale child env**: Verify the new terminal receives fresh `LIMUX_*` values
  matching its own workspace/pane/surface before considering Claude self-spawn
  ready.
- **Browser pane ambiguity**: Reject browser fields now; implement browser
  split as a later feature.
- **Standalone/core drift**: Decide whether `command` is host-only or accepted
  across the core dispatcher, then encode that decision in tests.
- **Local binary ambiguity**: Use `./target/debug/limux-cli` and a known-running
  local host for smoke tests; do not rely on an installed `limux` symlink.
- **Docs overclaiming readiness**: Keep `surface.read_text` listed as open
  until separately routed through the live bridge.
