# Dev workspace troubleshooting

Failure catalog for `paperclip-dev-workspace-run-verify-fix`. Read the matching
entry when a verification probe fails; the success conditions and hard rules
in `../SKILL.md` still apply.

## Common failures and fixes

### Service URL serves a different workspace (port squat)

Symptom: the workspace URL loads a working Paperclip app, but it is a sibling
worktree's app — wrong branch, wrong data, or the UI keeps bouncing you into
another workspace's pages.

Likely cause: the pinned service port is owned by another workspace's process
and start-time adoption attached that process to this workspace's runtime row.
Health checks pass because the sibling app is genuinely healthy.

Fix: follow the owner-first procedure in SKILL.md "Port conflicts and
workspace identity": identify and stop the sibling service or supervising run, verify
the port stays free, then start or restart the target service. Re-verify that
`/proc/<pid>/cwd` of the new port owner resolves inside the target worktree.

Verify: identity check passes, and one workspace-specific record served
through the URL (branch, issue key, or workspace id) matches the target
workspace.

### Recorded port drifts after an ad hoc start

Symptom: the runtime row and the actual listener disagree about the port, or a
previously pinned port changed after someone started the app by hand.

Likely cause: an unmanaged `pnpm dev` (for example with `--bind lan`) picks its
own port instead of using the managed pinned port.

Fix: stop the unmanaged process, then managed start. Verify the recorded URL
port equals the bound port and the identity check passes.

### Setup gate appears

Symptom: the page says no admin has claimed the instance.

Fix: run a full worktree reseed from the primary instance, then restart the
managed service. Claiming a first admin can clear the gate, but if the user
asked for the normal isolated workspace database, full reseed is the correct
fix.

Verify: `/api/health` has `bootstrapStatus: ready`, login works, and populated
data exists.

### Login fails

Symptom: bootstrap is ready, but the user's normal dev credentials do not work.

Likely cause: the isolated DB has roles or bootstrap state but lacks the
primary instance auth users/accounts.

Fix: full reseed. Do not manually copy only Better Auth user/account rows as
the final fix.

Verify: user login through browser/QA and `/api/agents/me` with the agent key.

### Login works but data is missing

Symptom: user can sign in, but companies/issues/projects/runs are empty or
clearly incomplete.

Likely cause: a partial auth repair was done instead of a full cloned database.

Fix: full reseed, managed restart, then verify representative cloned records.

### Port listens but health says database unreachable

Symptom: `curl -I /` returns a response, but `/api/health` reports
`database_unreachable`.

Likely cause: stale Node/web process remained alive after embedded Postgres
died.

Fix: managed stop first. If the process survives, identify the matching
Paperclip dev-runner process group for the target port and terminate only that
group. Then managed start.

Verify: `/api/health` is ok after a stability wait and the runtime record is
healthy.

### Main control plane loses track of a running service

Symptom: the service URL works, but the main app says the service was not
created or is stopped.

Likely cause: detached workaround process, stale provider ref, or service
adoption trusted the root URL instead of health.

Fix: shut down the unmanaged process and restart through the managed runtime.
If code repair is needed, ensure adoption checks `/api/health`, replaces
unhealthy adopted processes, and records the current provider ref.

Verify: main runtime row and `/api/health` agree.

### Served app loses track after full reseed

Symptom: the main app sees `paperclip-dev` running, but the cloned app copied a
runtime-service row whose id does not match the local registry.

Likely cause: normal DB clone copied persisted runtime rows from the primary
instance into an isolated environment with different local process metadata.

Fix: use managed start/adoption again. If code repair is needed, adoption
should reconcile by service identity and port, not only by copied row id.

Verify: main app and served app both show the same service as
`running` / `healthy`.

### Runtime start returns permission or run-FK errors in the cloned app

Symptom: starting from the served app returns `403` or a mutation partially
applies before activity logging fails.

Likely causes: the cloned issue/run/agent state does not match the current
heartbeat, or the run id is absent in the cloned database after reseed.

Fix: prefer the main control-plane managed runtime path and full reseed. If
the cloned app state itself must be repaired, use normal Paperclip issue/run
transitions first. Do not hide the condition with raw DB edits; report the
exact guard or missing row if it blocks the normal path.

## Recurring gotchas from past repairs

Each of these has burned a previous repair; check them before deep debugging.

- Worktree instance logs live at
  `~/.paperclip-worktrees/instances/<slug>/logs`. Read them before guessing at
  login or startup failures.
- Probing the service through a LAN or tailscale IP can return `403` while
  `127.0.0.1` (or the `paperclip-dev` hostname) works. Probe loopback first
  before concluding auth is broken.
- Browser QA that lands on a blank company path (for example `/FOR/...`)
  should switch to a company key that exists in the cloned data (for example
  `/PAP/...`) before concluding the reseed failed.
- A `409` on runtime or issue mutations usually means another live run holds
  the ownership lock. Do not force it; wait or coordinate through the issue.
- After a full reseed the cloned DB can contain copied runtime-service rows
  that do not match local processes. Re-run managed start and the identity
  check instead of trusting copied rows.
