# Report figures

TODO(team): Add evaluation figures with source data, generation commands, and units documented.

Store original diagrams and vector PDF plots here. Keep preview images and temporary exports under `../dist/verification/`. Every measured figure must identify its experiment batch and source table in `../PLAN.md` or its caption.

## Architecture figure and section evidence

`architecture.tex` is a new editable TikZ diagram, included directly by the
design section and compiled with the report. It is a conceptual architecture
figure, not an experimental measurement or a slide screenshot. Solid arrows
show segment-byte direction; dashed arrows show metadata/control requests.
The single Peer box groups independent sources and targets. In particular,
repair is source Peer -> Controller -> target Peer, not a direct Peer copy.

The introduction and design draft were checked against these repository sources
(all paths below are relative to the repository root):

| Topic / formula | Implementation or archived evidence |
| --- | --- |
| Raft state, read barriers, publication (Eq. 1) | `internal/metadata/{raft,fsm,handlers}.go` |
| Durable installation, integrity and quotas | `internal/storage/{store,handlers}.go` |
| Repair grace, serialized execution, target reconciliation | `internal/control/coordinator.go`, `internal/repair/policy.go` |
| Resource-aware placement | `internal/placement/{adaptive,plan}.go` |
| Shared token budget and L rule (Eqs. 2-3) | `internal/storage/{handlers,traffic}.go` |
| Headless download, buffer and source selection | `internal/playback/`, `docs/playback.md` |
| Gateway and browser MSE path | `internal/playback/media.go`, `cmd/demo-api/`, `frontend/`, `scripts/media/` |
| Scoped contribution/findings and unused age branch | `report/compact-experiment-design.md`, `report/compact-study-results.md`, `PLAN.md` |

Eq. 2 describes token capacity, not an exact per-interval network-throughput
measurement. Eq. 3 uses the oldest task represented by an active repair
request at that source, not all pending tasks in the cluster. The draft does
not attribute the current reconciliation features to every historical study
binary; experimental provenance belongs in Methodology/Evaluation.

Build from `docs/final-report/` with `latexmk final-report.tex` or
`tectonic --outdir dist --keep-logs final-report.tex`.

## Additive review evidence and startup table

The implementation-validation addition uses
`internal/metadata/{fsm,maintenance}_test.go`,
`internal/storage/{reliability,traffic}_test.go`, and `tests/system/repair.py`.
Successful archived checks are in `results/compact-study/validation/`:
`gates.json`, `02-go-test.log`, `05-repair.log`, and `09-media.log`.
Earlier Chromium acceptance is recorded in `docs/verification.md`, separately
from the headless performance study.

The bandwidth explanation derives from
`results/compact-study/study-config.json`: 128 KiB / 2 s = 64 KiB/s per
viewer; aggregate capacity is 96 + 144 + 192 = 432 KiB/s. It is a nominal
consumption comparison, not a fitted throughput model. The timing caveat
follows `experiments/repair_playback_study.py`, which records first verification
and its preceding observation, keeps unrecovered runs censored, and
distinguishes unknown from zero-copy states.

Regenerate `startup-table.tex` from this report directory:

```sh
python3 assets/generate-startup-table.py
```

The script selects archived E1 N/OFF/S sessions at low/high demand, grace 3,
persistent outage, and no second failure from
`results/compact-study/sessions.csv`. It checks unique sessions, scheduled
counts, and run-level completed-session means against
`results/compact-study/summary.csv`. Displayed means pool completed sessions;
ranges are individual-session minima and maxima, not confidence intervals.
No new experiment was run.
