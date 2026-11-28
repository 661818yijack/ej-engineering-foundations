# M03-A02 — Blueprint B: Complete Architecture (All-In Modular Reference)

Status: supplied reference blueprint, not independently validated or approved for execution. The example combines optional discovery, routing, trust, planning and execution mechanisms around the core schema in [Blueprint A](minimal-static-manifest.md).

## Complete Capability Envelope (`complete-capability.json`)

```json
{
  "core": {
    "id": "run_tests",
    "endpoint": "https://qa.example.org/rpc",
    "description": "Run unit tests in an isolated sandbox.",
    "input_schema": { "type": "object", "required": ["repo_path"] }
  },
  "identity": {
    "agent_did": "did:web:qa.example.org",
    "public_key_jwk": { "kty": "OKP", "crv": "Ed25519", "x": "<key>" }
  },
  "routing": {
    "domain": "software.qa",
    "embedding_vector": [0.012, -0.044, "..."]
  },
  "sla": {
    "p95_latency_ms": 1200,
    "max_cost_usd": 0.02
  },
  "attestation": {
    "issuer_did": "did:web:auditor.trustroot.org",
    "claims": { "sandbox_isolated": true, "pass_rate": 0.994 },
    "expires_at": "2026-12-31T00:00:00Z",
    "signature": "<ed25519_sig>"
  }
}
```

## Complete Modular Pipeline (Pseudocode)

```python
class CompleteDiscovery:
    def __init__(self, config: dict, local_manifests: list[str], deps: dict):
        self.cfg = config          # Feature flags toggled dynamically by the AI agent
        self.deps = deps           # Optional: ans_resolver, vector_db, trust_graph, signer
        self.local_index = load_manifests(local_manifests)

    def resolve_and_execute(self, task_goal: str, constraints: dict) -> dict:
        # 1. DAG COALITION PLANNER (Decompose if multi-step)
        steps = (
            self.deps["planner"].decompose_to_dag(task_goal)
            if self.cfg.get("enable_dag_coalition")
            else [{"query": task_goal, "args": constraints.get("args", {})}]
        )

        results = {}
        for step in steps:
            # 2. CANDIDATE RETRIEVAL (Static vs. Federated ANS/ANP)
            candidates = (
                self.deps["ans"].resolve(step["query"])
                if self.cfg.get("enable_federated_ans")
                else list(self.local_index.values())
            )

            # 3. HIERARCHICAL ACTIVE SEMANTIC ROUTING (Coarse domain -> Fine vector)
            if self.cfg.get("enable_semantic_routing"):
                candidates = self.deps["router"].filter_by_domain(candidates, step["query"])
                candidates = self.deps["router"].top_k_semantic(candidates, step["query"], k=5)

            # 4. ZERO-TRUST CRYPTOGRAPHIC ATTESTATION (Verify DID & signed claims)
            if self.cfg.get("enable_attestation"):
                candidates = [
                    c for c in candidates
                    if verify_did_attestation(c, self.deps["trusted_issuers"])
                ]

            # 5. TRUST-WEIGHTED & SLA SCORING (Social graph + latency/cost)
            ranked = self._rank_candidates(candidates, constraints)
            if not ranked:
                raise DiscoveryError(f"No valid provider for: {step['query']}")

            # 6. MUTUAL BINDING & AUTOMATIC FALLBACK (ACNBP)
            results[step["query"]] = self._execute_with_fallback(ranked, step["args"], constraints)

        return results

    def _rank_candidates(self, candidates: list[dict], constraints: dict) -> list[dict]:
        scored = []
        for c in candidates:
            if not meets_hard_sla(c.get("sla", {}), constraints):
                continue
            trust = (
                self.deps["trust_graph"].get_score(c["identity"]["agent_did"])
                if self.cfg.get("enable_trust_graph")
                else 1.0
            )
            utility = compute_sla_utility(c.get("sla", {}), constraints)
            scored.append((0.5 * trust + 0.5 * utility, c))
        return [c for _, c in sorted(scored, key=lambda x: x[0], reverse=True)]

    def _execute_with_fallback(self, ranked: list[dict], args: dict, constraints: dict) -> dict:
        for candidate in ranked:
            try:
                contract = None
                if self.cfg.get("enable_binding"):
                    contract = negotiate_and_sign_contract(candidate, constraints, self.deps["signer"])

                output = invoke_rpc(candidate["core"], args, contract=contract)

                if self.cfg.get("enable_trust_graph"):
                    self.deps["trust_graph"].record_success(candidate["identity"]["agent_did"])
                return output
            except Exception as err:
                if self.cfg.get("enable_trust_graph"):
                    self.deps["trust_graph"].record_failure(candidate["identity"]["agent_did"])
                audit_log("FALLBACK_TRIGGERED", candidate["core"]["id"], str(err))
                continue

        raise ExecutionError("All ranked candidates failed.")
```

## Adaptation notes

Treat module boundaries as candidates for composition rather than proof that every flag combination works. Normalize the flat minimal record and wrapped complete envelope; establish field and dependency requirements for selected modules. The example does not demonstrate DAG dependency execution, actual enforcement of end-to-end budgets, or safe fallback after an invocation with an uncertain outcome. A generic exception is not proof that a side effect did not occur; non-idempotent work must not be retried blindly. Identity and signed claims do not themselves grant authority or prove all advertised capabilities. The fixed ranking weights and top-k value are illustrative. No standards conformance, secure implementation, or guaranteed upgrade path is established by the pseudocode.
