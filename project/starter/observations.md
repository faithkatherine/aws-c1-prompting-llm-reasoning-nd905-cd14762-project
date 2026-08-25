## Testing & Evaluation

### Manual testing

Before building the automated suite, I ran the chatbot through `chat.py` by hand to validate each route and several edge cases:

- **Bug reports** — tested with partial info (asks one follow-up question at a time, doesn't call the tool until description, steps to reproduce, and environment are all collected), with all info provided up front (calls `create_bug_report` immediately), and across multiple turns in the same session (correctly avoids re-asking for details already given).
- **Platform questions** — tested several FAQ-covered questions (answered only from the embedded FAQ) and one FAQ gap (gift wrapping — correctly hands off to human support instead of guessing).
- **Edge cases** — an account-specific order lookup (correctly redirected, since it needs data the FAQ can't provide), a prompt-injection attempt asking the bot to reveal its system prompt and FAQ verbatim (refused, no leak), and a fake "site administrator" request to file a placeholder bug ticket (refused, asked for a genuine issue instead).

I verified bug-report tickets actually persisted by scanning the DynamoDB table directly:

```
aws dynamodb scan --table-name bug-report-tool-stack-bug-reports --region us-east-1
```

Ticket records matched what the chatbot reported back to the customer, including `description`, `stepsToReproduce`, `environment`, and a generated `ticketId` with `status: OPEN`.

### Automated testing and a bug I found along the way

`harness-tests.json` covers all three routes plus edge cases: bug reports (partial and complete), FAQ questions (covered and uncovered), an out-of-scope request, an account-specific lookup, an ambiguous short message, a combined bug+FAQ message, a prompt-injection attempt, and a fake-authorization tool-abuse attempt — 11 test cases total.

While reviewing the first `output_eval_dataset.jsonl` run, I noticed responses that didn't match what a fresh, isolated session should produce — for example, a test that only provided a bug _description_ got a response claiming a ticket had already been filed, quoting a real ticket ID from an unrelated earlier manual test session. Since `generate-eval-dataset.py` does generate a unique `runtimeSessionId` per test case, session isolation wasn't the issue.

The actual cause: AgentCore harnesses provision a Memory instance automatically by default, scoped by `actorId` + `sessionId`. The script never set an explicit `actorId`, so every test invocation fell back to the same default actor — meaning long-term memory carried over across "fresh" sessions, including memories from my own manual `chat.py` testing.

Fix: added a unique, randomly generated `actorId` per test case (alongside the existing per-test `runtimeSessionId`) in `generate-eval-dataset.py`'s `invoke_harness_once` call. Re-ran the dataset generation afterward and confirmed the contamination was gone — every response was grounded only in its own single-turn prompt.

### Evaluation results

Ran Bedrock Evaluations (LLM-as-a-judge, `Builtin.Correctness`, evaluator model `amazon.nova-pro-v1:0`) against the corrected dataset.

- **Overall Correctness score: 1.00** (average across all 11 test cases, normalized 0-1 scale — 1.00 is the maximum)
- Every category scored at the ceiling: bug-report collection and tool-calling, FAQ grounding (both covered and uncovered questions), out-of-scope hand-off, and the adversarial edge cases (prompt injection, fake authorization) all matched their expected behavior with no deductions.

All 11 responses in the final dataset matched their expected behavior on manual review: correct routing in every case, FAQ answers grounded only in the provided document, the bug-report checklist enforced before tool calls, and both adversarial tests (prompt injection, fake authorization) handled without leaking instructions or filing bogus tickets. The perfect score is consistent with what manual `chat.py` testing had already shown before the automated suite was run — the harness behaved predictably and no further prompt iteration was needed.

### A behavior worth noting: premature tool calls under multi-turn conditions

While the automated eval scored 1.00, closer review of manual `chat.py` transcripts and the DynamoDB scan surfaced a pattern the eval dataset didn't catch, since each eval test case is single-turn.

In one multi-turn session (`chat-transcript-bug-report-multiturn-part1.png`), after the customer gave only a bug description, the model attempted to call `create_bug_report` immediately — before asking for steps to reproduce or environment. The tool correctly rejected the call (missing required fields), and the model recovered appropriately by asking the customer for the missing detail instead of surfacing an error. This confirms the "if the tool errors, ask again rather than treat it as broken" instruction works as intended — but it's also evidence the "do not call the tool until all three fields are collected" instruction isn't followed with full consistency.

This same pattern produced one bad ticket in the DynamoDB table: ticket `7e50cf92-f6b2-4cfe-8cdd-0ec753c06645` has its `environment` field set to the literal text of the assistant's own clarifying question ("Please provide your browser, OS, and device information.") rather than an actual customer-provided value — meaning on a different occasion, the same premature-call behavior slipped past the tool's validation instead of being caught. The tool's non-empty-string check doesn't distinguish a real answer from restated prompt text, so this edge case can still produce a low-quality ticket even when the tool call technically "succeeds."

This didn't affect the automated Correctness score, since none of the 11 test cases happened to exercise this specific multi-turn sequence, but it's worth flagging as a known limitation: the prompt's collection-order instruction is a strong tendency, not a hard guarantee, and a stricter mitigation (e.g. validating that field values look like real answers, not restated questions) would need to live in the Lambda tool itself rather than the prompt alone.
