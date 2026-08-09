<!-- Diagnostic playbook for actionable MockFlow MCP failure recovery. -->
# Diagnostics

## Reference failures

- `mcp.input.value_invalid`: inspect the exact path and operation contract; do not add guessed fields.
- reference unavailable or `mcp-local-*`: reread the draft and use the returned bound reference plus that response's `reference_bindings` map. Use `reference_format: "expanded"` only to inspect the underlying opaque reference.
- `mcp.input.reference_binding_unknown`: the operation uses a `ref:<name>` absent from the submitted map. Reread, discard the old map, and echo the new response map unchanged.
- `reference_bindings_truncated: true` is not an error: the map is capped at 512 reusable entries, `omitted_reference_binding_count` reports the excess, and omitted structural references remain expanded. Use an omitted `mfref2.*` value directly or request a narrower graph context.
- A failure at `/reference_bindings/<name>` means the map value is malformed, stale, recursive, or unavailable to the current grant. Never merge maps, copy one from another connection, or retry the same map.
- output port not found: inspect component type/version and current ports; correct the alias or use the current opaque reference.

## Contract target failures

- `contract.target_ambiguous`: choose one returned compatible component/port/handler/edge candidate.
- `contract.target_unavailable`: add the missing executable graph behavior or select a compatible target.
- `contract.message_producer_emit_missing`: add an `emit` action for the selected edge to the intended executable handler.
- data-resource conformance codes: use the returned operation index, resource, component, port, public path, and remediation. Do not retry unchanged.

## Candidate and kind failures

- Schema/resource missing: upsert earlier in the same batch or use a current persisted reference.
- Cache: provide key template and value schema; optionally choose invalidation, and remove unsupported upsert semantics.
- Entity: provide at least one identifier schema.
- Transaction semantics implicit: change the operation to `not_required` or explicitly model the resource's unsupported transaction semantic as directed.

## Version and commit failures

- Stale token: reread and rebase once.
- `commit_status: committed`: verify by rereading; never repeat the mutation batch.
- `commit_status: not_committed`: correct the cause and retry with the latest token.
- Unknown outcome: reread before any write.

## Final report

State what was fixed, validation results, executed journeys, remaining non-blocking warnings, and any scope deliberately deferred.
