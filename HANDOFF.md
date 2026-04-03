# Hermes Agent — Handoff for Next Session

**Date**: 2026-04-03 02:50 UTC  
**Previous Session**: Fork setup, automation, community research  
**Next Session Focus**: Maintenance, monitoring, feature adoption

---

## Current State

### Fork: OPERATIONAL ✅
| Item | Value |
|------|-------|
| Fork | https://github.com/fabiofalopes/hermes-agent.git |
| Origin | `https://github.com/fabiofalopes/hermes-agent.git` |
| Upstream | `https://github.com/NousResearch/hermes-agent.git` |
| Local repo | `/home/fabio/.hermes/hermes-agent/` |
| Ahead of upstream | 3 commits |
| Behind upstream | 0 commits (fully synced) |

### Authentication
- PAT stored in `~/.git-credentials` (git credential helper: `store`)
- Works for all git operations including non-interactive cron

### Automation
- **Update script**: `/home/fabio/.hermes/scripts/update-hermes-fork.sh`
- **Cron**: `0 */6 * * *` (every 6 hours)
- **Logs**: `/home/fabio/.hermes/logs/`

---

## Customizations

### Voice Note Chunking (committed as `954d6b0d`)
- File: `tools/transcription_tools.py`
- Integration: GroqWhisperClient from `/home/fabio/voice_note_repo/src/api/groq_client.py`
- Pattern: Dispatcher with `_transcribe_groq_chunked()` and `_transcribe_groq_direct()`

### Config Files (NOT in git — survive updates)
- `/home/fabio/.hermes/config.yaml`
- `/home/fabio/.hermes/.env`

---

## Key Commands

```bash
cd /home/fabio/.hermes/hermes-agent

# Status check
git status && git log --oneline -5 && git remote -v

# Manual update
/home/fabio/.hermes/scripts/update-hermes-fork.sh --verbose

# Check logs
ls -la /home/fabio/.hermes/logs/ && tail -50 /home/fabio/.hermes/logs/update.log

# Gateway
hermes gateway status && hermes gateway restart

# Cron
crontab -l
```

---

## Next Session Priorities

### Immediate
1. Check if first automated cron update ran successfully (check `/home/fabio/.hermes/logs/`)
2. Review any upstream changes since last merge
3. Verify gateway stability after any updates

### Short Term
1. Create feature adoption decision framework
2. Monitor key issues: #3943 (MemoryProvider), #502 (Project Context), #157 (Multi-Model Routing)
3. Consider contributing voice note chunking upstream as PR

### Ongoing
1. Maintain fork sync — review upstream changes before merging
2. Anti-bloat: adopt interfaces, not implementations
3. Monitor community direction

---

## Mental Model

- **Origin** = Your fork (what you run)
- **Upstream** = NousResearch (source of updates)
- **Your changes** = Committed to fork, preserved through merges
- **Updates** = Merge upstream → resolve conflicts once → Git remembers
- **Selective adoption** = Review before merging, don't blindly accept

---

## Documentation

All session docs in Obsidian vault: `hermes-agent/` directory
- `HANDOFF-2026-04-03.md` — Full session handoff (detailed)
- `FORK-COMPLETE.md` — Fork status
- `COMMUNITY-PRIORITIES-2026-04.md` — Community research
- `MENTAL-MODEL.md` — Fork philosophy
- `CUSTOMIZATIONS-INVENTORY.md` — All customizations
- `INDEX.md` — Navigation hub

---

## What to Watch For

- Update script failures → check logs
- Merge conflicts → script stops, resolve manually, re-run
- Upstream breaking changes → review before merging
- Gateway stability after updates → `hermes gateway status`
