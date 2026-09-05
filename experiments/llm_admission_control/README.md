# LLM admission-control validation

## Research question

Does the proposed `UnifiedLLM` admission layer keep mixed Chat Completions and
Responses traffic within a simulated gateway's concurrency capacity, and does
that improve completion of an OVDR-shaped multi-stage workload?

## Experiment design

`validate.py` starts a loopback OpenAI-compatible HTTP server with a fixed
concurrency capacity. It then compares:

1. A burst with no admission limit.
2. The same burst with `max_in_flight` equal to server capacity.
3. Fifteen concurrent three-stage workflows—two Chat aliases followed by one
   Responses alias—sharing a single admission group.
4. Two spawned child processes, each configured with `max_in_flight=2`, against
   one shared loopback gateway.

The script uses real local sockets and LiteLLM's current Chat and Responses
client paths. It does not call an external model or require credentials.

## Key metrics

- Successful calls and completed workflows
- Gateway overload responses
- Peak concurrency observed by the gateway
- Wall-clock duration

## How to run

From the repository root:

```bash
uv run python experiments/llm_admission_control/validate.py
```

## Results summary

Five fresh-process runs of the complete experiment on 2026-09-02 produced the
same functional result:

| Mode | Successful | Gateway overloads | Gateway peak | Elapsed range |
|---|---:|---:|---:|---:|
| Unbounded 20-call burst | 3/20 | 17 | 3 | 0.426-0.585 s |
| Protected 20-call burst, limit 3 | 20/20 | 0 | 3 | 0.286-0.312 s |
| Protected OVDR shadow, limit 5 | 15/15 workflows | 0 | 5 | 0.274-0.314 s |
| Two processes, limit 2 each | 12/12 | 0 | 4 | 2.747-3.097 s |

The OVDR shadow completed 45 provider calls: each of 15 concurrent workflows
ran two Chat stages and one Responses stage.

The child-process peak of four demonstrates that process-local limits do not
combine into a hard aggregate ceiling.

These results validate local admission behavior only; they are not measurements
from OVDR or an NVIDIA inference gateway.
