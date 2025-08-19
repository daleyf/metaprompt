# Fix GPT-5
A mixture-of-agents orchestrator that treats GPT-5 as a system, not a single LLM.

## TL;DR
Pipeline: summarize → 5×candidates → judge → refine → final → verify (+optional repair)
Providers: OpenAI (cloud), Ollama (local), offline stub
Artifacts: everything logged to runs/<timestamp>/

## Quickstart

OpenAI (cloud):
export OPENAI_API_KEY=sk-...
python fixgpt5.py --op run --provider openai
--model "gpt-5" --nano "gpt-5-nano"
--prompt "Implement a fast LRU cache in Python with tests"

Ollama (local):
ollama pull qwen2.5:3b qwen2.5:1.5b
python fixgpt5.py --op run --provider ollama
--model "qwen2.5:3b" --nano "qwen2.5:1.5b"
--prompt "Design a sleep hygiene plan for teens"

## Smoke tests / demo:
python fixgpt5.py --op selftest
python fixgpt5.py --op demo

## Artifacts (per run):
prompt.txt, summary.txt, candidates.json, judge.json, refined_prompt.txt, final_solution.md, verify.json, run.json

## How Meta-Prompting Works (inside the orchestrator)

Roles & sequence:

Summarizer (nano): compresses the user prompt for lossless intent.

Candidate generators (main, 5 styles in parallel): Pragmatic, Performance, Security/Privacy, Research, Product.

Judge (main): ranks candidates and emits structured feedback (JSON).

Refiner (nano): builds a “mega-prompt” that fuses user intent + best candidate + judge feedback.

Final (main): produces the deliverable (Markdown).

Verifier (nano): returns a strict PASS/FAIL + fix-lists; optional repair loop re-runs steps 5–6 using must-fix items.

## Why XML-style tags in prompts?

We delimit prompt sections with lightweight XML-style tags (e.g., <task>…</task>, <constraints>…</constraints>) so the model cleanly separates instructions, context, and schema. This improves parsing reliability for multi-part prompts and complex constraints. Guidance from Anthropic and OpenAI recommends tags/prefixes to structure prompts when multiple components are present.

Example (refiner’s meta-prompt skeleton):

<role>Refiner</role>
<objective>Fuse user intent, top candidate, and judge feedback into a single executable plan.</objective>

<context> <user_intent>{{USER_PROMPT}}</user_intent> <best_candidate>{{CANDIDATE_MD}}</best_candidate> <judge_feedback>{{JUDGE_JSON}}</judge_feedback> </context> <constraints> <style>Concise, testable, production-minded</style> <non_goals>Do not add external deps beyond 'requests'</non_goals> </constraints> <deliverable> <format>markdown</format> <sections>Overview, Plan, Steps, Tests, Risks</sections> </deliverable>

<output_schema format="json">{"type":"object","properties":{"plan":{"type":"string"},"tests":{"type":"array","items":{"type":"string"}}},"required":["plan","tests"]}</output_schema>

## I/O Format Choice: XML vs JSON (and what we use)

Prompts (inputs to the model): XML-style delimiter tags to segment parts of the instruction/context. This improves comprehension and reduces prompt-parsing errors without requiring full XML tooling.
Model outputs (machine-readable): JSON with a JSON Schema and strict Structured Outputs when available (OpenAI / Azure AOAI), or “tool” schemas on Anthropic—this yields parse-safe, contract-enforced responses ideal for automation.

Why not full XML everywhere?

Verbosity: JSON is generally less verbose than XML for the same structure, which matters because LLM cost and context usage scale with token count.

Ecosystem & enforcement: Modern LLM stacks provide first-class JSON guarantees (e.g., OpenAI Structured Outputs). XML lacks equivalent, widely supported schema enforcement in current LLM APIs.

“Is JSON expensive?” Token cost ≈ bytes/structure. JSON is still usually smaller than XML, so if the alternative is XML, JSON typically saves tokens. Use tokenizer benchmarks to check.

Ultra-lean alternatives: For extreme token budgets, some engineers use TSV/CSV or minimal key:value lines; this can reduce tokens, but you lose schema validation and robustness—benchmark before switching.

Practical guidance:

Prefer JSON + JSON Schema for any output that your code will parse.

Use XML-style tags (or clear prefixes) inside prompts to mark sections: <task>, <constraints>, <context>, <examples>, <output_schema>. This is for model reading, not for your parser.

Benchmark tokens on your real prompts. Tiny syntax changes can move costs substantially.

## Schemas enforced

Judge output (stored in judge.json):
{
"type": "object",
"properties": {
"ranked": {
"type": "array",
"items": {
"type": "object",
"properties": {
"id": {"type": "string"},
"score": {"type": "number"},
"reasons": {"type": "string"}
},
"required": ["id", "score", "reasons"]
}
},
"must_fix": {"type": "array", "items": {"type": "string"}},
"nice_to_have": {"type": "array", "items": {"type": "string"}}
},
"required": ["ranked", "must_fix"]
}

Verifier output (stored in verify.json):
{
"type": "object",
"properties": {
"verdict": {"enum": ["PASS", "FAIL"]},
"reasons": {"type": "string"},
"must_fix": {"type": "array", "items": {"type": "string"}},
"nice_to_have": {"type": "array", "items": {"type": "string"}}
},
"required": ["verdict"]
}

## Token-Efficiency Tips

Keep tag names and keys short: <task>, not <primary_user_instruction_block>.

Avoid redundant nesting or repeated keys in outputs.

Put long evidence/context under one tag and reference by ID when possible.

Measure before/after with tokenizer; don’t guess.

Ops
--op run # full pipeline
--op selftest # offline stub checks
--op demo # canned example

Constraints / Non-Goals (current)

Single-file script; no DB, no web UI, no streaming

JSON parsing hardened (sanitizer + schema)

Production features (auth, sandboxing, telemetry) are out of scope

Extension Hooks (future)

--op gui (Tk) with toggles + live logs

Retry/backoff & per-stage timeouts

Pluggable candidate style list

Swap verifier for unit-test sandbox

## Instructions for ChatGPT to Help Daley Coding

Role: Teacher, not SWE.
Style: Fast, concise, high-signal.

Do:

Give architecture guidance (layout, APIs, design patterns).

Provide annotated Jupyter-style snippets only when essential.

Supply JSON schemas, prompt templates, test checklists.

Link official docs (OpenAI, Ollama).

Don’t:

Dump full code files.

Add GUIs, DBs, or deps beyond requests.

Hide JSON in prose.

Deliverables per reply:

Sectioned notes (short bullets).

Tiny code cells only when unavoidable.

Clear action checklist.

## Sources / Further Reading

Structured Outputs (JSON Schema guarantees) — OpenAI docs.

Azure AOAI: Structured outputs (JSON Schema) — recommended for multi-step workflows.

Prompt structuring with XML-style tags — Anthropic & Google Vertex guidance; OpenAI prompt engineering.

Token counting — OpenAI tokenizer & tiktoken cookbook.

JSON vs XML verbosity — JWT vs SAML (Auth0) and academic comparison.

TSV vs JSON anecdotal token cost — treat as workload-specific, benchmark.
