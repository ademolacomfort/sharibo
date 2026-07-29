# Sharibo Smart Contracts

This directory contains the Soroban smart contracts for **Sharibo**, private rotating savings circles on Stellar. The payout of the shared pot is anonymized by a real Groth16 zero-knowledge proof, verified on-chain.

The core implementation is located in [`contracts/sharibo/src/lib.rs`](sharibo/src/lib.rs), and its accompanying test suite is located in [`contracts/sharibo/src/test.rs`](sharibo/src/test.rs).

---

## 1. Building the Contracts

Before compiling the contracts, ensure you have the proper Rust toolchain installed.

### Prerequisites

- **Rust + WASM target**:
  ```bash
  rustup target add wasm32v1-none
  ```
- **Stellar CLI**: Ensure you have installed the current `stellar` CLI (superseding the old `soroban` CLI).

### Build Command

To build the contract and compile it to WebAssembly (WASM):

```bash
stellar contract build
```

The compiled WASM artifact will be generated at `target/wasm32v1-none/release/sharibo.wasm`.

---

## 2. Testing the Contracts

The contract test suite utilizes Soroban's testing environment and verifies the full ZK-proof verification logic locally using test fixtures.

To execute the test suite, run the following command from the `contracts/` directory:

```bash
cargo test
```

This will compile and run all the unit and benchmark tests (21 tests in total), including verification of happy path claims with a real Groth16 proof, authorization checks, and error condition tests.

---

## 3. Deploying the Contracts

### Required CLI

Deployments are performed using the `stellar` CLI.

### Deployment Commands

1. **Deploy the WASM contract onto Testnet**:
   ```bash
   stellar contract deploy \
     --wasm target/wasm32v1-none/release/sharibo.wasm \
     --source admin \
     --network testnet
   ```
   *This command returns the Contract ID (e.g., `CB64IZIBBSPUY63UMIVACKWDKRFNH6WJ2EPAOLM7QR4ZI6IJOT4N2LCF`), which should be recorded in your environment variables.*

2. **Retrieve the Test Token ID (using native XLM Stellar Asset Contract on Testnet)**:
   ```bash
   stellar contract id asset --asset native --network testnet
   ```

---

## 4. Contract API Reference

Below is the documentation for all public contract methods.

### `create_circle`

* **Signature**:
  ```rust
  pub fn create_circle(
      env: Env,
      admin: Address,
      token: Address,
      root: Fr,
      contribution: i128,
      size: u32,
      vk: VerificationKey,
  ) -> u64
  ```

* **Purpose**:
  Allows an administrator to initialize a new rotating savings circle with a designated payment token, Merkle root containing member commitments, expected contribution amount per member, total circle size (number of members), and the Groth16 verification key (`vk`).

* **Preconditions**:
  * The admin must authorize the transaction (`admin.require_auth()`).
  * The contribution amount and circle size must be valid and must not result in an integer overflow when multiplied to determine the pot target.

---

### `fund`

* **Signature**:
  ```rust
  pub fn fund(env: Env, circle_id: u64, from: Address)
  ```

* **Purpose**:
  Deposits exactly one `contribution` amount of tokens into the designated circle's pot for the current round.

* **Preconditions**:
  * The funder must authorize the transfer (`from.require_auth()`).
  * The circle associated with `circle_id` must exist and must **not** be cancelled.
  * The current round's pot must not be full. If the pot has already reached the target (`contribution * size`), further contributions are blocked.
  * The funder must hold a sufficient balance of the circle's configured token.

* **Open Funding Design**:
  Funding is intentionally unshielded and public. Any address can call `fund` on behalf of a circle (not restricted to Merkle root members). This allows external benefactors to top up community pots.

---

### `claim`

* **Signature**:
  ```rust
  pub fn claim(
      env: Env,
      circle_id: u64,
      recipient: Address,
      nullifier_hash: Fr,
      external_nullifier: Fr,
      proof: Proof,
  )
  ```

* **Purpose**:
  Anonymously pays out the full round pot (`contribution * size`) to the designated `recipient` address upon presenting a valid Groth16 zero-knowledge proof of membership.

* **Preconditions**:
  * The circle associated with `circle_id` must exist and must **not** be cancelled.
  * The pot must be fully funded (`pot == contribution * size`).
  * The provided `external_nullifier` must match the expected SHA-256 round tag of the current round, computed as `SHA256(circle_id, round) mod r`. This binds the proof to the exact circle and round.
  * The `nullifier_hash` must **not** have been previously used for any claim in this circle.
  * The Groth16 ZK proof must verify successfully against the circle's stored verification key (`vk`) and public inputs (`[nullifier_hash, root, external_nullifier]`).

* **Postconditions**:
  * The nullifier hash is marked as spent in persistent storage.
  * The entire pot balance is transferred to the `recipient` address.
  * The circle's `pot` is reset to `0`, the `round` is incremented by `1`, and the `contributors` list is cleared.

---

### `get_circle`

* **Signature**:
  ```rust
  pub fn get_circle(env: Env, circle_id: u64) -> Circle
  ```

* **Purpose**:
  A view method to retrieve the complete public state and configuration of a circle (e.g., admin, token, Merkle root, round, current pot, and contributors).

* **Preconditions**:
  * The circle associated with `circle_id` must exist.

---

## 5. Error Code Reference

When a transaction reverts, Soroban returns a typed contract error of the form `Error(Contract, #Code)`. Below is the complete table of error codes defined in the contract:

| Code | Error Name | Trigger / Cause | What the Caller Should Do |
| :---: | :--- | :--- | :--- |
| **1** | `CircleNotFound` | The specified `circle_id` does not exist in persistent storage. | Verify that the circle ID is correct and was successfully created. |
| **2** | `RoundNotFunded` | `claim` was called on a circle whose pot has not yet reached the required target size (`contribution * size`). | Ensure that the required number of contributors have successfully called `fund` for this round. |
| **3** | `WrongRoundTag` | The presented `external_nullifier` does not match the expected SHA-256 round tag (`SHA256(circle_id, round) mod r`) of the current round. | Re-generate the proof with the correct round tag matching the circle's current round number. |
| **4** | `AlreadyClaimed` | The `nullifier_hash` presented in `claim` has already been recorded in persistent storage as claimed. | Do not attempt to reuse a spent nullifier. Each member may only claim once per circle/round. |
| **5** | `InvalidProof` | The Groth16 pairing check failed, or the public signal order/values did not match the proof statement. | Verify that the zero-knowledge proof was correctly generated, utilizing the correct secret, nullifier, path elements, and verification key. |
| **6** | `RoundFull` | `fund` was called on a circle whose pot is already fully funded. | Wait for the current round to be claimed and advanced before attempting to fund the next round. |
| **7** | `Overflow` | Checked arithmetic failed during contribution calculation or pot addition. | Avoid using absurdly large contribution amounts or circle sizes that overflow integer capacities. |
| **8** | `CircleCancelled` | `fund`, `claim`, or `cancel_circle` was called on a circle that has already been cancelled. | Do not interact with a cancelled circle. Any funds were already refunded to the contributors. |

---

## 6. Test Coverage

The accompanying test suite in [`contracts/sharibo/src/test.rs`](sharibo/src/test.rs) contains **21 tests** verifying the correctness and robustness of the smart contract's state machine, ZK-verification path, and auxiliary mechanisms.

### Key Scenarios Covered

1. **Happy Path Payout**:
   - `happy_path_round_pays_out_and_advances`: Verifies that a fully funded round with a real valid Groth16 proof successfully transfers the pot to a fresh recipient, resets the pot, and increments the round.
2. **Rejection & Error Paths**:
   - `claim_reverts_on_tampered_public_input`: Verifies that `claim` panics with `Error::InvalidProof` when a tampered or invalid nullifier hash is submitted.
   - `claim_reverts_when_underfunded`: Ensures that a claim fails with `Error::RoundNotFunded` if any of the members have not funded.
   - `second_claim_with_same_nullifier_reverts`: Asserts that a nullifier cannot be replayed across rounds, reverting with `Error::AlreadyClaimed`.
   - `claim_reverts_on_stale_round_tag`: Asserts that providing a round tag for a different round reverts with `Error::WrongRoundTag`.
3. **Authorization**:
   - `create_circle_requires_admin_auth` and `fund_requires_member_auth`: Enforces that admin and funder authorizations are properly checked.
4. **Edge Cases**:
   - `sixth_fund_on_full_round_reverts`: Verifies that a sixth deposit on a 5-member circle is blocked with `Error::RoundFull` to prevent over-funding and bricking the claim.
   - `anyone_can_fund`: Verifies the open funding model, ensuring non-member addresses can contribute to a pot.
   - `fund_reverts_on_pot_target_overflow`: Ensures checked multiplication catches overflows.
5. **Cancellations & Refunds**:
   - `cancel_refunds_partial_funders_and_closes_circle`: Verifies that a circle admin can cancel an underfunded circle, automatically refunding all current round contributors in FIFO order and closing the circle permanently.
6. **State Persistence**:
   - `instance_ttl_extended_after_create_fund_claim`: Ensures that `extend_ttl` is executed on all write operations (`create_circle`, `fund`, `claim`) to prevent instance-storage archival issues.
7. **Gas / CPU Benchmarking**:
   - `cpu_instruction_benchmarks`: Benchmarks and prints the precise CPU instructions consumed by write operations (e.g., `create_circle`, `fund`, `claim`) and asserts that they remain safely under the 100M limit.
