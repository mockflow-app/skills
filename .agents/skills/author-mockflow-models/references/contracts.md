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

## JSON Schema patterns

Author every `pattern` and `patternProperties` key in MockFlow's bounded export-safe subset. The MCP rejects an unsafe pattern before saving the contract draft.

- Anchor the full value with `^` and `$` and keep the pattern at most 256 characters.
- Do not use groups `()`, alternation `|`, counted repetitions such as `{4,}`, or backreferences such as `\1`.
- Use at most one unescaped `*`, `+`, or `?` outside a character class, and close every character class and escape.
- Replace counted repetitions with explicit tokens. For an order ID containing at least four digits, use `^ord-[0-9][0-9][0-9][0-9][0-9]*$`, not `^ord-[0-9]{4,}$`.

Treat `json_schema.pattern_unsafe` as an authoring failure. Correct the reported schema path and resubmit the original atomic batch with its still-current token; do not wait for export to rediscover the problem.

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

For a producer, the selected handler must emit the selected edge. Queue routes use enqueue/asynchronous-message semantics; topic routes use publish/event-publish semantics. `resource_key` and `channel_address` declare the logical channel; the selected graph edge associates it with the queue/topic node, so there is no separate channel-creation operation. MockFlow derives the logical message registry deterministically from complete contract declarations and graph broker nodes.

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

Component-output bindings are independent of HTTP, message, and data-resource contracts. Creating those contracts never creates an output binding implicitly, and the gap report may remain clean when `componentOutputs.bindings` is empty.

Keep the UI and contract terms distinct. A component header such as “4 outputs” counts available graph/catalog output ports. A data operation's `result_edge_refs` selects executable result routes. Only entries in `componentOutputs.bindings` are Component Output schema bindings; an output row displaying “No schema” is unbound.

When component outputs or complete contract coverage are in scope:

1. Read the current graph, component catalog, and contract draft. Inventory every in-scope port whose direction is `output`. If “component outputs” does not make clear whether the user wants every surfaced output or only connected payload-bearing outputs, show both discovered counts and ask which scope to bind. Do not ask when the user already requested complete or route-scoped coverage.
2. Call `get_contract_operation_contract` for both `upsert_contract_schema` and `upsert_component_output_binding`.
3. Create or reuse an appropriate schema for each distinct output shape. Prefer current schemas, approved examples, and executed journey evidence over inference from labels. If those sources allow incompatible shapes that would change consumers, ask which shape is authoritative. Never invent a discriminator absent from the runtime payload. In one batch, a new schema may use `local:<schema-name>` and its output binding may refer to that same local schema.
4. Apply `upsert_component_output_binding` with the current component reference, current output-port reference or advertised semantic alias, and schema reference. Semantic aliases resolve only when that exact port exists on the component's current catalog version.
5. Reread `get_contract_draft` and verify that `document.componentOutputs.bindings` contains one binding for every selected output port. Return a component, port, outcome, schema-key, and binding summary plus the selected and omitted counts. An empty binding collection is incomplete unless the requested scope intentionally contains no component outputs.

An output binding has no journey selector or `when` condition. Model distinct outcomes with distinct component output ports and bind each port separately; for example, bind Cache `hitOut` and `missOut` independently. If several outcomes share one physical output port, use one schema that represents the permitted variants, such as an optional field or JSON Schema union, and keep the outcome condition in graph handler or edge logic. Journeys supply payloads, variables, and fixtures that select and prove those graph branches; they do not override the port's output binding.

## Completion

Budget bindings from the executable boundaries reported by the coverage gap report, not from public endpoint count. Synchronous request and message coverage is normally per edge, including actor-to-gateway, gateway-to-service, service-to-external, scheduler, and message hops that reach authored handlers. Canonical Topic-to-Queue relays are excluded, and data-access edges sharing one interaction are covered once for that interaction.

Call `get_contract_draft` after commit and `get_contract_gap_report`. A structurally valid contract batch can return `committed: true` while leaving blocking gaps. Inspect `gap_report_delta.after.blocking` and `introduced_items`, then reread the full report; `resolved_item_ids` alone is not completion evidence. A zero-gap report is also not evidence that optional component-output bindings were authored: inspect `document.componentOutputs.bindings` directly whenever they are in scope. Do not report contract completion while data usages are unsynchronized, schema references are missing, blocking gaps remain, or required output bindings are absent.

Use `dry_run` only when the live `apply_contract_operations` input schema advertises it. Never infer support from another mutation tool.
