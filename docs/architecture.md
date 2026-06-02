# Architecture

## Pattern
Deterministic orchestrator + role-scoped AI agents + one external tool.

There is **no orchestrator node**. The n8n canvas *is* the orchestrator: the
control flow is encoded in the graph, not decided at runtime by an LLM. This is
the correct choice because every run follows the same fixed sequence; the only
branches are on cheap deterministic signals (calendar toggle, Gemini-vs-Groq
fallback). An LLM router/orchestrator only earns its place when the execution
path is dynamic and unknown in advance.

This sits at the "structured multi-agent pipeline with agentic practices" point on
the spectrum (single prompt -> chained pipeline -> router agent -> autonomous
agent). It demonstrates agentic *practices* — role-scoped agents, tool use,
structured JSON I/O, deterministic verification, per-step fallback — without
autonomous decision-making.

## AI vs deterministic split
- **AI (Groq llama-3.3-70b, Gemini fallback)**: summarise web research, diagnose
  readiness honestly, draft the strategy + day plan, coach behavioural answers.
- **Deterministic code**: score the self-assessment dropdowns, pick prep
  intensity, repair the plan to the exact day count and hour cap, assemble the
  HTML email, build calendar events.

The single clearest example of the split is **Validate and Repair Roadmap**: the
planner is *asked* for exactly N days under an hour cap, but LLMs do not reliably
honour hard numeric constraints, so a code node re-checks and repairs the output.
AI for judgement, code for guarantees.

## Why HTTP Request nodes for the LLM calls
- Full control over request/response shape and explicit tool-vs-AI boundary.
- Reliable on the free tier and trivial to give each call a Gemini fallback via the
  node's error output.
- API keys are read from environment with `{{ $env.KEY }}` in headers
  (`N8N_BLOCK_ENV_ACCESS_IN_NODE=false`), so no secrets live in the workflow JSON.

## Why four agents instead of one prompt
Single responsibility: shorter prompts, individually retryable, lighter rate-limit
pressure, and tone isolation (the blunt reality-check voice cannot bleed into the
plan or the behavioural coaching).

## Fallback design
Each primary node (Groq) uses `onError: continueErrorOutput`. Its success output
goes to the Parse node; its error output goes to a Gemini node, which then feeds
the same Parse node. The Parse node reads either response shape
(`choices[0].message.content` for Groq, `candidates[0].content.parts[0].text` for
Gemini), JSON-parses with a regex fallback, and substitutes safe defaults if both
fail — so the email always sends.

(Groq is primary and Gemini is fallback only because Gemini requests were
returning "invalid syntax" in this environment; the wiring is symmetric, so they
can be swapped back in `build_workflow.py` once that is fixed.)

## Node map (21 nodes)
```
Intake Form -> Validate and Normalize -> Tavily Web Search
  -> Agent 1 Interview Intelligence (+Fallback Gemini) -> Parse Intelligence
  -> Agent 2 Reality Check          (+Fallback Gemini) -> Parse Reality
  -> Agent 3 Roadmap Planner        (+Fallback Gemini) -> Validate and Repair Roadmap
  -> Agent 4 Behavioural Prep       (+Fallback Gemini) -> Parse Behavioural
  -> Assemble Report -> Send Email -> Calendar Enabled?
        Yes -> Build Calendar Events -> Google Calendar - Create Events -> Done
        No  -> Done
```