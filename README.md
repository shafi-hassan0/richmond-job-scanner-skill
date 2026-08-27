# Richmond Job Scanner

Tracks QA and Software Engineering job openings at a fixed list of Richmond, VA-area
employers. Used by a Claude Code scheduled cloud routine that runs daily, diffs newly
posted roles against `state/seen_postings.json`, and reports new matches by opening/
updating a tracking issue in this repo (which sends the owner a GitHub notification).

- `SKILL.md` — the scan logic and matching criteria (also installed locally as a
  Claude Code skill at `~/.claude/skills/richmond-job-scanner` for on-demand runs).
- `references/companies.json` — the watch list. Edit to add/remove companies.
- `state/seen_postings.json` — last-seen postings per company, used to avoid
  re-reporting the same opening every day. Updated automatically by each run.

This repo's copy of the state file is updated by the scheduled cloud routine.
The local copy under `~/.claude/skills/richmond-job-scanner/state/` is updated by
manual/on-demand runs of the skill and is tracked separately.
