<!-- Canonical contract authoring patterns for MockFlow MCP operations. -->
# Contract authoring

Always call `get_contract_operation_contract` for the selected operation before `apply_contract_operations`. Discovery returns the strict schema, conditional requirements, reference rules, and validated examples.

## References and batching

- New same-batch objects: `local:<name>`.
- Persisted objects: graph/context/contract reads default to `ref:<name>` plus a top-level `reference_bindings` map. Echo the entire returned map unchanged into every `apply_contract_operations` batch that uses those names. A truncated response leaves omitted references expanded as usable `mfref2.*` values.
- Never reuse `mcp-local-*` values or a local alias in a later call.
- The server remembers no binding map. After committing schemas/resources, call `get_contract_draft`, discard the prior map, and use the new response's map and schema/resource references in later operations.
- Bound references resolve before same-batch `local:<name>` aliases. Use `reference_format: "expanded"` only for debugging or a whole-document workflow whose mutation does not accept bindings.
- Bound references also resolve inside structural arrays. For example, `target.result_edge_refs: ["ref:cache-hit", "ref:cache-miss"]` is valid when both names appear in the batch's top-level `reference_bindings` map.

## HTTP

`upsert_http_contract` accepts either an inline JSON example, an existing `schema_key`, or both. A schema key links the current draft schema; it does not create a new schema body. For 2–25 operations, prefer `upsert_http_contracts`: it plans definitions sequentially against one in-memory draft, accepts top-level `reference_bindings`, and performs one atomic CAS commit.

Canonical sequence:

1. Upsert or identify request/response schemas.
2. Add the HTTP operation against an executable request handler.
3. Upsert parameters and every success/error response.
4. Attach examples from approved artifacts when required.

High-level `upsert_http_contract.responses[]` may select an exact graph outcome with `edge_ref`:

```json
{
  "status_code": "200",
  "edge_ref": "<exact response edge_ref>",
  "description": "Accepted response route",
  "schema_key": "<optional existing response schema key>"
}
```

Omit `edge_ref` only when one compatible response edge exists for that status and section. When two graph outcomes intentionally share a status, provide one response entry per edge; `(edge_ref, status_code)` is the identity. Use the returned `candidate_object_refs` when selection is ambiguous. A catalog-advertised semantic target alias such as `gateway.requestIn` is valid for the high-level target; use the returned opaque port reference only if the instance is not on that catalog version.

## Message

A complete message operation should declare:

```json
{
  "op": "add_message_operation",
  "binding_ref": "local:playback-publish",
  "operation_key": "playback.events.publish",
  "direction": "producer",
  "resource_key": "channel.playback.events",
  "channel_address": "playback.events",
  "message_schema_ref": "<current schema_ref>",
  "delivery": {
    "mode": "at_least_once",
    "ordering": "none",
    "acknowledgement": "automatic",
    "retry_owner_ref": null
  },
  "target": {
    "component_ref": "<service component_ref>",
    "port_ref": "service.publishOut",
    "message_edge_ref": "<event_publish edge_ref>"
  }
}
```

The `binding_ref: local:playback-publish` value names the new binding created by this add operation; no prior binding-creation call is needed.

`correlation_field` and `correlation_schema_ref` are one declaration. Supply both or omit both; never write only the field name.

Use `delivery.acknowledgement: explicit` only when the selected executable handler owns ACK or NACK outcome edges. When those outcomes are absent, omit the field while drafting or use `none` or `automatic` for a complete declaration; explicit acknowledgement is not a generic reliability flag.

A topic consumer uses the same logical resource and selects its executable consume boundary:

```json
{
  "op": "add_message_operation",
  "binding_ref": "local:playback-consume",
  "operation_key": "playback.events.consume",
  "direction": "consumer",
  "resource_key": "channel.playback.events",
  "channel_address": "playback.events",
  "message_schema_ref": "<current schema_ref>",
  "delivery": {
    "mode": "at_least_once",
    "ordering": "none",
    "acknowledgement": "none",
    "retry_owner_ref": null
  },
  "target": {
    "component_ref": "<consumer component_ref>",
    "port_ref": "<current consume port_ref>",
    "message_edge_ref": "<topic-to-consumer edge_ref>"
  }
}
```

A Queue consumer may declare explicit acknowledgement when its arrival handler owns the modeled ACK/NACK outcomes. It may also declare its Queue-owned DLQ without inventing a worker failure route. When the selected delivery edge originates at a Queue, omitting `retry_owner_ref` selects that Queue. Omitting `dead_letter_edge_ref` selects its single compatible `queue.dlqOut` route; use the explicit reference returned in `candidate_object_refs` only when more than one route is compatible:

```json
{
  "op": "add_message_operation",
  "binding_ref": "local:fulfilment-consume",
  "operation_key": "fulfilment.consume",
  "direction": "consumer",
  "resource_key": "channel.fulfilment",
  "channel_address": "fulfilment.jobs",
  "message_schema_ref": "<current schema_ref>",
  "delivery": {
    "mode": "at_least_once",
    "ordering": "none",
    "acknowledgement": "explicit",
    "dead_letter_resource_key": "channel.fulfilment.dlq"
  },
  "target": {
    "component_ref": "<worker component_ref>",
    "port_ref": "<worker consume port_ref>",
    "message_edge_ref": "<queue dispatch edge_ref>",
    "dead_letter_edge_ref": "<optional queue dlqOut edge_ref>"
  }
}
```

For a message producer, the selected handler must emit the selected edge. An unedged `fail` in that handler may reject the enclosing synchronous request independently; it is not a producer error outcome and does not require an error edge in the message binding. Message-consumer failures still require explicit selected error edges. Queue routes use enqueue/asynchronous-message semantics; topic routes use publish/event-publish semantics. `resource_key` and `channel_address` declare the logical channel; the selected graph edge associates it with the queue/topic node, so there is no separate channel-creation operation. MockFlow derives the logical message registry deterministically from complete contract declarations and graph broker nodes.

## Artifact-backed examples

First-class contract examples point into immutable evidence; inline `json_example` values do not replace them. Call `create_checkpoint` after the scenario is stable, then pass the returned snapshot reference, matching content-hash token, and a valid JSON Pointer to `upsert_contract_example`.

- `journey_snapshot` requires `provenance: "authored"`.
- `approved_run` requires `provenance: "approved"`.
- Use the exact integrity token for the selected artifact. A route or manifest hash is not interchangeable with the snapshot content-hash token.

## Data resources

- Table/collection: current `key_schema_ref`; optional record schema; declared operations.
- Cache: `key_template` and `value_schema_ref`; optional `invalidation` and `ttl_ms`; no `upsert` operation.
- Entity: at least one `identifier_schema_refs` entry; optional `lifecycle_states` and `mapped_data_resource_refs`.
- `mapped_data_resource_refs` may identify only other authored data resources. Graph linkage comes from data-operation targets, never from this field.
- `implementation_mapping` is strict: repository, package, path, symbol, ref, schemaFileRef, and migrationRef only, with at least one location field populated.

For a data operation, choose a resource operation it declares. Use `transactional: not_required` unless the resource explicitly models required transaction semantics according to the returned operation contract.

When practical, upsert a data resource and its first data operation in one atomic batch with `local:` references. If a resource is committed first, `contract_gap.data_resource_unused` is an expected non-blocking warning until a data operation targets it; reread the gap report after adding usage and still require final blocking to be zero.

```json
{
  "op": "add_data_operation",
  "binding_ref": "local:progress-write",
  "operation_key": "playback.progress.write",
  "resource_ref": "<current progress resource_ref>",
  "operation": "write",
  "consistency": "read_your_writes",
  "transactional": "not_required",
  "target": {
    "component_ref": "<data-store component_ref>",
    "port_ref": "<current write port_ref>",
    "request_edge_ref": "<consumer-to-store request edge_ref>",
    "result_edge_refs": ["<successful write result edge_ref>"]
  }
}
```

## Output bindings

Use a current component and output port. Semantic aliases resolve only when that exact port exists on the component's current catalog version.

## Completion

Budget bindings from the executable boundaries reported by the coverage gap report, not from public endpoint count. Synchronous request and message coverage is normally per edge, including actor-to-gateway, gateway-to-service, service-to-external, scheduler, and message hops that reach authored handlers. Canonical Topic-to-Queue relays are excluded, and data-access edges sharing one interaction are covered once for that interaction.

Call `get_contract_draft` after commit and `get_contract_gap_report`. A structurally valid contract batch can return `committed: true` while leaving blocking gaps. Inspect `gap_report_delta.after.blocking` and `introduced_items`, then reread the full report; `resolved_item_ids` alone is not completion evidence. Do not report contract completion while data usages are unsynchronized, schema references are missing, or blocking gaps remain.

Use `dry_run` only when the live `apply_contract_operations` input schema advertises it. Never infer support from another mutation tool.
