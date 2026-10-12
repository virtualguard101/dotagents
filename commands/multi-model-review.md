The user wants an adversarial multi-model code review: several independent reviewers on different models, then a synthesized verdict. You MUST run those reviewers **in parallel**: issue every Task subagent launch in the same assistant turn (multiple Task tool calls together), not one after another — do not wait for one reviewer to finish before starting the next. Do not edit files or apply fixes unless the user explicitly asks.

## 1) Pick models

If a structured multiple-choice question tool is present in this session's tool list, call it before launching reviewers. Use the exact tool name from that list. Build a single question from the exact Task/Subagent model slugs listed in your system prompt/tool instructions. Do not hardcode, invent, or repeat a repo-local model list here. Use those slugs verbatim for `options[].id` so they can be passed directly to the Task tool, set `allow_multiple: true`, and use short human-readable labels. Present every available model neutrally and in the source order: do not recommend, rank, preselect, omit, or add "(Recommended)" to any option.

If that tool is not available, ask the user to choose models inline in your message (not by inventing a tool call) and wait for their response before launching reviewers. Offer the exact Task/Subagent model slugs from your system prompt/tool instructions, with the same neutrality rules.

If the user already chose models in the `/multi-model-review` line or chat, you may skip asking only when their selection is explicit and unambiguous; otherwise ask. If a selected slug is rejected at execution time, prefer the closest valid slug suggested by the Task/Subagent tool feedback.

## 2) Scope and intent

1. Determine what to review: user-specified paths or diff, else `git diff main...HEAD` (or the repo default base vs HEAD), else local unstaged + staged diffs, else files and symbols from the conversation.
2. State one clear paragraph of **intent** — what the change is trying to accomplish (from messages, commits, or the code).
3. Collect the material to pass to reviewers: full diff or key file excerpts.

## 3) Launch one subagent per selected model (parallel only)

For **each** model slug the user selected, launch the Task tool with `subagent_type`: `generalPurpose`, `readonly`: true, and `model` set to that slug. **Parallelism is mandatory:** fire all of these Task calls in one batch in a single response (concurrent tool use), never sequentially. Each subagent gets the same reviewer prompt; diversity comes from the models, not from different personas.

Each subagent prompt must include:
- The stated intent paragraph.
- The code/diff under review.
- Instructions to act as an adversarial reviewer: prioritize bugs, correctness, security, and maintainability; structured findings with severity (critical | warning | nit), concrete locations, evidence, optional suggestion; say "no findings" if nothing is wrong.

## 4) Synthesize (you, the parent agent)

Merge the subagent findings: **consensus** (2+ models), **lone-model** findings, deduplicate overlapping issues, note disagreements. Categorize for the user: act on / consider / noted / dismissed, with brief rationale. Do not auto-apply changes.
