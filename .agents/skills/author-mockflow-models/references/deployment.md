<!-- Deployment overlay workflow for MockFlow physical-view authoring. -->
# Deployment views

Deployment views are physical overlays on the validated logical model.

1. Read the logical graph or contract context that identifies the deployment targets and retain that response's bound-reference map with the deployment version token.
2. Apply focused changes with `apply_deployment_view_operations`, echoing `reference_bindings` whenever the batch uses `ref:<name>`.
3. Reread after commit; never reuse a stale deployment token.
4. Run `validate_deployment_view` and inspect fidelity diagnostics.

Use `replace_deployment_view_draft` only when intentionally replacing the complete overlay. Preserve logical component identities and express cloud/provider placement as implementation detail rather than changing the logical contract.

Deployment reference bindings are stateless and batch-only. The server remembers nothing; discard the map after a graph or contract reread. Whole-document replacement does not accept a map, so use current expanded references when that workflow is necessary.

Explain deviations between logical guarantees and physical topology, including regional placement, managed-service semantics, replication, and intentionally unresolved fidelity warnings.
