---
name: author-mockflow-models
description: Inspect, author, validate, simulate, map to implementation, and explain MockFlow distributed-system models through the MockFlow MCP. Use when creating or changing diagrams, components, ports, handlers, interactions, contract schemas and operations, data resources, message channels, journeys, scenarios, deployment views, implementation mappings, or when recovering from MockFlow MCP validation and version-token failures.
---

# Author MockFlow Models

Use the MCP as the source of live state and this skill as the operating procedure. Build executable system behavior, not a visual-only diagram.

## Workflow

1. Resolve the target with `list_diagrams`; use optional `query`, `cursor`, and `limit` inputs. `limit` is 1–50 and defaults to 10. Never infer a diagram reference from a browser URL.
2. Read `get_graph_draft`, `get_graph_context`, and the relevant discovery contract before planning mutations.
3. Discover component definitions with `search_component_types`, then inspect the selected definition with `get_component_type`. Select current ports and handlers from the returned catalog and graph.
4. Plan a coherent slice: graph behavior, contracts, resources, and at least one scenario that proves the route.
5. Dry-run fragile graph or contract batches when supported. Apply one coherent batch with the latest version token.
6. Reread after committed contract, deployment, or scenario batches. A dependent graph batch may chain directly from a successful `apply_graph_operations` response only when it returns `committed: true`, `validation.valid: true`, the canonical version token, and an untruncated `reference_bindings` map. Reread at every surface boundary, after an unexpected result or layout, and before final verification.
7. Run `validate_graph`, `get_contract_gap_report`, and `run_scenario_draft`. Do not call the model complete while blocking diagnostics or an unexecuted critical journey remain.
8. Explain the finished model in domain terms: entry point, handler-owned route, state interaction, failure outcomes, and scenario evidence.

For implementation mapping, use the manifest-linked implementation-context resource before scanning the local repository. Follow [implementation-mapping.md](references/implementation-mapping.md); never infer target references from a browser URL or send repository bodies to MockFlow.

## Non-negotiable Rules

- Use `local:<name>` only inside the batch that creates that binding, schema, resource, or example. Never persist or reuse it across calls.
- Bound `ref:<name>` values are valid only with the exact `reference_bindings` map returned by the current read or eligible committed graph mutation response. The server remembers nothing; echo the entire returned map unchanged into every related operation batch that uses it. The map never exceeds 512 entries.
- If `reference_bindings_truncated` is true, `omitted_reference_binding_count` reports eligible references left expanded as `mfref2.*`; use those values directly or request a narrower graph context.
- Discard bound maps after every reread and use the replacement response map. Request `reference_format: "expanded"` only as a debugging escape hatch or for a lossless whole-document workflow that does not accept bindings.
- Do not reuse `mcp-local-*` values. For an eligible dependent graph batch, use the committed response's bound references with its exact map; otherwise reread the draft and use its returned references.
- Never blindly retry a mutation after a projection or transport failure. Inspect commit status and latest version token first.
- Require `committed: true` after a successful persistent mutation and `committed: false` after a dry run. For a CAS-managed draft, use the returned canonical version token; do not expect deletions to fabricate one.
- For contract writes, inspect `gap_report_delta.after.blocking` and `introduced_items`; a committed incremental save may still be blocked.
- A graph edge is not executable by itself. The arrival handler must own response/failure actions, and a message producer handler must own an `emit` action for its selected edge.
- Use graph.v7 `assign` actions for bounded literal payload changes. Keep assignment order intentional because later actions and conditions see the updated payload.
- Treat graph.v7 and `mockflow.mcp.scenario-run.v5` as the only active graph and run schemas. Follow the live tool schemas instead of requesting retired aliases or previous envelopes.
- Pre-v7 graph documents have no compatibility or migration path in the authoring tools. If a read returns one, stop and report the unsupported artifact; never convert, patch, or write it as part of normal authoring.
- Prefer semantic aliases such as `service.publishOut` only when the current component catalog advertises that port. Use current opaque references otherwise.
- Use `get_graph_operation_contract` or `get_contract_operation_contract` for exact operation shapes. Do not guess conditional fields.
- Preserve compare-and-swap discipline: reread, rebase the intended change, then retry once with the latest token.
- Report unresolved warnings and accepted modeling limits explicitly.

## Route to References

- Read [workflow.md](references/workflow.md) for sequencing, batching, version recovery, and completion gates.
- Read [graph-authoring.md](references/graph-authoring.md) when changing components, handlers, ports, edges, or layout.
- Read [contracts.md](references/contracts.md) for HTTP, message, schema, data-resource, output-binding, and diagnostic examples.
- Read [journeys.md](references/journeys.md) for scenario creation, deterministic execution, and coverage evidence.
- Read [deployment.md](references/deployment.md) for deployment overlays and fidelity validation.
- Read [implementation-mapping.md](references/implementation-mapping.md) when matching an existing repository to MockFlow targets or applying active implementation mappings.
- Read [diagnostics.md](references/diagnostics.md) when any MCP call fails or returns gaps.

Load only the references needed for the current slice, but always load `workflow.md` before a multi-surface authoring task.
