# metaprompt
Project: “Fix GPT-5” — a mixture-of-agents orchestrator that turns a single user prompt into better results via summarization, parallel candidates, judging, refinement, and verification.
Form factor: One file, CLI, zero setup besides requests (and either OpenAI key or Ollama for local).
Goal: Minimize chat time → maximize runnable MVP + artifacts you can test.

Core Thesis

Treat “GPT-5” as a system (several models + roles), not a single LLM.

Pipeline: nano → 5 candidates → judge → nano prompt-refine → final solution → nano verify (optional repair loop).

Always log everything for auditability and quick iteration.

MVP (already provided as fixgpt5.py)

Providers: openai or ollama (auto-detect; falls back to a deterministic offline stub).

Models: main (gpt-5 or qwen2.5:3b) + nano (gpt-5-nano or qwen2.5:1.5b).

Ops: --op run | selftest | demo

Artifacts (per run, in runs/TS/):

prompt.txt, summary.txt, candidates.json, judge.json, refined_prompt.txt, final_solution.md, verify.json, run.json

Verification: Nano model returns JSON {verdict: PASS|FAIL, reasons, must_fix, nice_to_have}. If FAIL, one repair loop can re-call the main model with must-fixes.

Pipeline (exact steps)

Summarize (nano): compress user prompt into a tight brief.

Generate 5 candidates (main, parallel): styles = Pragmatic, Performance, Security/Privacy, Research, Product.

Judge (main): rank + JSON feedback (schema enforced).

Refine prompt (nano): produce a mega-prompt with intent, constraints, plan, I/O, acceptance tests, and “What NOT to do”.

Final solution (main): produce Markdown-only deliverable.

Verify (nano): PASS/FAIL. If FAIL → optional patch round with must-fix list.

Usage

OpenAI: set OPENAI_API_KEY

python fixgpt5.py --op run --provider openai --model "gpt-5" --nano "gpt-5-nano" --prompt "Implement a fast LRU cache in Python with tests"

Ollama local:

ollama pull qwen2.5:3b && ollama pull qwen2.5:1.5b

python fixgpt5.py --op run --provider ollama --model "qwen2.5:3b" --nano "qwen2.5:1.5b" --prompt "Design a sleep hygiene plan for teens"

Smoke tests:

python fixgpt5.py --op selftest

python fixgpt5.py --op demo

Constraints / Non-goals (for now)

Single-file script; no DB, no web UI, no streaming.

JSON parsing is resilient via a sanitizer; if judge/verifier returns non-JSON, we fallback gracefully.

Production features (auth, telemetry, sandboxing) are out of scope for MVP.

Extension hooks (when ready)

Add --op gui (Tkinter) for toggles + live logs.

Pluggable candidate style list; change or add roles.

Add retry/backoff and per-stage timeouts.

Swap verification for unit tests (execute snippets in a sandbox).

Step-By-Step: Learn & Code This Yourself

The target is one file that actually runs and outputs artifacts. You’ll build it in layers.

0) Prep (5 min)

python -V (3.9+), pip install requests

Optional: brew install ollama && ollama serve (then ollama pull qwen2.5:3b qwen2.5:1.5b)

Create a folder; save the single file as fixgpt5.py

1) Skeleton & CLI

Start with argparse flags:

--op (run|selftest|demo)

--provider (auto|openai|ollama)

--model, --nano, --timeout, --temperature, --candidates, --max-loops, --prompt

Add helpers: timestamp, mkdirp, write_text, write_json.

Checkpoint: running python fixgpt5.py --op demo creates a runs/<TS>/ directory with basic files.

2) LLM Client Abstraction

Build a single LLMClient.chat() that:

routes to OpenAI (chat completions) or Ollama (/api/chat),

returns string,

never raises (fallback to deterministic stub).

Add nano() and main() convenience wrappers.

Checkpoint: try ollama provider with tiny models; ensure you get a response (or stub if down).

3) Prompts & Roles

Define the 5 system prompts already specified:

SUMMARIZER_SYS, CANDIDATE_SYS, JUDGE_SYS (JSON schema), REFINER_SYS, VERIFIER_SYS.

Define CANDIDATE_STYLES list (IDs + style descriptions).

Checkpoint: --op selftest should run end-to-end and write all artifacts.

4) Orchestrator.run()

Implement the 6 stages in order (each writes to disk):

summary.txt via nano.

candidates.json via parallel threads → 5 markdown solutions.

judge.json (ensure valid JSON via sanitizer/fallback).

refined_prompt.txt (nano mega-prompt).

final_solution.md (main model).

verify.json (nano). If FAIL and --max-loops > 0, apply must-fix patch round and re-run step 5→6.

Checkpoint: final_solution.md is non-empty; verify.json.verdict exists.

5) Logging & Scoreboard

Print a short console digest:

Rankings from judge

Judge overall feedback

Verifier JSON + number of patch rounds

Checkpoint: You can scan console + artifacts quickly after each run.

6) Quick Test Plan (no external libs)

File existence: assert all artifact paths exist after --op selftest.

JSON well-formed: load judge.json, verify.json.

Deterministic fallback: simulate missing API / stopped Ollama; confirm stub still writes all outputs.

Run:

python fixgpt5.py --op selftest
python fixgpt5.py --op run --prompt "Build a CLI todo app in Python"

Acceptance Criteria (MVP is “done” when…)

python fixgpt5.py --op selftest succeeds and produces:

prompt.txt, summary.txt, candidates.json, judge.json, refined_prompt.txt, final_solution.md, verify.json, run.json.

verify.json.verdict is "PASS" or "FAIL" with reasons (never empty/invalid).

No crash even without internet or model servers (stub kicks in).

One file, one command to use.

Minimal “Cheat Sheet” You Can Paste Anywhere

How to run (local models):

ollama pull qwen2.5:3b
ollama pull qwen2.5:1.5b
python fixgpt5.py --op run --provider ollama --model "qwen2.5:3b" --nano "qwen2.5:1.5b" --prompt "Generate a Python LRU cache with tests"


How to run (OpenAI):

export OPENAI_API_KEY=sk-...
python fixgpt5.py --op run --provider openai --model "gpt-5" --nano "gpt-5-nano" --prompt "Draft a product spec for a focus timer app"


Artifacts appear in: runs/YYYYMMDD_HHMMSS/

If you want, next step I can fold in a tiny --op gui (same file) with Start/Stop + toggles (models, candidates, auto-patch) and live log pane—say the word and I’ll drop it in.
