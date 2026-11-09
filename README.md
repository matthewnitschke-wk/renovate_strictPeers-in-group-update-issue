# pnpm `strictPeerDependencies` — in-range peer update in the same group — Reproduction Repo

Renovate's pnpm artifact step fails a grouped branch under `strictPeerDependencies`, even though the
update that would satisfy the peer is in the **same branch**.

## Setup

| Package     | `package.json` | Lockfile | Renovate proposes | Change type      |
|-------------|----------------|----------|-------------------|------------------|
| `react-dom` | `18.2.0`       | 18.2.0   | **18.3.1**        | `package.json` edit |
| `react`     | `^18.2.0`      | 18.2.0   | **18.3.1**        | lockfile only (in range) |

- `react-dom@18.2.0` has peer `react: ^18.2.0`
- `react-dom@18.3.1` has peer `react: ^18.3.1`, which the locked `react@18.2.0` no longer satisfies
- `pnpm-workspace.yaml` sets `strictPeerDependencies: true`
- `renovate.json` groups everything into one branch (`rangeStrategy` is the default, `auto`)

## The Problem

Renovate generates the lockfile with two commands (`lib/modules/manager/npm/post-update/pnpm.ts`),
in this order:

1. `pnpm install --no-frozen-lockfile` — for `package.json` changes (`react-dom` → 18.3.1)
2. `pnpm update --no-save react@18.3.1` — for in-range, lockfile-only updates

Step 1 installs `react-dom@18.3.1` while the lockfile still has `react@18.2.0`. That violates the
peer range, so `strictPeerDependencies` fails the install. Step 2, which would fix the peer, never runs.
The PR is created with an `Artifact update problem` comment, or no PR at all.

## Expected Behavior

Renovate should produce one `all` PR updating both `react-dom` and `react` to 18.3.1, with an updated
`pnpm-lock.yaml`. For example by running the lockfile-only `pnpm update` before `pnpm install`, so both
are resolved together.

## Workaround

`"rangeStrategy": "bump"` changes `react` to a `package.json` edit (`^18.3.1`), so step 1 installs both
packages at once and succeeds. Not suitable as a global setting for libraries.

## Reproducing

Run the **Renovate** workflow (`workflow_dispatch`) and look at the branch / PR for
`matthewnitschke-wk/renovate_strictPeers-in-group-update-issue`. It needs a `RENOVATE_TOKEN` secret.
