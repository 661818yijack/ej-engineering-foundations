# M01-A01 — Classic Linear 5 Whys (Most Basic)

Status: supplied reference pseudocode, preserved unchanged; not independently validated or approved for execution.

```text
INPUT: problem_statement, max_depth = 5
SET current_event = problem_statement
SET depth = 0
SET chain = []

WHILE depth < max_depth AND NOT is_controllable_process_gap(current_event):
    cause = ASK_AND_RECORD("Why did " + current_event + " happen?")
    IF is_only_human_error(cause):
        cause = ASK_AND_RECORD("What process or system gap allowed " + cause + "?")
    APPEND cause TO chain
    current_event = cause
    depth = depth + 1

countermeasure = DEFINE_SINGLE_ACTION(current_event)
VERIFY_RECURRENCE_PREVENTION(countermeasure)
RETURN chain, countermeasure
```
