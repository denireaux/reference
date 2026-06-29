```
utilities/
└── anomaly_management/
    │
    ├── README.md                        # Tool overview, quickstart, scenario matrix
    ├── CHANGELOG.md                     # Version history tied to sprint/demo milestones
    │
    ├── bin/                             # Operator-facing entrypoints (CLI scripts)
    │   ├── inject_anomaly.sh            # Primary injection CLI — wraps all scenario types
    │   ├── clear_anomaly.sh             # Clear/resolve a named anomaly
    │   ├── list_anomalies.sh            # Show active anomaly state (Redis + Kafka lag)
    │   └── demo_run.sh                  # Orchestrated demo sequence (Tier 3 runbook automation)
    │
    ├── scenarios/                       # One file per MT-CM-09 scenario
    │   ├── scenario_cm09a_hv_status.sh          # SP02 HV status red/yellow
    │   ├── scenario_cm09b_task_request.sh       # p1.task.req injection
    │   ├── scenario_cm09c_cop_approval.sh       # COP approval path
    │   ├── scenario_cm09d_procedure_exec.sh     # Procedure execution → missionplanning
    │   ├── scenario_cm09e_notification.sh       # p2.notification trigger
    │   ├── scenario_cm09f_contract_topics.sh    # Contract topic stimulus (all 6)
    │   ├── scenario_cm09g_manual_accept.sh      # Manual accept path
    │   └── scenario_cm09h_unknown_routing.sh    # Unknown anomaly / cache miss
    │
    ├── lib/                             # Shared internals — not called directly
    │   ├── am_config.sh                 # All vars: topics, namespaces, labels, bootstrap
    │   ├── am_kafka.sh                  # Kafka produce/consume wrappers
    │   ├── am_redis.sh                  # Redis get/set/del for anomaly state tracking
    │   ├── am_s3.sh                     # S3 sync helpers (cge-int bucket)
    │   ├── am_hv.sh                     # HV inventory helpers (wraps fleet_util.sh)
    │   ├── am_icd.sh                    # ICD 2.0 message builders (SBI-652 H&S shape)
    │   ├── am_rest.sh                   # REST call wrappers (COP adapter, AM actuator)
    │   └── am_logging.sh               # Structured logging, anomaly event log writer
    │
    ├── payloads/                        # Static and templated message payloads
    │   ├── templates/
    │   │   ├── limit_violation.json.tpl         # p1.limitviolation ICD 2.0 template
    │   │   ├── task_request.json.tpl            # p1.task.req template
    │   │   ├── violation_update.json.tpl        # p1.violation.update template
    │   │   └── hv_health_status.json.tpl        # SBI-652 unified H&S template
    │   └── examples/
    │       ├── sp02_hv_status_red.json          # SP02 batterySoc red/low example
    │       ├── sp08_flux_capacitor.json         # SP08 flux capacitor anomaly example
    │       └── unknown_anomaly.json             # Unknown routing / cache miss example
    │
    ├── state/                           # Runtime state (gitignored, populated at runtime)
    │   ├── .gitkeep
    │   └── README.md                    # Explains runtime-only contents
    │
    ├── logs/                            # Local anomaly event logs (gitignored)
    │   ├── .gitkeep
    │   └── README.md                    # Explains S3 sync behavior
    │
    └── tests/                           # Self-tests for the tooling itself
        ├── test_am_kafka.sh             # Unit test — Kafka produce/consume wrappers
        ├── test_am_icd.sh              # Unit test — ICD 2.0 message shape validation
        ├── test_am_redis.sh            # Unit test — Redis state round-trip
        └── test_inject_dry_run.sh      # End-to-end dry run — all scenarios, no live traffic
```

---

## Key Design Decisions

### `bin/` vs `scenarios/` vs `lib/`
| Layer | Purpose | Who calls it |
|---|---|---|
| `bin/` | Operator CLI — argument parsing, help text, entrypoint | Humans during demo |
| `scenarios/` | One scenario per CM-09 step — pure stimulus logic | `bin/` scripts and `demo_run.sh` |
| `lib/` | Shared plumbing — Kafka, Redis, S3, ICD builders | Everything else |

This keeps scenarios thin and readable — a demo operator can open `scenario_cm09a_hv_status.sh` and understand exactly what it does without wading through infrastructure code.

---

### `am_icd.sh` — Critical for SBI-652 Alignment
This is where ICD 2.0 message construction lives. Every scenario calls into here rather than hand-rolling JSON. When the ICD changes, you fix it in one place.

---

### `payloads/templates/` — Parameterized, Not Hardcoded
Templates use substitution tokens so the same template covers any HV ID:
```
"anomalyKey": "{{HV_ID}}.{{SENSOR_PATH}}.{{SEVERITY}}.{{THRESHOLD}}"
```
Rendered at injection time by `am_icd.sh`.

---

### `state/` and `logs/` are Gitignored
Runtime anomaly state and local logs are never committed. `am_s3.sh` handles syncing logs to `cge-int` after each run — same pattern as the CM-09 report output.

---

### Ties Back to Existing Utilities
- `am_hv.sh` wraps `utilities/hv_inventory/fleet_util.sh` — no duplication
- `am_kafka.sh` mirrors patterns from `utilities/kafka/` — consistent bootstrap/exec approach using `${KAFKA_CONSUMER}` and `${KAFKA_POD}`
