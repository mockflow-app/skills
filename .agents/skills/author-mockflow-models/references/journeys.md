<!-- Scenario and journey practices for proving MockFlow model behavior. -->
# Journeys and scenarios

Use scenarios as executable acceptance evidence, not decorative examples.

## Workflow

1. Read `get_scenario_authoring_context` for current graph starts, fixture targets, and limits.
2. Use `list_scenario_drafts` and `get_scenario_draft` before creating duplicates.
3. Create with `create_scenario_draft`; update with `update_scenario_draft` and the latest scenario version token returned by the scenario read/create response. Never use a graph or contract token here.
4. Configure start node/port, bounded payload, fixtures, variables, seed, and stop conditions.
5. Execute `run_scenario_draft` and inspect status, `execution_coverage.invoked_nodes`, route, failures, emitted messages, and state effects. Treat `node_ref` as the reusable component identity; labels are explanatory only.

`summary_filter` is an object, not a string. For failure-focused summaries use `{ "focus": "failures", "include_context": true }`; valid focus values are `representative`, `failures`, and `issues`.

Full detail defaults to 25 events. Use `event_filter` to select kinds, severities, node/action/edge references, plus bounded `context_events`. Keep `runtime_reference_mode: "compact"` for page-local event, activation, stage, token, call-frame, correlation, payload, and state aliases; request `"resource"` only when an opaque runtime reference is specifically needed. Compact aliases are scoped to one page and must not be compared across pages. Continue with `trace.event_page.next_start_sequence` and use its matched, filtered, returned, and continuation counts.

Every evaluated branch exposes `condition_result`; `decision_coverage.entries` reports evaluation and selection counts without treating an intentionally unselected alternative as a warning.

For `mockflow.mcp.scenario-run.v5`, read `run.failure_digest.entries[].incident_count` as distinct failures grouped by code, attempt, and causal lineage. `activation_count` counts affected handler activations and `evidence_event_count` counts propagated trace evidence; do not report either as an incident total. `run.failure_digest.distinct_code_count` is the number of failure codes before the 50-entry bound.

## Coverage

For each critical boundary, include the smallest scenarios that prove:

- primary success;
- meaningful dependency failure or timeout;
- asynchronous delivery/acknowledgement where modeled;
- data read/write outcomes where modeled.

Keep fixtures deterministic and provider-neutral. A scenario that never traverses a newly authored handler or edge does not validate it.

Use the authored contract and handler semantics to determine the expected journey outcome. If the user has not defined whether an ambiguous dependency result should succeed, reject, degrade, retry, or dead-letter and current artifacts support several choices, ask before creating a scenario. Never choose an outcome only because it produces a completed run.

Database fixtures are collection-keyed. `get` preserves the input payload and adds `/record`; `query` adds `/records`. Object Store fixtures are `{ "<bucket>": { "<payload key>": { "status": "stored" } } }`, where the key comes from the component's `keyPath` (default `/key`); correlation IDs never address objects.

`execution_coverage` is present for both summary and full detail. It is derived from every `NODE_ENTERED` event in the run rather than the filtered trace or current event page. Repeated entries increment `invocation_count`. If `nodes_truncated` is true, request full detail and follow `trace.event_page.next_start_sequence` until exhausted, collecting `node_ref` from `trace.events` for every `NODE_ENTERED` event.

Use the live MCP tool input schemas plus `get_scenario_authoring_context` for exact create/update fields; there is no graph- or contract-operation schema to reuse for scenarios. If the requested scope proves only success, report failure/timeout coverage as deferred rather than implying it was executed.

Use `create_checkpoint` only after graph, scenario, and contract drafts are stable and validated. Retain the returned pinned snapshot references for examples and approval evidence.
