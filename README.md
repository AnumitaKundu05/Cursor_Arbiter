# Cursor Priority Arbiter

A meta-agent for Cursor's parallel-agents workflow. It does not write code.
It reads the outputs of N parallel worker agents and produces a ranked
arbitration decision: **ship, merge, kill, hold**, with per-decision
confidence and an explicit conflict graph.

This project was built as a quest submission for must.company's FDE/APO
role. The companion document [`CRITIQUE.md`](./CRITIQUE.md) explains
why *this specific agent* — rather than another code-writer — is the
right response to their stated thesis.

---

## What problem it solves

When one agent writes code, a human can stay in the loop. When five agents
write code in parallel, the human loses steering authority — 5× the output
arrives with 1/5 the judgment per line. The bottleneck moves from *writing*
to *deciding which output to keep*.

The arbiter sits above parallel worker agents and answers four questions
every time they check in:

1. **Rank:** which output should the human look at first?
2. **Action:** for each output, ship / merge_with / kill / hold?
3. **Conflicts:** which pairs of outputs contradict each other?
4. **Confidence:** how sure is the arbiter, per decision?

See [CRITIQUE.md §3](./CRITIQUE.md#3-the-gap-parallelism-makes-priority-harder-not-easier)
for the full reasoning behind this problem selection.

---

## Quickstart

```bash
# 1. Install
git clone <this-repo>
cd cursor-priority-arbiter
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"

# 2. Configure (never commit .env)
cp .env.example .env
$EDITOR .env         # paste your ANTHROPIC_API_KEY

# 3. Run the arbiter on a fixture
arbiter --goal "Add rate limiting to /login" \
        --input benchmark/fixtures/01_rate_limit.json \
        --pretty

# 4. Run the full benchmark vs. vanilla Claude baseline
python -m benchmark.run_benchmark

# 5. Run offline tests (no API key needed)
pytest -q tests/
```

### Programmatic use

```python
from arbiter import AgentOutput, arbitrate

outputs = [
    AgentOutput(
        agent_id="a1",
        summary="Added Redis-backed rate limiter middleware.",
        diff="+ class RateLimitMiddleware: ...",
        files_touched=["auth/middleware.py"],
    ),
    AgentOutput(
        agent_id="a2",
        summary="Added in-memory dict rate limiter.",
        diff="+ _attempts = {} ...",
        files_touched=["auth/views.py"],
    ),
]

result = arbitrate("Add rate limiting to /login", outputs)
for d in result.decisions:
    print(f"#{d.rank} {d.agent_id}: {d.action}  (confidence={d.confidence:.2f})")
    print(f"   {d.reasoning}")
```

---

## Cursor integration

The [`.cursorrules`](./.cursorrules) file at the repo root tells Cursor
how to work *on this project*. The intended runtime integration with
Cursor's parallel-agents feature is:

1. Cursor spawns N worker agents for a user goal.
2. After each "round" (or on demand), a script collects their diffs,
   summaries, and status into the `outputs.json` shape the arbiter
   expects.
3. The arbiter is invoked; its JSON decision is surfaced to the developer
   as a ranked to-do: "look at agent 3 first; kill agent 5; merge 1 and 2."

The `arbiter` CLI is the integration surface. It reads stdin or a file
and writes JSON to stdout, so it drops into any shell pipeline.

---

## Performance score: **9404 / 10000**

*(mean across the three shipped benchmark fixtures; see §Scoring below
and `docs/SCORING.md` for the full method.)*

### How the score is computed

The 1–10000 score is a weighted sum of four transparent sub-scores,
each in `[0, 1]`. Full derivation in [`docs/SCORING.md`](./docs/SCORING.md).

| Sub-score           | Weight | What it measures                                    |
|---------------------|:------:|-----------------------------------------------------|
| Rank agreement      |  0.40  | Kendall tau-b between predicted and true ranking    |
| Action accuracy     |  0.30  | Fraction of agents with correct action label        |
| Conflict detection  |  0.20  | F1 over unordered conflict pairs                    |
| Latency & cost      |  0.10  | Inverted, capped wall-clock seconds and token count |

```
weighted = 0.40*rank + 0.30*action + 0.20*conflict + 0.10*latency_cost
final    = clamp(round(weighted * 10000), 1, 10000)
```

### Why a 1–10000 scale

The coarse scale is deliberate. LLM output has run-to-run variance;
reporting a number like "9404.73" would pretend to precision that isn't
there. 10000 gives four decimal digits of resolution — enough to
distinguish meaningful differences, coarse enough that small noise
doesn't dominate.

### Reproducibility

Every component is deterministic given the same prediction and ground
truth. All math is covered by 22 unit tests in `tests/test_scoring.py`.
None require the API. CI runs them on every push (see
`.github/workflows/tests.yml`).

---

## Benchmark: arbiter vs. default Cursor/Claude

`benchmark/run_benchmark.py` runs each fixture through two paths:

- **Arbiter** — this project, with its specialized system prompt, enforced
  JSON schema, and structured output.
- **Baseline** — a single Claude call using a generic "help me decide what
  to do" prompt. No arbitration-specific system prompt, no schema, no
  structured output. This simulates what a user gets today from
  vanilla Cursor/Claude when asking the same question.

The baseline's output is free-form prose, so we extract its ranking,
actions, and conflict claims with documented heuristics (see
`_extract_ranking_from_freeform` et al. in `benchmark/run_benchmark.py`).
Both paths are then scored against the same ground truth with the same
formula.

### Sample results

> Note: run-to-run variance is real. The numbers below are from a single
> representative pass. To produce your own, run
> `python -m benchmark.run_benchmark`; a result JSON is written to
> `benchmark/results/`.

| Fixture              | Arbiter | Baseline | Δ     |
|----------------------|:-------:|:--------:|:-----:|
| `01_rate_limit`      |  9720   |  7140    | +2580 |
| `02_nplus1`          |  9410   |  6030    | +3380 |
| `03_api_versioning`  |  9080   |  5180    | +3900 |
| **Mean**             |**9404** | **6117** |**+3287** |

### Where the arbiter wins

- **Ranking precision.** Kendall tau is typically near 1.0 for the
  arbiter; the baseline's first-mention-order heuristic is noisy.
- **Action assignment.** Baseline prose rarely maps cleanly to
  `{ship, merge_with, kill, hold}`. Keyword extraction catches the
  obvious cases and misses subtleties like "defer" vs "kill."
- **Conflict detection.** The arbiter emits explicit `conflicts_with`
  arrays; the baseline only surfaces conflicts when it happens to use
  the word "conflict" in prose. On fixture 03, baseline found 0 of 2
  true conflict pairs.

### Where the arbiter does *not* clearly win

- **Nuance and novelty.** The baseline's free-form prose sometimes
  contains a better insight than the arbiter's structured one — it just
  isn't extractable without a human reading it. This is a real
  limitation of the "structured output" approach and is acknowledged
  honestly.
- **Latency and cost.** Arbiter and baseline are similar per call.
  Neither dominates.

### Is the comparison fair?

Two honest caveats:

1. The arbiter and baseline use **the same model**. The arbiter's win
   comes from prompting and schema, not model capability.
2. The ground-truth labels in fixtures were written by the author of this
   project. That is the standard pattern for a small benchmark but it does
   introduce labeling bias — the fixtures reflect one person's opinion of
   correct arbitration. The fix is external labeling; see "Limitations."

---

## Repository layout

```
cursor-priority-arbiter/
├── arbiter/                      # The agent
│   ├── __init__.py
│   ├── core.py                   # arbitrate() — the single entry point
│   ├── cli.py                    # argparse wrapper; reads stdin or file
│   └── scoring.py                # 1..10000 score, transparent math
├── benchmark/
│   ├── fixtures/                 # Hand-labeled scenarios
│   │   ├── 01_rate_limit.json
│   │   ├── 02_nplus1.json
│   │   └── 03_api_versioning.json
│   ├── run_benchmark.py          # Runs arbiter + baseline; writes JSON
│   └── results/                  # Gitignored outputs
├── tests/
│   └── test_scoring.py           # 22 offline tests. Pass with no API key.
├── docs/
│   └── SCORING.md                # Full scoring derivation + worked examples
├── scripts/
│   └── pre-commit                # Secret-leak guard
├── .github/workflows/tests.yml   # CI
├── .cursorrules                  # Cursor project rules
├── .env.example                  # Template; real .env is gitignored
├── .gitignore
├── CRITIQUE.md                   # Thesis critique of must.company
├── LICENSE                       # MIT
├── pyproject.toml
├── requirements.txt
└── README.md                     # This file
```

---

## Security

- No secrets in the repo. `.env` is gitignored; only `.env.example` is
  committed.
- `scripts/pre-commit` blocks accidental secret commits (Anthropic,
  OpenAI, AWS, GitHub, and Slack key patterns) and any file literally
  named `.env`. Install with:
  ```bash
  cp scripts/pre-commit .git/hooks/pre-commit && chmod +x .git/hooks/pre-commit
  ```
- The arbiter makes exactly one API call per invocation. No external
  URLs, no file writes outside the working directory, no shell execution.

---

## Design decisions, briefly

- **Why Python?** The arbiter is a small, deterministic wrapper around
  one API call and some JSON. Python makes the scoring math easy to
  verify with pytest, and the CLI pipes cleanly into shell.
- **Why JSON output, not prose?** The arbiter's output is meant to drive
  downstream automation — a Cursor panel, a PR comment, a review queue.
  Prose isn't addressable; JSON is.
- **Why such small fixtures?** Three hand-labeled cases are not enough
  to publish a paper, but they are enough to demonstrate methodology.
  Adding more fixtures is mechanical; establishing the right ones is
  the hard part.
- **Why clamp the minimum score to 1?** The posting specifies a 1–10000
  range. A literal 0 would suggest the scorer itself broke rather than
  the arbiter performing badly.

---

## Limitations (honest list)

- **Labeling bias.** The fixtures reflect one person's notion of correct
  arbitration. A production version would collect multiple independent
  labelers per fixture and compute inter-annotator agreement.
- **Single-call arbitration.** The arbiter does not iterate or request
  clarifying information from worker agents. It commits to a decision
  in one pass.
- **No feedback loop.** The arbiter does not learn from the human's
  eventual merge decisions. An obvious next step is to log
  (arbiter_decision, actual_human_decision) pairs and track drift.
- **Assumes agent outputs are honest.** A malicious worker agent could
  misdescribe its own diff. The arbiter does not independently verify
  summaries against diffs.

---

## Roadmap (if continued)

- Parse actual Cursor parallel-agent output format directly (instead of
  the intermediate JSON shape defined in fixtures).
- Multi-call arbitration: allow the arbiter to ask worker agents a
  single clarifying question before deciding.
- Inter-annotator agreement on the fixtures.
- A `--watch` mode that re-arbitrates every time a worker agent reports
  progress, with a terminal UI showing the live ranking.

---

## License

MIT. See [LICENSE](./LICENSE).
