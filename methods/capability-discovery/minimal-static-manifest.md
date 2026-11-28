# M03-A01 — Blueprint A: Minimal Baseline (Zero-Dependency Core)

Status: supplied reference blueprint, not independently validated or approved for execution.

## Minimal Schema (`capability.json`)

```json
{
  "id": "run_tests",
  "endpoint": "http://127.0.0.1:8081/rpc",
  "description": "Run unit tests in a repo and return pass/fail counts.",
  "input_schema": { "type": "object", "required": ["repo_path"] }
}
```

## Minimal Implementation

```python
class MinimalDiscovery:
    def __init__(self, manifest_paths: list[str]):
        # Load static JSON files or /.well-known/agent-card.json endpoints
        self.index = {c["id"]: c for p in manifest_paths for c in load_json(p)}

    def find(self, capability_id: str) -> dict:
        if capability_id not in self.index:
            raise KeyError(f"Unknown capability: {capability_id}")
        return self.index[capability_id]

    def call(self, capability_id: str, args: dict) -> dict:
        cap = self.find(capability_id)
        validate_schema(args, cap["input_schema"])
        return http_post(cap["endpoint"], {"method": capability_id, "params": args})
```

## Adaptation notes

The supplied example illustrates lookup by an exact capability ID and invocation through a known endpoint. Helper functions are unspecified; the example is not a complete runnable implementation, and its dependency and protocol-compatibility claims have not been verified. Define manifest shape, duplicate-ID handling, input validation, authority, endpoint trust and failure handling where required. Discovery alone does not grant permission to call a capability.
