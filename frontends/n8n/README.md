# n8n submit form — a web front-end for the bullpen

> **Retired 2026-07-01.** The reference deployment (LXC 113, `/form/bullpen`) was torn
> down — the form was a demonstration, and the standing dispatch paths are `cr-submit`
> and direct NAS-inbox writes. Everything below is kept as a working recipe: the
> workflow JSON and compose file still deploy as described if a form front-end is
> ever wanted again.

A point-and-click way to drop a job onto the fleet. It's just another **producer**:
the form writes a job file into the NAS `inbox/` exactly like `cr-submit` does — same
spec format, same write-then-rename — so any idle worker picks it up and opens a PR. No
SSH, no coupling to a specific runner; if every worker is down, the job waits in the
queue until one comes up.

> Built with Claude (Claude Code) at Chris's request, 2026-06.

```
  browser form  ──►  n8n  ──►  write *.partial → rename  ──►  NAS inbox  ──►  a worker claims it
  (project,                    into /srv/jobs/inbox/                        (same path as cr-submit)
   model, task)
```

The workflow (`bullpen-submit-form.json`) is three nodes:

| Node | Does |
|---|---|
| **Submit a Bullpen Job** (Form Trigger) | Serves the form: Project dropdown (the 13 registry projects + ad-hoc), Model dropdown, Task box |
| **Write job to inbox** (Code) | Builds the spec and writes it with `fs` (`writeFileSync` to `*.partial`, then `renameSync` — the same atomic publish the invariant requires) |
| **Show result** (Form completion) | Shows the queued path back to the submitter |

**Spec contract** (kept identical to `cr-submit`): a project and/or model selection →
a `.json` spec `{prompt, project?, model?}`; otherwise a plain `.txt` (the whole file is
the prompt). Filenames are `<ts>-<project|adhoc>-n8n-<rand>.<ext>` so they never collide,
and the producer writes `*.partial` then renames into place so the poller never reads a
half-written file.

## Where it runs

An n8n instance in its own unprivileged LXC, deployed via Docker. The one requirement
that makes the direct-write work: **the same NAS bind-mount a worker has** —
`claude-jobs` → `/srv/jobs` on the LXC, then `/srv/jobs:/srv/jobs` into the container
(see `docker-compose.yml`). Because the CIFS mount forces `uid/gid 101000, mode 0770`
and the n8n LXC is unprivileged with the same uid base as the workers, the container's
`node` user (uid 1000 → host 101000) *is* the inbox owner and can write.

Reference instance: LXC 113 `n8n` on pve4, `http://192.168.139.13:5678`, form at
`/form/bullpen`.

## Deploy

1. **LXC** — unprivileged Debian, `features: nesting=1,keyctl=1`, plus the NAS mount
   (identical to a worker's):
   ```
   mp0: /mnt/<host-cifs-mount>/claude-jobs,mp=/srv/jobs
   ```
   Install Docker (`curl -fsSL https://get.docker.com | sh`).

2. **Compose** — drop `docker-compose.yml` at `/opt/n8n/` and `docker compose up -d`.
   `NODE_FUNCTION_ALLOW_BUILTIN=fs` lets the Code node use `fs` for the atomic write;
   `N8N_SECURE_COOKIE=false` lets the editor work over plain LAN HTTP.

3. **Import + activate the workflow** (CLI, no account needed):
   ```bash
   docker cp bullpen-submit-form.json n8n:/tmp/wf.json
   docker exec n8n n8n import:workflow --input=/tmp/wf.json
   docker exec n8n n8n publish:workflow --id=bullpenSubmitForm1
   docker compose restart        # required — registers the form webhook
   ```
   The form is then live at `http://<host>:5678/form/bullpen`.

## Updating the form

Edit in the n8n UI and re-export over `bullpen-submit-form.json`, **or** edit the JSON
and re-import. The workflow keeps a fixed `id` (`bullpenSubmitForm1`) so re-import
overwrites in place. After any import: `publish:workflow` then restart the container.
If a workflow gets wedged (active with a node type the running n8n can't load),
`unpublish:workflow --id=…`, restart, then re-import the fixed version.

## Why the Code node writes the file (and not a file node)

The obvious build — *Convert to File → Read/Write Files* — works, but n8n's file-write
node can't atomically rename (so it'd write the final name directly, violating
write-then-rename), and its path guard (`isFilePathBlocked`) rejects `/srv/jobs` with a
misleading *"file is not writable"* unless you also set `N8N_RESTRICT_FILE_ACCESS_TO`.
The **Execute Command** node (which could `mv`) is disabled by default in current n8n
("Unrecognized node type"). Doing the write in the Code node with `fs.writeFileSync` +
`fs.renameSync` is fewer nodes, needs no file-access env vars, and gives the atomic
publish the fleet requires.

## Gotchas

- **`NODE_FUNCTION_ALLOW_BUILTIN=fs` is required** — without it the Code node can't
  `require('fs')`. (Single-tenant instance; this lets any Code node touch the filesystem.)
- **Form submits are multipart/form-data.** A plain JSON POST returns
  "Expected multipart/form-data" — the rendered form does the right thing; only matters
  if you script a submit.
- **Container `node` writes, container `root` can't** (inbox is `0770`, owned by uid 1000)
  — verify writes with `docker exec n8n …` (default user), not `-u root`.

## Security

The form is **unauthenticated** — anyone who can reach it can queue jobs, and jobs spend
tokens. It's intended for a trusted LAN. To lock it down, set the Form Trigger's
`authentication` to `basicAuth` and attach an `httpBasicAuth` credential.
