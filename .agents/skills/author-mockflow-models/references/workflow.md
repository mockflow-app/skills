<!-- Operating sequence for safe end-to-end MockFlow MCP authoring. -->
# End-to-end workflow

## Inspect

1. `list_diagrams` and select the intended diagram. Supply its advertised `query`, `cursor`, and optional `limit` fields; `limit` is 1–50 and defaults to 10.
2. `get_graph_draft` for the current document, response-scoped `reference_bindings`, and version token.
3. `get_graph_context` for bounded neighboring objects, valid bound references, and its own response-scoped map.
4. Read the matching authoring guide or operation contract before composing writes.

## Author

Build in executable vertical slices. A slice may require more than one batch when later operations need persisted references returned by an earlier batch:

1. Graph component and handler-owned interactions.
2. Contract schemas and logical resources.
3. HTTP, message, or data operations tied to graph targets.
4. Scenario inputs and fixtures that exercise the route.
5. Optional deployment overlay after logical behavior is stable.

V2 graph, graph-context, and contract reads default to `reference_format: "bound"`. Use each returned `ref:<name>` only while echoing that same response's complete `reference_bindings` map unchanged into the related graph, contract, or deployment operation batch. The returned map has at most 512 entries and is always valid as one batch input. When `reference_bindings_truncated` is true, the reported `omitted_reference_binding_count` references remain expanded as usable `mfref2.*` values; request a narrower graph context when more bound names are useful. The server stores no map or session state. A reread replaces the map: discard the old one rather than merging maps across responses. Use `reference_format: "expanded"` only for debugging or an explicitly lossless whole-document workflow.

Use same-batch `local:<name>` aliases for newly created contract objects. Bound references resolve before these local aliases. After commit, discard every local alias and reread for a fresh bound map and CAS token.

Treat `committed` as the public write outcome. A successful persistent mutation returns `committed: true`; a dry run returns `committed: false`. A mutation that leaves a CAS-managed draft also returns its canonical version token. A deletion does not invent a token for a draft that no longer exists.

## Verify

- Reread the mutated surface, discard the submitted map, and compare the intended object counts and references using the new response map.
- Run `validate_graph` and require `valid: true` for structural graph completion.
- Run `get_contract_gap_report`; blocking must be zero before approval.
- Run all critical scenario drafts and inspect their terminal status and route evidence.
- For deployment work, run `validate_deployment_view` and report fidelity.
- Apply layout only after semantic validation, then reread and run `validate_graph` again. Layout is a committed graph mutation and its new token is authoritative.

## Recover from concurrency or uncertain commit state

Graph, contract, scenario, and deployment surfaces have independent version tokens. Never substitute one surface's token for another.

1. Read the error's `commit_status` and `latest_version_token`.
2. If committed, do not repeat the batch; reread and verify the intended objects.
3. If not committed, reread, rebase onto the latest draft, and retry with the new token.
4. If status is unknown, reread before doing anything else.

Never substitute an old token, an old bound map, an opaque reference from another grant, or an identifier copied from another diagram.
