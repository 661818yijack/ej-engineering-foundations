# M01-A02 — Formal Causal-Graph Why-Because Analysis (Most Complete)

Status: supplied reference pseudocode, preserved unchanged; not independently validated or approved for execution. The title is the supplied label, not a verified claim of completeness.

```text
INPUT: target_incident
SET nodes = [CREATE_NODE(target_incident, type = "TARGET_EVENT")]
SET edges = []
SET queue = [nodes[0]]

WHILE queue IS NOT EMPTY:
    E = POP_FRONT(queue)
    candidates = IDENTIFY_CONTRIBUTING_FACTORS(E)

    FOR EACH C IN candidates:
        evidence = GATHER_EVIDENCE(C)
        IF evidence IS EMPTY:
            TAG_AS_ASSUMPTION(C)
            evidence = INVESTIGATE_EVIDENCE(C)
        IF NOT IS_NECESSARY(C, E):
            CONTINUE
        C_node = ADD_OR_GET_NODE(nodes, C, evidence)
        ADD_EDGE(edges, from = C_node, to = E, relation = "CAUSES")
        IF NOT IS_ACTIONABLE_SYSTEM_CONTROL(C_node):
            APPEND C_node TO queue

    active_causes = GET_INCOMING_NODES(edges, E)
    WHILE NOT IS_SUFFICIENT(active_causes, E):
        missing_factor = FIND_MISSING_SUFFICIENCY_FACTOR(active_causes, E)
        missing_node = ADD_OR_GET_NODE(nodes, missing_factor)
        ADD_EDGE(edges, from = missing_node, to = E, relation = "CAUSES")
        APPEND missing_node TO queue
        active_causes = GET_INCOMING_NODES(edges, E)

actions = MAP_PREVENTIVE_ACTIONS_TO_NODES(nodes, edges)
RETURN nodes, edges, actions
```
