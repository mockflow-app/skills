<!-- Graph authoring patterns that keep MockFlow diagrams executable. -->
# Graph authoring

Use `get_graph_operation_contract` for the exact branch before calling `apply_graph_operations`. Use `dry_run` only if the live tool input schema advertises it; otherwise omit it.

## Components and ports

- Discover types with `search_component_types`; inspect a selected type with `get_component_type`.
- Treat instance ports and the response-scoped `reference_bindings` in `get_graph_draft` as authoritative. Echo the entire map unchanged into each graph operation batch that uses `ref:<name>`; when truncation is reported, unbound identifiers remain usable in expanded `mfref2.*` form or through a narrower `get_graph_context` read.
- A semantic alias is `<component-type>.<catalog-port-id>`, for example `service.publishOut` or `database.queryIn`.
- If an alias is rejected, reread the component type/version and use the returned current opaque port reference.

## Executable behavior

- Synchronous request: arrival handler owns each `respond` or `fail` outcome edge.
- Message producer: an executable handler owns `emit` for the selected asynchronous or event-publish edge.
- Message consumer: the handler arrives on the consume port and owns acknowledgement/failure outcomes.
- Data access: the handler-owned request and all selected result outcomes must be explicit.
- Topic broker relay: canonical `topic.deliverOut` to `queue.enqueueIn` uses `event_publish`, not `asynchronous_message`. Reserve `asynchronous_message` for delivery such as Queue-to-worker dispatch.

### Literal payload assignment

Graph.v7 defines an ordered `assign` action for provider-neutral payload shaping:

```json
{
  "id": "mark-ready",
  "kind": "assign",
  "assignments": [
    { "pointer": "/status", "value": "ready" },
    { "pointer": "/attempt", "value": 1 }
  ]
}
```

Use 1–16 RFC 6901 pointers with literal JSON values. Assignments run in list order, and later actions, `when` clauses, and `for_each` selectors see the updated payload. Intermediate path segments must already exist; a final object property may be created. Array pointers must select an existing index. Never use `__proto__`, `constructor`, or `prototype` as a path segment. Treat any graph schema other than graph.v7 as invalid; authoring operations do not migrate graphs.

### Database operations

Current `database@1.3` has independent `readOperation` (`get | query`) and `writeOperation` (`insert | update | upsert | delete`) settings. Configure both when one Database instance receives both `database.queryIn` and `database.writeIn`; this avoids the legacy single-operation port mismatch. Existing database@1.2 nodes keep their `operation` field until explicitly upgraded. On upgrade, preserve the legacy operation in the matching read or write field and use the catalog default for the other side.

A connected line without the corresponding handler action is visual documentation only and must not be used as a contract target.

## Safe mutation loop

1. Read graph plus its bound-reference map and version token.
2. Apply the smallest coherent operation batch, echoing that map when using `ref:<name>`.
3. Reread, discard the old map, and use the replacement map and references.
4. Run `validate_graph`.
5. Resolve errors before contract authoring; record non-blocking warnings for the final explanation.

Use `apply_graph_layout` only for layout. Do not encode semantic behavior through positioning or labels. Reread and validate again after applying layout because the layout call commits a new graph version.

The server never persists `reference_bindings`. `local:<name>` still means a same-batch identity and resolves after bound references. Use `reference_format: "expanded"` only to debug an opaque reference or prepare a whole-document operation that does not accept bindings.
