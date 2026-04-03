# HANDOFF.md — Hermes Fork Recovery & Resilience Fix

**Date**: 2026-04-03 14:46 UTC
**Session**: Recovery from Telegram gateway failure caused by botched cron update
**Status**: ✅ RESOLVED — Gateway operational, automation hardened

---

## What Broke

Telegram gateway threw `ModuleNotFoundError: No module named 'prompt_toolkit'` after today's cron runs.

**Root cause chain**:
1. Cron runs with minimal PATH (`/usr/bin:/bin`) — `uv` at `~/.local/bin/uv` was not found
2. Update script used bare `uv` command → `uv: command not found`
3. Script's error handler ran `git merge --abort` to "roll back" — but merge was already committed, so abort was a no-op
4. Dependencies were NEVER reinstalled after upstream changes
5. 12:00 cron run hit a real conflict in `tools/transcription_tools.py` → repo stuck mid-merge with 27 modified files
6. Gateway loaded new code from incomplete merge but venv was stale → `prompt_toolkit` missing

---

## What Was Fixed

### 1. Completed the Stuck Merge
- Resolved conflict in `tools/transcription_tools.py` (upstream refactored `_has_openai_audio_backend()` — accepted upstream version)
- Committed as `c721b3a1`

### 2. Rewrote Update Script (`~/.hermes/scripts/update-hermes-fork.sh`)
**Before** (fragile):
- Used bare `uv` command → fails in cron
- No PATH resolution
- `git merge --abort` on dep failure even after merge committed (destructive no-op)
- No fallback if `uv` missing
- No mid-merge state detection
- No venv existence check
- `hermes CLI` calls without PATH check

**After** (resilient):
- Multi-path `uv` resolution: `~/.local/bin/uv` → `/usr/local/bin/uv` → `/usr/bin/uv` → `command -v uv`
- **pip fallback**: if `uv` not found, uses `.venv/bin/pip`
- Mid-merge detection: checks for `.git/MERGE_HEAD` before starting
- No destructive rollback: merge is already committed by the time dep install runs — warns instead of aborting
- Venv existence verification before install
- `hermes CLI` calls guarded with `command -v hermes`
- Gateway restart has systemctl fallback if `hermes gateway restart` fails
- Push failures don't abort — update itself succeeded
- Submodule failures are warnings, not fatal

### 3. Reinstalled Dependencies
```bash
/home/fabio/.local/bin/uv pip install --python /home/fabio/.hermes/hermes-agent/.venv/bin/python -e ".[all,messaging]"
```
- `prompt_toolkit` restored (v3.0.52)
- All upstream dependencies synced

### 4. Gateway Restarted & Verified
- `systemctl --user restart hermes-gateway.service`
- Status: **active (running)** since 14:46:11 WEST
- Telegram connected (fallback IP: 149.154.167.220)

### 5. Pushed to Origin
- All commits pushed to `https://github.com/fabiofalopes/hermes-agent.git`
- Branch: `main`, up to date with `origin/main`

---

## Current State

### Git
```
Branch: main
Tracking: origin/main (up to date)
Origin: https://github.com/fabiofalopes/hermes-agent.git
Upstream: https://github.com/NousResearch/hermes-agent.git
Latest: 358f0e30 (merge of origin changes)
Untracked: FINAL-INSTRUCTIONS.md, proxmox_mcp.log (not repo-related)
```

### Dependencies
- Venv: `/home/fabio/.hermes/hermes-agent/.venv/`
- Python: 3.13
- `prompt_toolkit`: 3.0.52 ✅
- All `[all,messaging]` extras installed ✅

### Gateway
- Service: `hermes-gateway.service`
- Status: active (running)
- PID: 3065312
- Memory: ~74MB
- Telegram: connected

### Cron
```cron
0 */6 * * * /home/fabio/.hermes/scripts/update-hermes-fork.sh >> /home/fabio/.hermes/logs/update.log 2>&1
0 9 * * 1 /home/fabio/.hermes/scripts/weekly-model-check.sh >> /home/fabio/.hermes/logs/model-health.log 2>&1
```
Next run: 18:00 UTC today

### Update Script
- Location: `/home/fabio/.hermes/scripts/update-hermes-fork.sh`
- Size: ~340 lines
- Executable: ✅
- Features: multi-path uv, pip fallback, mid-merge detection, no destructive rollback, graceful degradation

---

## Lessons Learned / Anti-Fragility Measures

| Failure Mode | Prevention |
|---|---|
| `uv` not found in cron | Multi-path resolution + pip fallback |
| Mid-merge state | Pre-flight check for `.git/MERGE_HEAD` |
| Destructive rollback after committed merge | Script no longer calls `git merge --abort` after merge completes |
| Stale dependencies | Venv existence check before install |
| Gateway silent failure | Health check after restart, systemctl fallback |
| Push failure aborts update | Push is non-fatal — logs warning and continues |
| `hermes CLI` not in PATH | All CLI calls guarded with `command -v` |
| Submodule failure blocks update | Submodule errors are warnings |

---

## Known Issues / Watch Items

1. **Conflict in `tools/transcription_tools.py`**: This file changes frequently upstream. Our voice note chunking customization lives here. Next upstream change to this file will likely conflict again. Consider:
   - Moving our chunking logic to a separate file/module
   - Contributing chunking upstream as a PR
   - Using a post-merge hook to re-apply our patch

2. **`uv` location**: Currently at `~/.local/bin/uv`. If system updates or `uv` is reinstalled elsewhere, the script will find it via the multi-path resolution.

3. **Untracked files**: `FINAL-INSTRUCTIONS.md` and `proxmox_mcp.log` in repo root — clean these up or add to `.gitignore`.

---

## Next Session Priorities

1. **Test the fixed script**: Run `~/.hermes/scripts/update-hermes-fork.sh --dry-run` to verify
2. **Monitor next cron run** (18:00 UTC today) — check `/home/fabio/.hermes/logs/update-*.log`
3. **Clean up untracked files**: Decide what to do with `FINAL-INSTRUCTIONS.md` and `proxmox_mcp.log`
4. **Voice chunking strategy**: Decide whether to keep in `transcription_tools.py` (conflict-prone) or extract to separate module
5. **Feature adoption**: Review upstream changes since last merge — check `git log --oneline` for new features

---

## Key File Locations

| File | Purpose |
|---|---|
| `~/.hermes/scripts/update-hermes-fork.sh` | Resilient update automation |
| `~/.hermes/scripts/setup-update-cron.sh` | Cron installer |
| `~/.hermes/logs/update-*.log` | Timestamped update logs |
| `~/.hermes/config.yaml` | Agent configuration |
| `~/.hermes/.env` | API keys |
| `~/.config/systemd/user/hermes-gateway.service` | Gateway service |
| `~/.git-credentials` | Git PAT storage |
| `tools/transcription_tools.py` | Voice note chunking (our customization) |

---

## Quick Commands

```bash
# Check gateway
systemctl --user status hermes-gateway.service

# Check last update log
ls -lt ~/.hermes/logs/update-*.log | head -1 | awk '{print $NF}' | xargs tail -30

# Manual update (safe)
~/.hermes/scripts/update-hermes-fork.sh --dry-run
~/.hermes/scripts/update-hermes-fork.sh --verbose

# Check for upstream changes
cd ~/.hermes/hermes-agent && git fetch upstream && git log --oneline HEAD..upstream/main

# Reinstall dependencies manually
~/.local/bin/uv pip install --python ~/.hermes/hermes-agent/.venv/bin/python -e ".[all,messaging]"

# Restart gateway
systemctl --user restart hermes-gateway.service
```
