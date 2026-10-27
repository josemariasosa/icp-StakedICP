# neuron-audit operator runbook

`neuron-audit` keeps its audit commands read-only and adds an explicitly gated,
controller-verified `StartDissolving` workflow. `StartDissolving` begins the
neuron's dissolve countdown; it does not disburse ICP or transfer tokens.
`RecoverDisburse` remains a separate legacy capability with its existing safeguards.

## Canonical neuron inventory (`audit-in/neurons.txt`)

`audit-in/neurons.txt` is the canonical list of every known neuron ID: one `u64`
per line, numerically sorted, deduplicated, regardless of owner, accessibility or
dissolve state. It lists *candidates* only; it is never authorization evidence.

1. **Maintain** the canonical file. Entries are never removed by tooling; a
   malformed canonical file makes `sync` fail without touching it.
2. **Synchronize** when new sources appear (offline, no NNS access, atomic write,
   idempotent). Run from the repo root or the crate directory:

```sh
cargo run --manifest-path tools/neuron-audit/Cargo.toml -- inventory sync --include-history
# extra explicit sources: --source bash-array=recovery/2026-10-22/check-neurons.sh
#   kinds: bash-array|csv|snapshot|control-report|controller-trace|disburse
# preview only: --dry-run
```

   `--include-history` reads a fixed trusted set (recovery `check-neurons.sh`
   `NEURONS=( ... )` array, `neurons-audit.txt`, `audit-out` snapshots, CSV,
   control report, controller trace, disburse records) and extracts IDs only from
   named neuron-ID fields/columns. Malformed records are reported, not imported.
   The deterministic report with per-source counts, duplicates, unique-to-source
   IDs and per-ID provenance is `audit-out/inventory-report.md`.
3. **Audit** the inventory with fresh authenticated reads (`audit --ids-file audit-in/neurons.txt ...`).
4. **Plan** from the canonical inventory (`--ids-file` is optional and defaults to it):
5. **Keep separate**: controller verification (the plan) and dissolution
   execution (`dissolve start` / `batch`) are distinct operations.

## Build and prepare a fresh plan

Run from the repository root. The planner defaults to the canonical
`tools/neuron-audit/audit-in/neurons.txt` (override with `--ids-file`). It
fetches fresh signed `list_neurons` responses and includes only full records
whose controller equals the live Oracle principal. Hotkeys and info-only records
never qualify.

```sh
cargo run --manifest-path tools/neuron-audit/Cargo.toml -- dissolve plan \
  --out-dir audit-out --owner-pem -
```

Inspect the generated `audit-out/dissolve/plan-*.json`. Review the eligible
IDs, amounts, delays, expected dates, exclusions and `plan_sha256`. Keep this
plan immutable. A later run with a new live state should make a new plan.

## Execute one neuron

Select one reviewed ID from the eligible list. The operator must explicitly
provide `--execute`. Example (replace the ID and plan path):

```sh
cargo run --manifest-path tools/neuron-audit/Cargo.toml -- dissolve start \
  --neuron-id 16136654443876485299 \
  --plan audit-out/dissolve/plan-YYYYMMDDTHHMMSSZ.json \
  --owner-pem - --execute
```

The command rechecks signing-canister ownership and key identity, then fetches
the neuron's current full record. It requires the Oracle to be the controller
and the state to be locked. Before submitting, it writes a create-only intent
record under `audit-out/dissolve/journal/`. It submits exactly one
`manage_neuron(Configure(StartDissolving))`, decodes the Governance result,
and reads the neuron again. Do not rerun after an intent exists without a
result file. An ambiguous outcome must be reconciled first.

Verify independently with the existing inspect command against the ID after
the call. Inspect the journal result as well; a successful Governance reply
alone is not the final verification.

## Batch execution and recovery

### Reconciling a mis-classified result

Early runs decoded Governance's `Configure : record {}` reply incorrectly and
journaled `governance_rejected` / `missing command response` for neurons that
were in fact dissolving. This is fixed. To reconcile such a journal entry
(read-only: one signed `list_neurons` query, no update), run:

```sh
dfx identity export sticpowner | cargo run --manifest-path tools/neuron-audit/Cargo.toml -- \
  dissolve reconcile --neuron-id <ID> --plan audit-out/dissolve/plan-YYYYMMDDTHHMMSSZ.json \
  --owner-pem - --out-dir audit-out
```

It refuses anything other than that exact mis-classification and writes a
create-only `<plan>-<id>.reconciled.json`; `start` and `batch` then skip the neuron.

Batch is sequential and requires both explicit flags. Use a protected PEM file
for repeatable per-neuron authentication (batch rejects `--owner-pem -`):

```sh
cargo run --manifest-path tools/neuron-audit/Cargo.toml -- dissolve batch \
  --plan audit-out/dissolve/plan-YYYYMMDDTHHMMSSZ.json \
  --owner-pem /secure/path/owner.pem --execute --confirm-batch
```

The command stops at the first operation not verified as successful. Verified
successful results are skipped on resume. An intent without a result or a
result marked unknown blocks a repeat. Reconcile the live neuron with a fresh
read and the journal before taking any further operator action. Already
dissolving or dissolved neurons are recorded as skipped. Never delete or edit
journal files to bypass these checks.

## Final audit record

Retain the plan, its hash, every write-once `*.intent.json` and `*.result.json`,
and a fresh post-operation `inspect` output for each attempted ID. Report
successful, skipped, rejected, unknown and unattempted IDs from those records.
No call in this workflow disburses or transfers ICP.

## Verification boundary

Offline tests cover payload encoding/decoding, the exact signer permission,
eligibility classification and journal write-once behavior. A real NNS
environment is still required to verify the deployed Governance response
shape, signing-canister availability, replica certification/polling and the
on-chain resulting state. Do not use a mainnet update during development or
testing.
