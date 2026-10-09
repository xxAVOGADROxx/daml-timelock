# daml-timelock

A small, tested **Timelock for Canton (Daml 3.x)**, modeled on OpenZeppelin's EVM `TimelockController`.

> Learning/portfolio project. **Not audited, not production code.**

## What a timelock is

A timelock forces privileged actions to wait a minimum delay between being *scheduled* and being *executed*. During that window the people affected can see what is coming and react (exit, object, or have a guardian cancel it), so a compromised or malicious admin cannot change the system instantly.

## Design overview

| Party | Role |
|---|---|
| `admin` | Proposer. Signatory of the config and of every pending action. Schedules actions. |
| `executor` | Executes an action once `ledger time >= eta` (may be the same party as admin). |
| `guardians` | Any one of them can cancel a pending action. |
| `watchers` | Observers only: can *see* pending actions, can do nothing. |

Contracts (`timelock/daml/Timelock.daml`):

- **`Timelock`** – config (`admin, executor, guardians, watchers, minDelay`). Signatory `admin`; observers `executor, guardians, watchers`. `ensure minDelay > 0`.
  - `Schedule` (nonconsuming, controller `admin`): requires `eta >= now + minDelay`, creates a `PendingAction`.
  - `ApplyDelayUpdate` (controller `admin`): changes `minDelay`, but only given an `ExecutedAction` receipt for an `UpdateDelay` action (see below).
- **`PendingAction`** – the scheduled `TimelockedAction` and its `eta`.
  - `Execute` (controller `executor`): requires `now >= eta`. *Consuming*, so it can run once.
  - `Cancel` (flexible controller `canceller`): the body requires `canceller ∈ guardians`. Consuming.
- **`ExecutedAction`** – receipt/evidence of an execution. Signatories `admin` **and** `executor`.
- **`TimelockedAction`** – a plain sum type: `UpdateParameter`, `UpdateDelay`, `TransferTreasury`. Only `UpdateDelay` has a real effect; the others are just recorded.

### Design decisions

- **Changing the delay goes through the delay.** There is no `SetDelay` choice. `UpdateDelay` is a `TimelockedAction`; when executed, `Execute` creates an `ExecutedAction` and passes it to `ApplyDelayUpdate`. The admin alone cannot forge that receipt (it needs the executor's signature), and the receipt is bound to one `Timelock` contract id, which is archived when the delay changes, so it cannot be replayed.
- **No contract keys** (unsupported on Canton 3): `PendingAction` stores `timelockCid` explicitly.
- **Ledger time** is read with `getTime` inside choices; tests move it with `setTime` / `passTime`.
- **Daml authority**: inside a choice body, authority = controller + the contract's signatories. That is why `Execute` (controller `executor`, signatory `admin`) can create the `ExecutedAction` that needs both signatures.
- **Cancel's flexible controller**: any party may *attempt* it, so the body checks guardian membership. A party that cannot even see the contract cannot name it anyway.

## Privacy design (the important part)

On EVM, every pending operation is public on-chain: anyone can watch the mempool/events. On Canton there is no global state. A party only receives contracts it is a **stakeholder** (signatory or observer) of. So `PendingAction` is visible *only* to `admin`, `executor`, `guardians` and `watchers`; everybody else (including other participants on the same synchronizer) learns nothing, not even that it exists. The test `testPrivacy` proves this with per-party queries.

Consequence: **choosing the observer set is part of the security model.**

- *Too few observers*: affected users never see the pending action, so the delay gives them no chance to react. The timelock becomes security theatre.
- *Too many observers*: you leak governance intent (e.g. a treasury transfer or parameter change) to parties with no need to know, possibly allowing front-running or information leakage.

Here, `watchers` are meant to be the users of the governed system. Note that the watcher list is fixed in the `Timelock`; changing it means a new `Timelock` contract (not implemented).

## Security considerations

Covered by tests (`tests/daml/TimelockTest.daml`, each test states what it proves):

- Happy path: schedule → wait → execute; schedule → guardian cancel → execute fails; delay update through the timelock.
- Must fail: execute before `eta`; schedule with `eta < now + minDelay`; non-executor executes; non-guardian cancels (including impersonation); double execution; admin bypassing the delay (forged receipt, wrong-action receipt, replayed receipt); zero delay.
- Privacy: an unrelated party sees neither the pending action, nor the timelock config.

Known limitations:

- **Executor never executes**: nothing forces execution; an action may stay pending forever (no expiry/grace period like some designs).
- **Admin key compromise**: the attacker can schedule anything, but only after `minDelay`, and guardians can cancel. If the attacker also controls the executor, only guardians stand in the way.
- **Guardian collusion/compromise**: a single guardian can cancel any action (censorship). There is no threshold.
- **Stale `timelockCid`**: after a delay update, pending actions scheduled earlier reference the archived `Timelock`; executing an `UpdateDelay` among them will fail. Other actions still execute.
- **Wall-clock vs ledger time**: Canton ledger time is bounded-skew relative to wall-clock, not exact; do not rely on second-level precision.
- The non-`UpdateDelay` actions have no real effect (demo only).

## Build and test

Requires JDK 17+ and the Daml SDK 3.6.0 via [`dpm`](https://docs.digitalasset.com) (the Daml Package Manager).

```bash
dpm build --all     # builds timelock/ then tests/ (multi-package.yaml)
cd tests && dpm test
```

The code lives in a separate package from the tests so the deployable DAR does not depend on `daml-script`.

## Learning-project note

This was written to learn Daml/Canton coming from Haskell and Cardano (PlutusTx/Aiken). It has not been audited and should not be used to guard real assets.
