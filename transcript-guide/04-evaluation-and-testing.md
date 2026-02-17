# Evaluation and Testing

## Why Testing Matters

Before trusting an agent with something important (like your email), you want to know that it behaves correctly. The course uses **LangSmith** to run and log evaluations so you can:

- Check that **triage** decisions match what you expect.
- Check that the **right tools** are called (and in a sensible order).
- Check that **final responses** (e.g. drafted emails) meet your criteria.

You can evaluate at different **levels**: triage only, tool calls only, or the full end-to-end output.

---

## Levels of Evaluation

1. **Triage (routing)** – Did the router output the correct label (e.g. respond / notify / ignore) for each test email? This is a **discrete** decision, so you can compare to a **ground truth** label (e.g. string match or exact match).
2. **Tool calls** – Did the agent call the expected tools (e.g. `write_email` then `done`), and are there no important tools missing? Again, you can compare to an expected list (e.g. set of tool names or sequence).
3. **End-to-end (response quality)** – Is the final output (e.g. the drafted email) acceptable? There’s no single “correct” text, so you use **criteria** and often an **LLM-as-judge** to grade whether the response meets those criteria.

So: **triage and tool calls** → simple comparisons (strings, sets, or lists); **full response** → criteria + LLM judge.

---

## Two Ways to Run Evaluations in LangSmith

### 1. Pytest + LangSmith logging

- Write normal **pytest** tests that run your graph (or triage, or agent) on specific inputs.
- Use LangSmith’s **decorators** (or helpers) to **log** each run to LangSmith (inputs, outputs, pass/fail, latency).
- You can then inspect **experiments** in the LangSmith UI: see all test cases, which passed/failed, and drill into traces.

Good when you want **developer-friendly** tests and to add/change test cases in code (e.g. a list of example emails and expected tool calls).

### 2. Datasets + Evaluate API

- Create a **dataset** in LangSmith: each row has an **input** (e.g. email text) and **reference outputs** (e.g. expected classification, or expected tool-call list).
- You define a **target function** that runs your assistant on the dataset input and returns the **agent output** (e.g. classification or list of tool names).
- You define **evaluator** function(s) that compare **reference** vs **agent output** and return a score (e.g. true/false or 0–1).
- The **Evaluate API** runs the target on every dataset example, runs your evaluator(s) on each, and logs everything to LangSmith.

Good when you have a **shared dataset** and want to run several evaluators (triage accuracy, tool-call accuracy, etc.) without writing a separate pytest for each; great for teams building a test suite over time.

---

## Unit Test: Triage Decision

- **Input:** Email text.  
- **Reference:** Expected classification (e.g. `"respond"`).  
- **Agent output:** Whatever your triage node writes to state (e.g. `state["classification_decision"]`).  
- **Evaluator:** Simple comparison, e.g. `agent_output["classification_decision"] == reference["classification_decision"]`.

You can run this either:

- In **pytest**: one test per example (or parametrized), logging to LangSmith, or  
- Via **Evaluate API**: one dataset of (email, expected_classification), one target that runs the graph and returns the classification, one evaluator that does the string match.

---

## Unit Test: Tool Calls

- **Input:** Email text.  
- **Reference:** Expected list of tool names (e.g. `["write_email", "done"]`).  
- **Agent output:** List of tool names actually called (extracted from the final `messages` or state).  
- **Evaluator:** Check that no **expected** tool is missing (and optionally that no unexpected tool was called). For example: `set(expected) <= set(actual)` or compare ordered lists.

Again, this can be pytest (e.g. `test_tools.py` with a few email examples) with LangSmith logging, so you see pass/fail and traces in the LangSmith experiment.

---

## End-to-End: LLM-as-Judge

For the **full response** (e.g. the email the agent drafted), you usually don’t have a single “correct” string. Instead you define **success criteria** (e.g. “Acknowledge the question and confirm it will be investigated”) and use an **LLM** to judge whether the response meets them.

Typical setup:

1. **Schema** – e.g. `grade: bool`, `justification: str`. You bind this as **structured output** to a “grader” model.
2. **Prompt** – You give the grader: (a) the **criteria** (bullet points or short paragraph), (b) the **response** to grade (e.g. the concatenated messages or the drafted email).
3. **Instructions** – Tell the grader to evaluate against **each** criterion, that **all** must be met for pass, and to cite specific text. This improves consistency.
4. **Output** – You get `grade` (true/false) and `justification`. You can log both to LangSmith and use `grade` as the test result.

Important nuances from the course:

- **Criteria** – Not too specific (or many valid answers get marked fail), not too vague (or everything passes). A good middle ground: specify the **tool** you expect (e.g. “use write_email”) and the **intent** (e.g. “acknowledge and confirm investigation”).
- **Grader prompt** – Iterate on the instructions (e.g. “evaluate each bullet; all must be met; cite evidence”) so results are stable across runs.

You can run this in pytest (e.g. `test_response.py`) over a set of (email, criteria) pairs, with each run logged to LangSmith so you can audit the grader’s reasoning in the trace.

---

## Viewing Results in LangSmith

- **Experiments** – After running pytest or Evaluate, open the experiment in LangSmith. You’ll see inputs, outputs, scores, and latency. “Full” view shows full payloads; “compact” gives a summary.
- **Traces** – Click into a run to see the full trace: triage, agent nodes, tool calls, and (for LLM-as-judge) the grader call with prompt and structured output.
- **Programmatic use** – You can use the LangSmith client to **read** experiment results (e.g. by experiment name) and compute stats or export data for further analysis in a notebook or script.

---

## Summary

| What you evaluate | How | Where it’s logged |
|-------------------|-----|-------------------|
| Triage decision | Compare agent classification to reference (string/match) | Pytest or Evaluate API → LangSmith experiment |
| Tool calls | Compare expected vs actual tool names (set/list) | Pytest or Evaluate API → LangSmith experiment |
| Response quality | LLM-as-judge with criteria + structured output (grade + justification) | Pytest (e.g. test_response.py) → LangSmith experiment |

Use **pytest** when you want tests in code and quick iteration; use **datasets + Evaluate API** when you want a central dataset and multiple evaluators. For both, LangSmith gives you traces and experiments to debug and improve the agent.

Next: [Human-in-the-Loop](./05-human-in-the-loop.md) — how to add approval and feedback before sensitive actions.
