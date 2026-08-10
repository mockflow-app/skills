<!-- Diagnostic playbook for actionable MockFlow MCP failure recovery. -->
# Diagnostics

## Reference failures

- `mcp.input.value_invalid`: inspect the exact path and operation contract; do not add guessed fields.
- reference unavailable or `mcp-local-*`: reread the draft and use the returned bound reference plus that response's `reference_bindings` map. Use `reference_format: "expanded"` only to inspect the underlying opaque reference.
- `mcp.input.reference_binding_unknown`: the operation uses a `ref:<name>` absent from the submitted map. Reread, discard the old map, and echo the new response map unchanged.
- `reference_bindings_truncated: true` is not an error: the map is capped at 512 reusable entries, `omitted_reference_binding_count` reports the excess, and omitted structural references remain expanded. Use an omitted `mfref2.*` value directly or request a narrower graph context.
- A failure at `/reference_bindings/<name>` means the map value is malformed, stale, recursive, or unavailable to the current grant. Never merge maps, copy one from another connection, or retry the same map.
- output port not found: inspect component type/version and current ports; correct the alias or use the current opaque reference.
- `graph.edge.incompatible_message`: use the compatible semantic port aliases in the remediation. Common data routes use `service.commandOut` for command inputs and `service.recordOut` for record inputs; do not retry unchanged with `service.requestOut`.

## Contract target failures

- `contract.target_ambiguous`: choose one returned compatible component/port/handler/edge candidate.
- `contract.target_unavailable`: add the missing executable graph behavior or select a compatible target.
- `contract.message_producer_emit_missing`: add an `emit` action for the selected edge to the intended executable handler.
- data-resource conformance codes: use the returned operation index, resource, component, port, public path, and remediation. Do not retry unchanged.

## Candidate and kind failures

- `graph.edge.invalid_kind` on `queue.dlqOut`: terminal source roles derive `error`; correct the edge kind instead of using `asynchronous_message`.
- Rejected catch on a component without a terminal output: Remove the rejected catch to propagate the failure to its caller, or model a degraded response. A Gateway may shape the terminal rejection through `errorOut`.
- Handler action and catch `when` conditions use `expression.v1`; graph edge `when` filters use `expression.v2`.
- Schema/resource missing: upsert earlier in the same batch or use a current persisted reference.
- Cache: provide key template and value schema; optionally choose invalidation, and remove unsupported upsert semantics.
- Entity: provide at least one identifier schema.
- Transaction semantics implicit: change the operation to `not_required` or explicitly model the resource's unsupported transaction semantic as directed.

## Version and commit failures

- `graph.version.unsupported`: stop authoring and report the artifact. Pre-v7 graphs have no compatibility or migration path. Do not use `replace_graph_draft` as a converter or fallback.
- An ordinary `apply_graph_operations` validation failure rejects the atomic candidate before persistence. Correct the reported fields and retry with the same token; a concurrent write will still produce a stale-token conflict.
- Stale token: reread and rebase once.
- `commit_status: committed`: verify by rereading; never repeat the mutation batch.
- `commit_status: not_committed`: correct the cause and retry with the latest token.
- Unknown outcome: reread before any write.
- `simulation_unavailable` with `reason: artifact_invalid`: validate the graph and scenario once. If validation is clean, do not retry or alter the unchanged draft; report the correlation ID because the worker artifact failed an integrity check.

## Final report

State what was fixed, validation results, executed journeys, remaining non-blocking warnings, and any scope deliberately deferred.
