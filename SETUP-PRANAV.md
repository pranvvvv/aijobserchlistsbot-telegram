# Why this repo exists — and how it actually helps me

This is my personal, configured instance of
[tarunlnmiit/autopilot-jobhunt](https://github.com/tarunlnmiit/autopilot-jobhunt),
an open-source AI job agent. I'm Shiva Pranav — B.Tech CSE student (graduating
Dec 2028), founder at codebrain.ai, looking for **remote-only** roles and
remote internships at AI/GenAI startups. No UK or US work authorisation, so
most "apply on our careers page" postings that demand local citizenship are a
hard no for me before I even open them.

## The actual problem this solves

Manually checking 140 company careers pages for new postings, reading each
one, and judging whether it's worth 20 minutes writing a cover letter for —
that doesn't scale, and it's exactly the kind of repetitive triage that
burns an evening without moving the job search forward. This tool automates
that loop:

1. **Discovery** — hits 140 real company careers pages (Mistral AI, Hugging
   Face, GitLab, Monzo, Anthropic-adjacent EU/remote-friendly startups, and
   more) via the TinyFish API and pulls every open role.
2. **Scoring** — an LLM reads my *actual resume* (`resume/Shiva_Pranav_Resume.md`,
   built from my real profile — no fabricated experience) against each job's
   full description and gives a 0–100 fit score with a one-line reason. Not
   keyword matching — it understands that "Senior Staff ML Engineer, 10 YOE"
   is a bad fit for someone with 1.9 years of experience, even if every
   keyword matches.
3. **Alerting** — top matches land in my Telegram, so I see new roles the
   morning after a scan runs, not by remembering to go check a dashboard.
4. **Drafting** — `autopilot draft #N` writes a tailored cover letter and
   resume bullets for a specific posting in under a minute, using my real
   background, not a generic template.

It never applies on its own — every draft is reviewed by me before anything
goes out. See `PRIVACY.md` (from upstream) for exactly what leaves my machine.

## What's configured here specifically for me

- `config.json` (gitignored, not in this repo) — my real candidate profile:
  remote-only, no UK/US work authorisation, target titles (AI/ML/GenAI
  engineer, frontend, full-stack), `min_score: 55`, `top_n: 10`.
- `resume/Shiva_Pranav_Resume.md` (gitignored) — built from my actual
  `CLAUDE.md` candidate profile in the `ai-job-search` repo, not invented.
- Telegram alerts wired to my own bot.
- `llm_provider` — see the fix below; day-to-day I run scans through Claude
  Code's own CLI auth (`claude_cli`) rather than OpenRouter.

## A real bug I hit and fixed (and why it's in this fork)

On Windows, `claude` resolves to a `claude.cmd` npm shim, not a `.exe`.
`subprocess.run(["claude", ...])` without `shell=True` calls Windows'
`CreateProcess` directly, which can't execute `.cmd` files — every
`claude_cli`-scored job failed instantly with "claude binary not found in
PATH", even though `claude --print "hi"` worked fine from a normal terminal.
Fixed in `job_hunt/llm_utils.py` by routing through the shell on Windows only
(POSIX behavior is unchanged). Worth upstreaming if `tarunlnmiit/autopilot-jobhunt`
doesn't already have a fix for this.

## A real constraint worth knowing before relying on this nightly

OpenRouter's free tier is **50 requests/day per account**, not per model —
easy to burn through in one scan if the configured free model IDs have gone
stale (two of the four default fallback models — `meta-llama/llama-3.3-70b-instruct:free`
and `qwen/qwen3-coder:free` — had already been deprecated by OpenRouter when
I set this up; `openrouter_fallback_models` in `config.json` now points at
models confirmed live against the `/api/v1/models` endpoint). For a true
unattended nightly cron, either accept 50 scored jobs/day free, pay
OpenRouter's one-time $10 top-up for 1000/day, or run occasional on-demand
scans through `claude_cli` (works, but each call spends ~25–30k tokens of my
Claude subscription's context, so it's for periodic real passes, not a
nightly job).

## A real market pattern, not a bug: senior-biased startups vs. a junior candidate

Two full scans (140 upstream companies, then a curated 41-company remote-friendly
list) both came back with **0 matches at `min_score: 55`** — verified via
`scan.log`'s DEBUG output that this wasn't a scoring bug: the LLM correctly
scored dozens of genuinely on-site/hybrid Mistral AI roles at 10–40 with
accurate location reasoning, and correctly scored Supabase/Modal postings low
because what those specific companies had open *right now* skewed almost
entirely senior/staff (Postgres internals, Rust/C systems roles, $150K–$350K,
several explicitly US-onsite). Small remote-first startups tend to hire
senior engineers who need no ramp-up — precisely because they're too lean to
mentor juniors. That's a real pattern in the current market, not something to
route around by picking different companies.

Lowered `min_score` to **40** in `config.json` (gitignored, not shown here) to
surface "partial fit, worth trying" roles instead of nothing, matching the
tool's own scoring guide (40–59 = "apply if pipeline is thin" — true here,
pipeline is empty). Also worth knowing: job **boards** (LinkedIn, Remotive,
RemoteOK — see the separate `ai-job-search` repo) surfaced genuine intern-titled
real postings (e.g. "AI Engineer Intern", "GenAI Intern") at zero LLM cost,
because they aggregate across thousands of companies rather than a hand-picked
few — structurally better suited to entry-level search than this tool's
company-by-company careers-page model, which shines more for someone senior
targeting specific employers.

## Day to day

```bash
autopilot scan              # find + score new postings
autopilot draft #1          # tailored cover letter + resume bullets for match #1
autopilot export --min 60   # CSV of everything scoring 60+
```
