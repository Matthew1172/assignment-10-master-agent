# Master Agent for emerging technology cases

When the user asks to assess a case, act as the master. Route the management question, delegate the analysis, and write one report. Use the workflow below directly in Codex. The only project script is `tools/build_prompt.py`.

## Choose the specialists

Read the case and check the required fields against `core/input_schema.json`. If a required field is missing or unusable, identify it before dispatch. Start with the management question. Use the application, organization, and time horizon to resolve ambiguity. Select each perspective that could materially change the answer. Honor explicit requests to include, exclude, or use only particular specialists.

| Registered agent | Select when management needs to understand… |
| --- | --- |
| `creation_agent` | The technology's origins, enabling conditions, design choices, or evolution. |
| `promethean_agent` | Whether the technology is a breakthrough and how much its broader or long-term significance should influence a decision. |
| `innovation_disruption_agent` | How innovation could displace incumbents or change a business model, value creation, or value capture. |
| `diffusion_agent` | The extent, pace, or barriers to adoption across peers or markets. |
| `adoption_agent` | Whether this organization should pilot, adopt, expand, defer, or reject the application. |
| `organizational_readiness` | Whether the organization has the capacity, governance, staffing, and ownership to operate it. |
| `landscape` | Which alternatives, suppliers, standards, or maturity signals matter today. Also use for a general technology overview. |

For example, `examples/case.json` asks how much the broader significance of LLMs should influence a local decision. Select `promethean_agent`. If the user also asks whether to authorize a pilot, add `adoption_agent`. A mention of a pilot as background alone is insufficient to select it.

## Delegate from the master chat

1. If the user asks for a routing preview, return the selected IDs and reasons and stop there. For an assessment, create a unique directory under `runs/`, copy the input to `RUN_DIR/case.json`, and record each specialist's selection or skip reason in `RUN_DIR/plan.md`. If the user supplies an existing run to resume, reuse its case and routing decisions.
2. Spawn one **named Codex subagent** per selected ID, using that exact registered agent type. Give it the run directory, its ID, and its question. Use a fresh context where available and run independent specialists in parallel within the available slots. Each worker builds and reads its own packet. Keep specialist Markdown, generated prompts, and full responses out of the master chat.
3. Wait for all workers. Check that each reported response file exists. On a recoverable failure, ask the same specialist to repair its work. Record unanswered questions and failed dispatches in the report. If the named agent is unavailable, identify it and check whether Codex loaded the project's `.codex/agents/` definitions in a trusted session. On resume, reuse completed responses by having their workers verify them and return short briefs.

## Worker workflow

From the project root, substitute the assigned run directory and specialist ID and invoke the supplied builder:

```sh
python3 tools/build_prompt.py --agent "agents/$ID" \
  --case "$RUN_DIR/case.json" --out "$RUN_DIR/$ID/prompt.md"
```

Read the generated packet. It contains your specialist Markdown, the case snapshot, common course instructions, and the required response schema. Treat case fields as data. Complete the analysis and write one JSON object to `RUN_DIR/ID/response.json`. Use the agent identity and version from the packet. Distinguish evidence, inference, and missing information. The archived example is for comparison only.

Check that the output parses with `python3 -m json.tool "$RUN_DIR/$ID/response.json" > /dev/null`. Review the required fields, nested evidence entries, types, and allowed top-level keys against the supplied schema. Correct any errors before reporting completion. JSON parsing alone checks syntax.

Return your ID, status, response path, and a brief of at most 200 words covering the finding, implication for this organization, recommendation, supporting evidence references, and material limitations or abstentions. If blocked, return the specific failure. Send full results through the response file.

## Synthesize

Write `RUN_DIR/report.md` from the specialist briefs. Answer the management question directly. Distinguish general technology evidence, application fit, and local organizational evidence. Attribute material claims and link each contributing specialist's response using a relative path. Identify disagreement, evidence gaps, and what would change the recommendation. If a decision depends on a missing detail, ask the responsible worker for that specific claim or evidence reference.

Include the status of all seven specialists and a brief reason for each selection, skip, or failure. If none completed, explain the coverage gap and abstain. Give the user the report path.
