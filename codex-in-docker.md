# Plan: Codex in Docker (Codex Worker)

## 1) Objective
Create a Docker-based “Codex worker” that:
- bind-mounts a host folder (e.g. `…/workspace/openclaw-dev`) as `/work`
- clones a repo URL provided at runtime (message/env)
- runs Codex CLI (`codex exec …`) in that repo
- persists Codex authentication/config via a mounted directory (preferred: a **dedicated bot auth** dir)

Primary use-case: run coding tasks in a contained environment while keeping artifacts visible to the main OpenClaw agent via the shared bind mount.

## 2) Mature alternative worth evaluating: DeepBlueDynamics/codex-container ("Gnosis Container")

Repo: https://github.com/DeepBlueDynamics/codex-container

This looks like a feature-complete, batteries-included Codex-in-container setup that overlaps heavily with what we’re designing here.

Notable capabilities (from repo description/readme):
- Codex CLI automation inside Docker with **session management** (resume, recordings)
- **Cron/self-scheduling** (Codex can create its own triggers)
- **File watcher** workflows (drop a file → triggers a Codex run)
- **HTTP API** mode (e.g. `/completion` endpoint) for remote triggers
- **Security tiers** (sandboxed → “danger” → “privileged” style modes)
- **Auth management** (`-Login` flow; persists credentials)
- Optional **local model** support via Ollama
- Large tool surface (claims 275+ MCP tools: Gmail/Calendar/Slack/etc.)

Implications for this plan:
- We can likely **use it directly** (fastest path), or **adapt its patterns** (mounts, auth persistence, TTY support, gateway/API wrapping).
- If we proceed with our own “Codex worker”, we should at least benchmark against this for: UX, security posture, and maintenance burden.
- Follow-ups: check license, update cadence, how it persists secrets, and whether the API mode is compatible with our trigger pipeline.

---

## 3) High-level design
### Runtime mounts
- `/work`  ← bind mount to host `openclaw-dev/`
- `/home/codex/.codex` ← bind mount to a persisted auth dir (recommended)

### Inputs (runtime contract)
- `REPO_URL` (required)
- `REPO_DIR` (optional, default: `repo`) — where under `/work` to place it
- `REPO_REF` (optional) — branch/tag/sha to checkout
- `PROMPT` (required)
- `CODEX_MODE` (optional):
  - `exec` (default) → `codex exec ...`
  - later: `review`, `chat`, etc.
- `CODEX_FLAGS` (optional) — e.g. `--full-auto`

### Outputs
Write everything into `/work/out/` (host-visible):
- `out/run.log` (stdout/stderr)
- `out/summary.md` (human-readable result)
- `out/exit.json` (exit code, timestamps, repo info)

## 3) Auth strategy (recommended)
### Option A (recommended): dedicated bot auth dir
- Create: `openclaw-dev/.codex-bot/` on host
- Mount it into container as `/home/codex/.codex`
- First run: do `codex auth login` inside container once (interactive)
- Subsequent runs reuse the mounted auth.

### Option B: mount host user’s real `~/.codex`
- Fastest, but the container gains access to the user’s Codex credentials.
- Only do this with a trusted image; document the risk.

## 4) Container user / permissions
- Run as non-root `codex`.
- Prefer passing host UID/GID so files created in `/work` are not root-owned.
- Example pattern:
  - `docker run --user $(id -u):$(id -g) ...`
  - Ensure `$HOME` points to a writable home (may need `-e HOME=/tmp/home` or create a home path that matches UID).

## 5) TTY / PTY requirement
Codex behaves like an interactive terminal app.
- Use `docker run -it …` for interactive commands (including `auth login`).
- For non-interactive runs, TTY still helps; document that the worker expects a TTY.

## 6) Git safety / trust
Codex can require a “trusted git directory”.
- Ensure repo is a git repo.
- Prefer cloning inside `/work/<repo_dir>`.
- Optionally set git safe directory: `git config --global --add safe.directory /work/<repo_dir>`

## 7) Execution flow (entrypoint)
Pseudo-flow:
1. Validate `REPO_URL` and `PROMPT`
2. Ensure `/work/out` exists
3. Clone or update repo into `/work/$REPO_DIR`
4. Checkout `REPO_REF` if provided
5. Run codex:
   - `codex exec $CODEX_FLAGS "$PROMPT"`
6. Capture logs and write `summary.md` + `exit.json`

## 8) Suggested commands (examples)
### First-time auth (interactive)
```bash
docker run --rm -it \
  -v /ABS/PATH/openclaw-dev:/work \
  -v /ABS/PATH/openclaw-dev/.codex-bot:/home/codex/.codex \
  -e HOME=/home/codex \
  codex-worker:latest \
  bash -lc 'codex auth login'
```

### One-shot run
```bash
docker run --rm -it \
  -v /ABS/PATH/openclaw-dev:/work \
  -v /ABS/PATH/openclaw-dev/.codex-bot:/home/codex/.codex \
  -e HOME=/home/codex \
  -e REPO_URL=https://github.com/deadronos/repohub \
  -e PROMPT='Run tests and summarize the project structure.' \
  -e CODEX_FLAGS='--full-auto' \
  codex-worker:latest
```

## 9) Milestones
1. **M0:** Decide whether to use existing Codex installation vs install in image.
2. **M1:** Build image + entrypoint that clones repo + echoes prompt.
3. **M2:** Add Codex invocation + log capture.
4. **M3:** Auth persistence via mounted `.codex-bot` verified.
5. **M4:** Polish: outputs, error handling, docs, security notes.

## 10) Open questions
- Which base image do we want (debian-slim vs node-based)?
- Do we want the worker to support multiple providers/agents (Codex/Claude/OpenCode), or Codex-only first?
- Should we enforce an allowlist for repos/domains?
