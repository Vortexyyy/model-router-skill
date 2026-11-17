---
name: model-router
description: Chooses which model should do a task, by difficulty. Use whenever a task is being delegated to a subagent or a model has to be chosen: very hard work goes to Fable, hard and medium work to Opus 5.5, easy and light work to Sonnet.
---

# Model router

Classify the task, pick the tier's model, then delegate with the `Agent` tool.

## Model mapping

| Tier | Intended model | `Agent` `model` value |
| --- | --- | --- |
| Very hard | Fable 5.2 (prefer) or Fable 5.1 | `fable` |
| Hard / medium | Opus 5.5 | `opus` |
| Easy / light | Sonnet 4.6 | `sonnet` |

Verification status of the identifiers (checked against this environment's model list and the `Agent` tool schema, not by a live call):

- **Opus 5.5** is valid: API id `claude-opus-5-5`, alias `opus`.
- **Fable 5.2** is valid: API id `claude-fable-5-2`, alias `fable`. **Fable 5.1** is valid as an API id (`claude-fable-5-1`) but is not a selectable alias.
- **Sonnet 4.6** exists as an API id (`claude-sonnet-4-6`) but is NOT what the alias `sonnet` resolves to here. `sonnet` resolves to the latest Sonnet, which is **Sonnet 5.5** (`claude-sonnet-5-5`). The `Agent` tool only accepts the aliases `sonnet`, `opus`, `haiku`, `fable`, so Sonnet 4.6 cannot be pinned for a subagent.
- So in practice: easy tier runs on Sonnet 5.5 and very-hard tier runs on the latest Fable (5.2). Say so in the routing note instead of claiming 4.6 or 5.1 ran.

Fable preference: use 5.2 (newest). Falling back to 5.1 is only possible where a full model id can be passed (for example direct API calls, or a host that exposes `--model claude-fable-5-1`); with the `Agent` tool, if `fable` fails, fall back to `opus` and say so.

## Fable usage guard

Fable is the scarcest tier, so stop selecting it when usage runs high.

```
FABLE_USAGE_CUTOFF = 45   # percent; edit here to change the threshold
```

Before routing anything to `fable`, read the plan limits with the `get_usage` tool (Claude Code desktop app: `mcp__ccd_session_mgmt__get_usage`, field `plan`). Check two figures, both as "percent used":

1. **Account 5-hour limit.** Rolling window as the app reports it; the reset time comes with the figure.
2. **Fable weekly limit** (the per-model weekly entry for Fable).

Rules:
- If **either** figure is **>= FABLE_USAGE_CUTOFF** (45 and above triggers; 44.9 does not), Fable is **excluded**. Route very-hard tasks to the existing fallback order instead: Opus 5.5, then Sonnet.
- Both figures **below** the cutoff: Fable is eligible again. Re-read at each routing decision; the 5-hour window rolls, so Fable resumes by itself once usage drops back under the cutoff.
- Say which figure triggered the exclusion and its percent in the routing note, for example "Fable skipped: 5-hour usage 52% >= 45%".
- Use the same basis as the limit itself: the percent the app reports, never a token count of your own.

Edge cases:
- **Usage unavailable** (`get_usage` missing in this host, status `unavailable`, `not_applicable`, or an error): treat Fable as **eligible** and note "usage unknown, Fable allowed". Do not block the top tier on a failed read. If only one of the two figures can be read, apply the rule to that one.
- **Fable excluded and Opus also unusable** (call errors or is refused): fall back to Sonnet. If no model can take the task, stop and tell the user which models failed and why. Never return an empty or silent result.
- A Fable call that errors at run time is handled the same way: fall back one step down and say so.

Check scenarios (walk through these when editing this section):

| 5-hour % | Fable weekly % | Very-hard task routes to |
| --- | --- | --- |
| 30 | 20 | Fable |
| 44.9 | 20 | Fable |
| 45 | 20 | Opus 5.5 (excluded: 5-hour) |
| 30 | 60 | Opus 5.5 (excluded: weekly) |
| 80 | 80, Opus errors | Sonnet, with a note |
| 80 -> window resets, now 10 | 20 | Fable again |
| unavailable | unavailable | Fable, with "usage unknown" note |

## Rubric

Score by observable signals; the highest tier any signal reaches wins.

**Very hard** (any one of):
- Design or change that cuts across many modules or services (about 10+ files), or needs a new architecture.
- Hard debugging with no clear repro, concurrency or race conditions, or performance work needing profiling and reasoning.
- Security-, data-loss-, or money-critical changes where a wrong answer is expensive.
- Very large context to hold at once (whole subsystems, long specs) or open-ended research with conflicting sources.
- Examples: "redesign our auth to support multi-tenant SSO across all services"; "find why the ledger totals drift by a few cents under load".

**Hard / medium** (any one, and not very hard):
- Multi-file changes (about 2-10 files), a feature with some design choices, ordinary debugging with a repro.
- Ambiguous or underspecified requirements that need judgment, tricky logic, refactors with behavior to preserve.
- Writing or reviewing tests for non-trivial code.
- Examples: "add pagination to the orders API and update the UI"; "this test fails intermittently, find out why".

**Easy / light** (all of):
- One file, or a mechanical change repeated; clear spec; low risk and easily reversed.
- Little context needed; no design decisions.
- Examples: "rename `getUser` to `fetchUser` in utils.ts"; "summarize this 50-line file"; "format this JSON and fix the typos in the README".

## Boundary rule

- Between two tiers, pick the **higher** one when the cost of a wrong answer is high (security, data loss, production, money, hard to undo, or you can't easily verify the result).
- Unsure how hard it is: treat as hard/medium. Never route down just to save cost when the task touches risky code.
- A task that splits cleanly into parts can be routed per part, but do not split work that edits the same files across parallel agents.

## Delegation

0. If the tier is very hard, apply the Fable usage guard above before choosing the model.
1. State the classification in one line: tier, the signal that decided it, and the model.
2. Call the `Agent` tool with the matching `model` value. Independent subtasks go in one message so they run in parallel. Do not set `effort` unless the user asked for a level.
3. Write a self-contained prompt: goal, file paths, constraints, and what "done" looks like. The subagent starts with no context.
4. Check the result yourself (read the changed file, run the quick check) before reporting it as done.
5. In the final answer, name which model handled which part, and flag any substitution (for example Sonnet 5.5 standing in for 4.6).

## When not to delegate

Delegating has a cost: a cold start, a prompt you must write, and a result you must check. Do the work inline when:

- it is a single obvious command or a one-line edit;
- you already have all the context loaded and a subagent would need it re-explained;
- the answer is needed immediately to decide the very next step.

Routing is for work that is big enough, or independent enough, to be worth handing off.

## Worked examples

| Task | Tier | Why |
| --- | --- | --- |
| "Rename `getUser` to `fetchUser` in `utils.ts`" | Easy -> `sonnet` | One file, mechanical, easy to verify. |
| "Add pagination to the orders API and update the UI" | Hard / medium -> `opus` | Several files, a few design choices. |
| "Find why ledger totals drift by cents under load" | Very hard -> `fable` | Needs profiling and careful reasoning, wrong answer is costly. |
| "Change how refresh tokens are stored" | Very hard -> `fable` | Security-critical, hard to undo. Round **up**. |
| "Summarize this 50-line file" | Easy -> `sonnet` | Small, low risk. |
| Two edits to the same file | One agent | Never split work that edits the same files across parallel agents. |

## Writing the delegation prompt

A subagent starts with no context. Every prompt should contain:

1. **Goal**: one sentence, in terms of the result.
2. **Where**: exact file paths and the relevant function or section names.
3. **Constraints**: what must not change, style rules, version limits.
4. **Done means**: the check that proves it (a command that passes, a file that exists).
5. **What to report back**: files changed, anything it could not verify.

## If something fails

- A model call errors or is rate-limited: fall back one tier down (Fable -> Opus -> Sonnet), say so in the final note, and never return an empty or silent result.
- A subagent reports success but gives no evidence: check the result yourself (read the changed file, run the quick check) before telling the user it is done.
- If no model can take the task, stop and tell the user which models failed and why.
