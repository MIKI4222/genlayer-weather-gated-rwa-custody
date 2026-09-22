# Weather-Gated Multi-Cycle RWA Custody

A reusable GenLayer Intelligent Contract primitive for weather-sensitive real-world asset custody.

The contract evaluates live weather data through independent GenLayer validators, opens an optimistic challenge period for safe release decisions, and authorizes a configured custodian to claim custody after finalization.

This is an authorization primitive for DAOs, logistics systems, tokenized physical assets, insurance workflows, and treasury adapters. It does not directly transfer tokens or perform physical handover.

## Deployed Contract

- **Network:** GenLayer Bradbury Testnet
- **Contract:** `0x4ea1c077793ACD368EBe8Ff59d8fF21A8067a32a`
- **Source:** `weather_gated_rwa_custody_submission.py`

## Why GenLayer?

A traditional deterministic smart contract cannot directly interpret live weather data or decide whether an external response is complete and safe to use.

This contract uses GenLayer to:

1. Fetch a live external weather response from inside the contract.
2. Ask an LLM to classify the response according to an explicit policy.
3. Let validators independently repeat the operation.
4. Compare the decision-bearing fields through the Equivalence Principle.
5. Apply state changes only after the consensus result is accepted.

The model's free-form explanation is stored for transparency, but authorization depends on the canonical decision and risk fields, not on reasoning text alone.

## Use Case

Consider a DAO or logistics protocol responsible for releasing a weather-sensitive shipment:

- agricultural commodities;
- construction materials;
- temperature-sensitive equipment;
- warehouse or delivery custody;
- insurance-triggered release workflows.

The owner starts a custody cycle. GenLayer validators assess live weather conditions. A `SAFE` result opens a challenge period. Any account can challenge the proposal. If the proposal is not challenged, it can be finalized and the configured custodian can claim custody exactly once.

The same deployment supports multiple independent custody cycles.

## State Machine

```text
NO_CYCLE
   |
   v
OPEN
   |
   | propose_release() -> UNSAFE
   v
CHALLENGE_PERIOD
   |                    \
   | challenge_release()  \ finalize_release()
   v                      v
CHALLENGED              RELEASED
   |                      |
   | reopen...            | claim_rwa_custody()
   v                      v
REOPENED                CLAIMED
   |
   | propose_release()
   v
CHALLENGE_PERIOD
```

An `UNSAFE` cycle can be closed by the owner with `close_unsafe_cycle()`. After a completed claim, the owner can start another cycle on the same contract.

## Weather Policy

A release is considered safe only when all of the following are true:

- temperature is between `-10` and `35` degrees Celsius;
- wind speed is no more than `60 km/h`;
- precipitation is no more than `10 mm`;
- `temperature_2m` is present;
- `wind_speed_10m` is present;
- `precipitation` is present.

Missing, invalid, contradictory, or unavailable data fails closed as `UNSAFE`.

The default source is Open-Meteo for Kyiv coordinates. The owner may configure another source before starting a cycle.

## Consensus Design

The nondeterministic operation is executed inside `gl.eq_principle.prompt_comparative()`.

Each validator independently:

1. fetches the configured weather URL with `gl.nondet.web.get()`;
2. treats the response as untrusted external data;
3. sends the data to `gl.nondet.exec_prompt()`;
4. returns a canonical result:

```text
SAFE|LOW|reason
```

or:

```text
UNSAFE|MEDIUM|reason
```

Validators must agree exactly on:

- `SAFE` or `UNSAFE`;
- `LOW`, `MEDIUM`, or `HIGH`.

The explanation may differ between validators. The contract stores only the accepted leader result after consensus.

## Roles and Permissions

### Owner

The owner is the deployer by default. The owner can:

- transfer ownership;
- configure the custodian;
- change the weather source between cycles;
- change the challenge period between cycles;
- start a new cycle;
- reopen a challenged proposal;
- close an unsafe cycle.

### Custodian

The custodian is the deployer by default and can be replaced with `set_custodian()` before a cycle starts.

Only the configured custodian can call:

```text
claim_rwa_custody(label)
```

### Challenger

Any account can call:

```text
challenge_release(evidence_url)
```

No owner permission is required for a challenge.

## Contract Methods

### Configuration

| Method | Access | Purpose |
|---|---|---|
| `transfer_ownership(new_owner)` | Owner | Transfers administrative ownership |
| `set_custodian(new_custodian)` | Owner | Sets the account allowed to claim custody |
| `set_weather_url(new_url)` | Owner | Changes the external weather source between cycles |
| `set_challenge_period(period)` | Owner | Changes the challenge period between cycles |

### Cycle Management

| Method | Access | Purpose |
|---|---|---|
| `start_cycle(label)` | Owner | Starts a new reusable custody cycle |
| `propose_release()` | Anyone | Runs weather assessment and validator consensus |
| `challenge_release(evidence_url)` | Anyone | Freezes a SAFE proposal |
| `reopen_challenged_proposal()` | Owner | Reopens a challenged proposal for fresh assessment |
| `close_unsafe_cycle()` | Owner | Closes an UNSAFE cycle |
| `finalize_release()` | Anyone | Finalizes an unchallenged SAFE proposal after the deadline |
| `claim_rwa_custody(label)` | Custodian | Claims custody exactly once after release |

### Read Methods

```text
get_status()
get_cycle_id()
get_cycle_label()
get_weather_url()
get_challenge_period()
get_decision()
get_risk()
get_reason()
get_source_used()
get_challenge_deadline()
get_challenge_count()
get_assessment_count()
get_completed_cycles()
can_finalize()
can_claim_custody()
```

## Quick Test Guide

### 1. Initial State

After deployment:

```text
get_status()          -> NO_CYCLE
get_cycle_id()        -> 0
get_completed_cycles()-> 0
```

### 2. Configure Roles

The deployer is initially both owner and custodian.

To use a different custodian before starting a cycle:

```text
set_custodian("0x...custodian-address...")
```

For a simple test, leave the default custodian unchanged.

### 3. Start a Cycle

```text
set_challenge_period(10)
start_cycle("kyiv-shipment-001")
```

Expected:

```text
get_status()   -> OPEN
get_cycle_id() -> 1
```

### 4. Run Consensus

```text
propose_release()
```

For safe weather:

```text
get_status()   -> CHALLENGE_PERIOD
get_decision() -> SAFE
get_risk()     -> LOW
```

### 5. Test a Challenge

Before the deadline:

```text
challenge_release("https://example.com/evidence")
```

Expected:

```text
get_status()         -> CHALLENGED
can_finalize()       -> false
get_challenge_count()-> 1
```

Then:

```text
finalize_release()
```

Expected: the transaction is rejected with a contract error because a challenged proposal cannot be finalized.

### 6. Reopen and Reassess

The owner calls:

```text
reopen_challenged_proposal()
propose_release()
```

A fresh consensus assessment is required.

### 7. Finalize and Claim

After a SAFE proposal survives the challenge period:

```text
can_finalize() -> true
finalize_release()
```

Then the custodian calls:

```text
claim_rwa_custody("warehouse-kyiv-01")
```

Expected:

```text
RWA custody granted to: warehouse-kyiv-01
```

A second claim is rejected.

### 8. Reuse the Same Deployment

After a completed claim:

```text
set_challenge_period(300)
start_cycle("kyiv-shipment-002")
```

The same deployment now has a new independent cycle:

```text
get_cycle_id() -> 2
get_status()   -> OPEN
```

## Verified Test Evidence

The following deployment was tested on GenLayer Bradbury Testnet:

```text
0x4ea1c077793ACD368EBe8Ff59d8fF21A8067a32a
```

Verified behaviors include:

- SAFE weather assessment through validator consensus;
- LOW risk classification;
- challenge acceptance;
- blocked finalization after challenge;
- successful proposal reopen;
- fresh assessment after reopen;
- successful finalization;
- successful custody claim;
- rejected repeated custody claim;
- multiple cycles on the same deployment.

Selected transaction IDs:

```text
SAFE assessment:
0xfec7df4b1d198f3c9e47a065154031975a20500490fa459058a07c6d73f949fc

Challenge:
0x92b26e80391a1ad21140fa9c1978f9d5b0fb0844812f7303e8784d6c3409ffe7

Blocked finalization:
0xecf8b470946eaf106cceaf10fed9b64e4bad13e13256362b0736a9fb437a1d00

Reopen:
0x739ddb5bcd7124f835734dafa7c0d7701e232a972e37a72c6f2395d54c60083f

Reassessment:
0xec910cda86f94c1f388dd68ac7988b1521be558d6c2ca157e71cbb680833eda2

Finalization:
0xda9f71404407451b34b9e6efddcef70b64141e3b7c8d43f592c3277fa1bdb878

Custody claim:
0x721e3b8affe774e37f1128ff1496674a68a8d83cd66da402743107335b78df17
```

## Limitations

- The contract records custody authorization; it does not itself transfer native tokens or physical assets.
- Challenge evidence is recorded as a URL and acts as a veto signal. It is not automatically adjudicated by a second evidence-verification flow in this version.
- The owner controls the configured weather source and policy period.
- External web availability can cause consensus delays or leader timeouts.
- The contract should be integrated with a treasury, custody, logistics, or physical handover adapter for production use.

## License

MIT
