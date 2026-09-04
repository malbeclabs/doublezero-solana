---
name: solana-program-review
description: Use when reviewing on-chain program (Rust) code or PRs in this repo — Solana programs (passport, revenue-distribution) using bytemuck zero-copy + 8-byte precomputed discriminator. Covers account/signer validation, PDA seeds+bump, CPI safety, checked arithmetic, rent/space/realloc, access control, serialization, error handling, and testing.
---

# Reviewing on-chain program changes in doublezerofoundation/doublezero-solana

This skill distills the review conventions actually applied in this repo's PRs (feedback range **2025-08 through 2026-04**). These are Solana programs (**passport**, **revenue-distribution**) built on **bytemuck zero-copy Pod accounts** with an **8-byte precomputed discriminator**, read via `ZeroCopyAccount::try_next_accounts`. Accounts carry a `StorageGap`, guard their layout with `const _: () = assert!(size_of::<T>() == N)`, and integrate other programs over CPI (the rewards-integration path). Keep that model in mind for every review.

**Scope note**: Do not import the borsh + 1-byte `AccountType` enum idioms used by `malbeclabs/doublezero`'s serviceability/geolocation programs — those are a different serialization framework than this repo's bytemuck zero-copy + 8-byte discriminator, not just a different program. This is a framework distinction, not a network one: `malbeclabs/doublezero-shreds` runs on the same DoubleZero Ledger as `malbeclabs/doublezero` yet also uses bytemuck zero-copy + 8-byte discriminators. So when borrowing guidance from elsewhere, check which serialization framework the source program actually uses, not which repo or chain it lives on.

Apply the classes below **class-by-class against the diff**, highest-frequency conventions first. Each class gives the current standard, the on-chain risk, diff signals to scan for, and real review quotes with PR links.

---

## How to apply — severity and voice

Not every finding carries the same weight. Sort each one before raising it.

**Blocking (fix before merge)** — real correctness or safety defects:
- A missing owner or discriminator check on an account that is **not** loaded through `ZeroCopyAccount` (a raw `AccountInfo` read that trusts unverified bytes).
- Arithmetic that can over/underflow on a count, index, or lamport/balance value.
- A collection cap that isn't enforced at the mutating instruction (see the heap-cap hazard below).
- A missing changelog entry when CI enforces one.

**Nit (raise, but non-blocking)** — naming/idiom drift, debug/`Display` representations, cosmetic wording. Label these explicitly as nits so the author can weigh them.

**Follow-up (defer)** — broad cosmetic sweeps and refactors that don't belong to this diff. Note them and move on; don't hold the PR.

**Voice**:
- Frame judgment calls as questions ("should this be `>=`?", "can this checked-sub ever be `None`?") rather than directives.
- Give the exact replacement code, not a prose description of it.
- Prove any serialization/layout claim with a runnable `#[test]` (or a `const _: () = assert!(size_of::<T>() == N)`), not an assertion in the comment thread.
- Distinguish "this is more than just a suggestion" (blocking) from "up to you" (optional) so the author knows what must change.

---

### Sibling-processor parity *(reviewer-curated)*

**Current guidance**: A new instruction handler should mirror its **closest established sibling** unless the diff states a reason to diverge. Reviewing a new handler in isolation is how drift slips in — the handler reads fine on its own but quietly differs from the pattern every other handler follows. Compare against the nearest existing `try_*` processor and confirm it matches on:
- **Account loading + validation shape** — same `ZeroCopyAccount::…::try_next_accounts` usage, same set of `is_signer`/owner/discriminator checks in the same order.
- **Authority gating** — same `VerifiedProgramAuthority` / upgrade-authority routing, same read-only-authority + separate-payer split.
- **Status / journal handling** — same replay-bit discipline, same journal-level aggregation, same idempotency guard.
- **Counter discipline** — increments/decrements the same snapshot/collected counters the sibling does, in the same place, guarded the same way.

**What to look for in a diff**:
- A new `try_*` handler that loads accounts differently from the sibling it's modeled on (raw fetch vs. `try_next_accounts`, different check order).
- Authority gating that differs from sibling handlers without a stated reason.
- Missing status/replay-bit or journal update that every sibling performs.
- A counter (snapshot/collected/bitmap) that the sibling increments/decrements but this handler doesn't — or updates in a different, unguarded spot.

---

### naming & idiom / code style

**Current guidance** (as of 2026-04-21): Names must be precise and consistent. Suffix `AccountInfo` bindings with `_info`. Rename types/vars to match their true meaning (`SolanaValidatorPayment` -> `SolanaValidatorDebt`; a bare `doublezero` -> `doublezero_ledger`). Use a `get_` prefix on fetchers. From an integration's perspective, prefer `associated_`/`parent_`/`canonical_` prefixes over a program-specific abbreviation. Reuse the existing `new_transaction` helper rather than re-rolling transaction construction. Provide `From` conversions instead of ad-hoc casts. Express domain quantities in the domain's own units (slots/epochs over human time). Use a `//` separator/summary line to describe the next several lines, not each individual line.

**What to look for in a diff**:
- `AccountInfo` bindings missing the `_info` suffix.
- Type/variable names that no longer describe what the value is (payment vs. debt, ambiguous `doublezero`).
- Fetcher functions without a `get_` prefix.
- Integration-facing bindings named after this program's internals instead of `associated_`/`parent_`/`canonical_`.
- Hand-rolled transaction construction that could call `new_transaction`.
- Ad-hoc `as` casts where a `From` impl belongs.
- Time-like constraints expressed in human time instead of slots/epochs.

**Examples**:
- "Nit: can we add `_info` suffix?" — on `use doublezero_program_tools::account_info::{ try_next_enumerated_account, ... };` [doublezerofoundation/doublezero-solana#115](https://github.com/doublezerofoundation/doublezero-solana/pull/115)
- "Could we actually rename `rev_distr_distribution` to something like `associated_distribution`, `parent_distribution` or `canonical_distribution`? These are how an integration might label the Revenue Distribution's distribution PDA" [doublezerofoundation/doublezero-solana#115](https://github.com/doublezerofoundation/doublezero-solana/pull/115)
- "I'm really sorry, but can we name this `get_access_requests`?" [doublezerofoundation/doublezero-solana#30](https://github.com/doublezerofoundation/doublezero-solana/pull/30)
- "It may make more sense to put these sorts of time-like constraints in terms of Solana-terms (slots and epochs) as opposed to human time" [doublezerofoundation/doublezero-solana#30](https://github.com/doublezerofoundation/doublezero-solana/pull/30)

---

### account (de)serialization & zero-copy

**Current guidance** (as of 2026-04-23): For zero-copy Pod structs, put the seed/bump as the **first field** of the struct, size fields to their real range while keeping 8-byte alignment (add explicit `_padding`, prefer padding placed near related fields), and name types by intent (e.g. `UnitShare32`). Avoid magic bit masks — define named mask/shift consts for flag bits. When a bespoke instruction selector is used, use the **standard 8-byte discriminator** (`try_from_slice` into the shared enum) rather than a one-byte selector, and fall through to the program's own instruction enum on `Err`. Factor repeated balance-reads into a small `#[inline(always)]` deserialization helper.

**Why it matters**: A one-byte instruction selector collides with the standard 8-byte discriminator space, and Pod field order/alignment errors corrupt how bytes are read.

**What to look for in a diff**:
- New Pod struct where seed/bump is not the first field.
- Fields wider than needed, or missing explicit `_padding` to preserve 8-byte alignment.
- Raw bit masks/shifts instead of named `*_BIT` / mask consts.
- A one-byte instruction selector instead of the 8-byte discriminator + `try_from_slice` with `Err(_)` fallthrough to the program enum.
- Repeated inline balance deserialization that should be one `#[inline(always)]` helper.

**Examples**:
- "I don't think we should have a one-byte discriminator for the withdraw instruction selector. We should use the standard 8-byte one. And so we can do... match IntegrationInstructionData::try_from_slice(data) { Ok(ix) => ... Err(_) => { let ix = BorshDeserialize::try_from_slice(data)... } }" — on `pub enum IntegrationInstructionData { WithdrawIntegrationRewards, }` [doublezerofoundation/doublezero-solana#116](https://github.com/doublezerofoundation/doublezero-solana/pull/116)
- "Prefer the seed itself be the first element of the struct. Can you change that?" — on `pub struct RewardsIntegration { pub bump_seed: u8, _padding: [u8; 7], pub program_id: Pubkey, }` [doublezerofoundation/doublezero-solana#113](https://github.com/doublezerofoundation/doublezero-solana/pull/113)
- "I like that. The mask here does feel like a magic mask. I'll push a change" — around `pub const FLAG_IS_BLOCKED_BIT: usize = 31;` [doublezerofoundation/doublezero-solana#47](https://github.com/doublezerofoundation/doublezero-solana/pull/47)
- "Maybe have an `#[inline(always)]` private fn that does a balance deserialization?" — in `fn try_collect_integration_rewards(accounts: &[AccountInfo]) -> ProgramResult {` [doublezerofoundation/doublezero-solana#116](https://github.com/doublezerofoundation/doublezero-solana/pull/116)

---

### account & signer validation

**Current guidance** (as of 2026-04-22): Validate accounts and signers only where the runtime doesn't already guarantee it. Trim redundant checks: don't re-check an account is writable when the CPI/account-meta already requires it, don't re-verify a program account that a downstream CPI will implicitly require, and lean on the system program's own revert for already-created accounts rather than hand-rolling an owner check. Conversely, **do add missing signer checks the runtime won't catch** (e.g. verifying the associated distribution's `info.is_signer`). Prefer the shared `ZeroCopyAccount::try_next_accounts` helpers over ad-hoc account fetching.

**Hardcoded expected program id**: When checking an account's `owner` against another program's id (e.g. the rewards-integration program), the expected id **must** come from a hardcoded in-repo constant — never from an instruction argument or a caller-passed account. An "expected owner" supplied by the caller is no check at all: the caller just names whichever program it wants and the comparison trivially passes.

**Why it matters**: A missing signer check is a security hole the runtime won't catch — redundant checks just cost compute.

**What to look for in a diff**:
- Writable re-checks on an account whose account-meta already forces writability.
- Owner/program-account checks that a downstream CPI would already enforce.
- Hand-rolled "already initialized" owner checks instead of letting the system program revert.
- Missing `info.is_signer` assertions on accounts that must sign (e.g. the associated distribution).
- Ad-hoc account iteration instead of `ZeroCopyAccount::try_next_accounts`.
- An owner check where the expected program id comes from an instruction arg or a caller-passed account instead of a hardcoded in-repo constant.

**Examples**:
- "This actually isn't necessary because your instruction is already trying to invoke rewards_integration.program_id at line 1994. If the integration program weren't provided as an account, your CPI call would fail" — on `// - 6: Integration program (executable, must match rewards_integration.program_id).` [doublezerofoundation/doublezero-solana#116](https://github.com/doublezerofoundation/doublezero-solana/pull/116)
- "We don't have to check that this is writable in this instruction, do we? As long as we pass the account meta from the EOA for the collect instruction, the Revenue Distribution program should be fine" — in `fn try_collect_integration_rewards(...)` [doublezerofoundation/doublezero-solana#116](https://github.com/doublezerofoundation/doublezero-solana/pull/116)
- "I'm looking at this again, and it feels like we should use `ZeroCopyAccount::Distribution::try_next_accounts` and then also check that the associated distribution's info.is_signer is true. What do you think?" [doublezerofoundation/doublezero-solana#115](https://github.com/doublezerofoundation/doublezero-solana/pull/115)
- "This is fine, but we don't need this (can just lean on the system program revert when it tries to create an already-created account)" — on `if new_rewards_integration_info.owner == &ID { msg!("Rewards integration already initi...` [doublezerofoundation/doublezero-solana#113](https://github.com/doublezerofoundation/doublezero-solana/pull/113)

---

### rent / account space / realloc

**Current guidance** (as of 2026-04-16): Reserve generous forward-compatible space in every account: keep a `StorageGap` and add extra reserved gap/flags fields (e.g. reserve 8 bytes for a future `_flags: Flags`, bump a gap from 1 to 4) so new fields can be added without a migration. Add compile-time `size_of` asserts so any accidental layout change is caught and tied to a deliberate migration. Fold optional deposits directly onto the rent-exemption amount (via an `additional_lamports` option on the create-account recipe) so an admin cannot misconfigure a deposit below the rent minimum; this also avoids extra rent syscalls. Don't request extra lamports for token accounts.

**Heap-cap hazard** *(reviewer-curated)*: If a diff grows an account-stored collection (a `Vec` or other dynamically-sized region), its maximum size **must** be enforced at the mutating instruction — the handler that appends/reallocs — not just documented. An over-grown account overflows the 32KB program heap when it's later loaded, which bricks **every** instruction that reads that account, not only the one that grew it. Pair the cap check with a near-cap boundary test (fill to `cap - 1`, then `cap`, then assert the `cap + 1` append reverts).

**Why it matters**: Undetected layout drift forces a migration once live, and a deposit added separately from rent risks falling below the rent-exempt minimum.

**What to look for in a diff**:
- New account struct without a `StorageGap` or with no reserved flags/gap headroom.
- Layout change without an accompanying `const _: () = assert!(size_of::<T>() == N)` update.
- A deposit added separately from rent (risk of falling below rent-exempt minimum) instead of via `additional_lamports`.
- An extra `Rent::get`/rent syscall that could be avoided.
- Extra lamports requested for a token account.
- A diff that grows an account-stored `Vec`/dynamic region with no cap check at the mutating instruction, or no near-cap boundary test for it.

**Examples**:
- "Also can we reserve 8 bytes for flags just in case? Can just add `_flags: Flags` and comment that this is reserved" — on `pub struct RewardsIntegration { ... _storage_gap: StorageGap<1>, }` [doublezerofoundation/doublezero-solana#113](https://github.com/doublezerofoundation/doublezero-solana/pull/113)
- "These compile-time checks ensure that if these consts were to change, that it would be intentional (and should be associated with a program account migration)." — on `const _: () = assert!(size_of::<ContributorRewards>() == 600); const _: () = assert!(size_of::<Distribution>() == 448);` [doublezerofoundation/doublezero-solana#109](https://github.com/doublezerofoundation/doublezero-solana/pull/109)
- "I think we should have an optional additional lamports argument in the create account recipe... By adding it to the rent exemption amount, we also guarantee the admin not misconfiguring the deposit, which will have to be greater than the minimum amount for rent exemption..." [doublezerofoundation/doublezero-solana#16](https://github.com/doublezerofoundation/doublezero-solana/pull/16)
- "Simplify by removing this (and avoid another rent syscall). It is okay that the deposit is on top of the rent for the access request." [doublezerofoundation/doublezero-solana#16](https://github.com/doublezerofoundation/doublezero-solana/pull/16)

---

### instruction/data validation

**Current guidance** (as of 2026-04-22): Keep on-chain instruction handlers dumb: read validation-only data (signatures, IDs) from instruction data and let the offchain process do heavy verification, at least for early iterations. Do not persist in account state anything the offchain reader can get straight from the instruction. Validate config invariants at the point of setting (e.g. fee must be `<` deposit). **Enforce idempotency in the program itself** — don't rely on an external integration to stop the program from collecting/processing twice.

**Why it matters**: Trusting an external integration for idempotency instead of guarding in-program lets a repeated call double-collect rewards.

**What to look for in a diff**:
- On-chain signature/heavy verification that the offchain process already does.
- Account state storing data the offchain reader can read straight from the instruction.
- A collect/process instruction with no in-program guard against being called twice on the same target.
- Config setters that don't validate invariants (fee `<` deposit) at set time.

**Examples**:
- "What happens if we call this instruction multiple times on the same integration? We aren't enforcing the integration to prevent subsequent calls, are we? I don't think we can count on the integration to prevent the Revenue Distribution program from trying to collect multiple times" — in `fn try_collect_integration_rewards(...)` [doublezerofoundation/doublezero-solana#116](https://github.com/doublezerofoundation/doublezero-solana/pull/116)
- "I would leave all the validation checks to the offchain piece and keep this program dumb, at least for the first iteration" — in `fn try_request_access(...)` [doublezerofoundation/doublezero-solana#16](https://github.com/doublezerofoundation/doublezero-solana/pull/16)
- "We actually do not need this data in this account because the offchain process should just read this data from the instruction." — on `ProgramConfiguration::AccessRequestDeposit { request_deposit_lamports, request_fee_lamports }` [doublezerofoundation/doublezero-solana#16](https://github.com/doublezerofoundation/doublezero-solana/pull/16)
- "Remove all of this. This instruction simply creates the access request account. The signature etc will be read by the offchain process to perform the sig verify. This native program cannot be CPI'ed into anyway." [doublezerofoundation/doublezero-solana#16](https://github.com/doublezerofoundation/doublezero-solana/pull/16)

---

### documentation

**Current guidance** (as of 2026-04-21): Keep the CHANGELOG accurate: add a version heading for each prior change and describe the entry with the actual scope and correct PR number. Comments should describe the block that follows (place a blank line + short summary before several lines rather than annotating each line). Update layout comments when field sizes change (e.g. storage-gap accounting). Strip dependencies and doc noise that are out of scope for the PR.

**What to look for in a diff**:
- CHANGELOG entry missing a version heading for the previous release, or with a wrong/placeholder PR number.
- Changelog wording that overstates or misdescribes the actual scope.
- Layout/size comments not updated after a struct field change.
- New dependencies or doc noise unrelated to the PR's purpose.
- Per-line comments where one summary line above the block would do.

**Examples**:
- "We need a `## [v0.3.4]` heading for the previous change" — on a `## Unreleased` block [doublezerofoundation/doublezero-solana#113](https://github.com/doublezerofoundation/doublezero-solana/pull/113)
- "\`\`\`suggestion\n- scaffold integration harvesting (#115)\n\`\`\`" — correcting `- scaffold rev-distr for integration harvesting (#1057)` [doublezerofoundation/doublezero-solana#115](https://github.com/doublezerofoundation/doublezero-solana/pull/115)
- "Out of scope. We can remove this dependency" — on `solana-ed25519-program = "2"` [doublezerofoundation/doublezero-solana#16](https://github.com/doublezerofoundation/doublezero-solana/pull/16)
- "The space is meant to briefly describe the next several lines (as opposed to writing a comment immediately above a code line or scope)" [doublezerofoundation/doublezero-solana#46](https://github.com/doublezerofoundation/doublezero-solana/pull/46)

---

### state & invariants

**Current guidance** (as of 2026-04-20): Model accounting invariants so a single stuck distribution can't block others: aggregate SOL/debt at the **journal level** rather than requiring every distribution to fully collect its own debt before sweeping. Where an ordering feels load-bearing, confirm it isn't — it doesn't matter *when* a replay-protection bit is set within an instruction as long as it isn't already set and all required state changes land by the end of the instruction. Track cross-account counters with snapshots (`integrations_count_snapshot` vs `integrations_collected_count`) to gate downstream steps. Relax over-strict rules that reject valid states (e.g. allow `>=` where `==` was used).

**Why it matters**: An over-strict per-distribution rule can wedge the whole sweep, and an exact `==` gate can wrongly reject valid advanced states.

**What to look for in a diff**:
- Sweep/collect logic that requires each distribution to fully settle its own debt before proceeding.
- Exact-equality (`==`) gates on counters where `>=` is the correct relation.
- Replay-protection bits set in a way that assumes a specific ordering — verify the "already set?" check and that all state changes complete within the instruction.
- Snapshot counters used to gate downstream steps.

**Examples**:
- "There is a bug in the sweep distribution tokens instruction. There should not be a hard rule that all debt needs to be accounted for. We should be able to sweep 2Z into the distribution if the journal says there is enough SOL in its balance" [doublezerofoundation/doublezero-solana#45](https://github.com/doublezerofoundation/doublezero-solana/pull/45)
- "It actually doesn't matter when this bit is set in the instruction, as long as we check that it hasn't been set already. As long as all state variables that have to change end up being changed by the end of the instruction, it shouldn't matter whether it is set here or after" — in `fn try_initialize_swap_destination(...)` [doublezerofoundation/doublezero-solana#45](https://github.com/doublezerofoundation/doublezero-solana/pull/45)
- "Can we do >= instead?" — on `pub fn are_all_integrations_collected(&self) -> bool { self.integrations_collected_count == self.integrations_count_snapshot }` [doublezerofoundation/doublezero-solana#115](https://github.com/doublezerofoundation/doublezero-solana/pull/115)

---

### PDA seeds + bump checks

**Current guidance** (as of 2026-04-23): Choose PDA seeds that stay stable and unambiguous for the program's whole future, not just today's caller. Prefer a **single canonical seed** used identically by every integration (e.g. `b"integration_distribution"`) over letting each integration pick its own, and expose a shared `find_*_address` helper so external programs cannot mis-derive. Cache bump seeds on the account at initialization. When picking between two candidate seed keys, favor the one that generalizes to future onboarding use cases (e.g. `service_key` over `validator_id`).

**Why it matters**: Per-integration custom seeds are easy to mis-derive; a canonical seed plus a shared `find_*_address` helper stops external programs from computing the wrong PDA.

**What to look for in a diff**:
- Integration-specific/custom PDA seeds instead of one canonical seed prefix.
- PDA derivation duplicated in a caller instead of a shared `find_*_address` helper.
- Account that derives from a bump but doesn't cache `bump_seed` at initialization.
- A seed key that won't generalize to future onboarding (e.g. narrow `validator_id` vs. broader `service_key`).

**Examples**:
- "We may want to not allow custom seeds per integration. May be hard to track down which integrations seed their own integration distribution. I think the seed should be the same for all integrations (like `b\"integration_distribution\"`)." [doublezerofoundation/doublezero-solana#116](https://github.com/doublezerofoundation/doublezero-solana/pull/116)
- "Would you want to instead implement a `find_integration_distribution_address` method in the integration submodule, which `MockIntegrationDistribution::find_address` can call? It could be too easy for an integration to mess up how to derive its account without a find address method the Revenue Distribution's integration module can deliver" [doublezerofoundation/doublezero-solana#116](https://github.com/doublezerofoundation/doublezero-solana/pull/116)
- "Now that I'm looking at this, we should cache the integration program's bump seed... Wanted to make a note of this because we should cache it at the initialize instruction" — on `pub struct RewardsIntegration { pub bump_seed: u8, ... }` [doublezerofoundation/doublezero-solana#113](https://github.com/doublezerofoundation/doublezero-solana/pull/113)
- "If I had to choose between the two, I would lean towards using the service key as a seed because this passport program could be used to onboard other sorts of folks in the future... we wouldn't have confusion about which key to use to derive this account." — on `pub struct AccessRequest { pub service_key: Pubkey, pub validator_id: Pubkey, ... }` [doublezerofoundation/doublezero-solana#16](https://github.com/doublezerofoundation/doublezero-solana/pull/16)

---

### testing & coverage

**Current guidance** (as of 2026-04-08): Cover error scenarios (program-log checks for failures can follow in a subsequent PR but should exist), and add missing negative/edge tests when a reviewer flags them. Match **exact** error variants in test assertions. Document non-obvious test setup with a brief comment (e.g. why balances need a `-10_000` lamport correction for signature fees, or why the distribution is initialized twice to uptick the DZ epoch).

**Test realism**: Reach the target state by running the **real instruction sequence**, not `set_*`/direct byte-poke helpers that mutate account bytes behind the program's back — a test that fabricates state can pass while the actual path to that state reverts. Also flag any test gated behind a feature flag that the default CI build never enables: it reports as coverage but never runs, which is worse than an absent test because it looks covered.

**Why it matters**: Wildcard error assertions can pass even when the program reverts for the wrong reason, masking real regressions.

**What to look for in a diff**:
- New instruction with no failure/negative-path test.
- Test assertions on `_` where a specific error variant is known.
- Magic lamport corrections or repeated setup steps without an explaining comment.
- Test setup that reaches target state via `set_*`/direct byte-poke helpers instead of running the real instruction sequence.
- A test behind a feature flag the default CI build never enables (false coverage).

**Examples**:
- "I had to do a double-take on the -10_000 lamports in the grant access test. It might be worth it to say that the original payer to create the access request account is also the transaction payer, which is why the balance looks like it doesn't reconcile without this 10k lamport correction (10k lamports comes from the cost for two signatures in the transaction)." [doublezerofoundation/doublezero-solana#16](https://github.com/doublezerofoundation/doublezero-solana/pull/16)
- "Very good call. Added here: .../commits/a72c6c9..." — adding a missing `WithdrawSolanaValidatorDeposit` setup path [doublezerofoundation/doublezero-solana#111](https://github.com/doublezerofoundation/doublezero-solana/pull/111)
- "We can add program log checking for error scenarios in a subsequent PR." [doublezerofoundation/doublezero-solana#20](https://github.com/doublezerofoundation/doublezero-solana/pull/20)

---

### CPI safety

**Current guidance** (as of 2026-04-22): Prefer `invoke_signed_unchecked` over `invoke`/`invoke_signed` so the redundant re-serialization/verification dependency can be dropped. Build CPI instructions with the shared `try_build_instruction` helper rather than hand-constructing `Instruction` structs. Fix account-list assembly (e.g. `From` impls that must include the SPL Token program ID) at the source rather than patching the program ID in at the call site. Native programs that cannot be CPI'ed into should not carry CPI-oriented logic.

**Why it matters**: Hand-built CPI instructions and call-site program-ID patches drift from the canonical account list, making it easy to pass the wrong account.

**What to look for in a diff**:
- `invoke`/`invoke_signed` where `invoke_signed_unchecked` would let a re-serialization dependency be dropped.
- Hand-constructed `Instruction` structs instead of `try_build_instruction`.
- A program ID (e.g. SPL Token) patched in at the call site instead of fixed in the account list's `From` impl.
- CPI-oriented logic on a native program that can't be CPI'ed into.

**Examples**:
- "Prefer `invoke_signed_unchecked` so we can remove this" — around `use solana_cpi::invoke_signed_unchecked; use solana_program::program::invoke;` [doublezerofoundation/doublezero-solana#16](https://github.com/doublezerofoundation/doublezero-solana/pull/16)
- "Can you use `try_build_instruction` instead?" — in `fn try_collect_integration_rewards(...)` [doublezerofoundation/doublezero-solana#116](https://github.com/doublezerofoundation/doublezero-solana/pull/116)
- "Instead of doing this, we should fix the `WithdrawIntegrationRewardsAccount`'s `From` impl to add the SPL Token program ID" — on `// - 7: SPL Token program (required so...` [doublezerofoundation/doublezero-solana#116](https://github.com/doublezerofoundation/doublezero-solana/pull/116)

---

### off-chain RPC / transaction hygiene *(reviewer-curated)*

**Current guidance**: The off-chain clients and tooling that build and send transactions have their own failure modes distinct from on-chain code:
- Never `unwrap()`/`expect()` on RPC-derived data (account fetches, blockhashes, simulation results) — a transient RPC hiccup then panics the process. In a `Result`-returning fn, `?`-propagate instead.
- Fetch the recent blockhash **inside** the send method, right before signing, not as a parameter passed down from a caller — a blockhash captured earlier goes stale and the send fails.
- Prefer a single `get_multiple_accounts` over N single `get_account` calls.
- Pre-flight existence checks (does the account / ATA already exist?) and add the create instruction **only** when it's actually needed, rather than unconditionally.
- Don't send zero-value or no-op transactions (e.g. a transfer of 0, a collect with nothing to collect) — check first and skip.
- Don't set a compute-unit *price* (priority fee) on uncontended transactions; and build the CU *limit* from real measured costs plus headroom, not a guessed constant.

**Why it matters**: These bugs don't fail a build or on-chain assertion — they surface as intermittent panics and stale-blockhash failures in production.

**What to look for in a diff**:
- `.unwrap()`/`.expect()` on the result of an RPC call inside a `Result`-returning fn.
- A blockhash fetched by a caller and threaded into the send method as an argument.
- A loop of single `get_account` calls where `get_multiple_accounts` fits.
- An unconditional create-account/create-ATA instruction with no prior existence check.
- A transaction built and sent without checking the value/effect is non-zero.
- A hardcoded CU price on a path with no contention, or a CU limit that isn't derived from measured costs + headroom.

---

### integer overflow / checked arithmetic

**Current guidance** (as of 2026-04-23): Use saturating/checked arithmetic for lamport and balance math and surface a real error via `ok_or_else` on the `None` case. But **don't add `checked_` operations that can never fail** — a defensive checked-sub is pushed back on when the invariant guarantees no underflow, so justify each checked op with a real overflow/underflow scenario. When indexing bitmaps, ask whether `bit`/`set_bit` should be `checked_` variants that validate the index against the type's bit width.

**Why it matters**: Unchecked lamport math can wrap silently, and an unvalidated bitmap index can write out of bounds.

**What to look for in a diff**:
- Raw `+`/`-` on lamports or balances instead of `saturating_`/`checked_` with an `ok_or_else` error.
- `checked_sub`/`checked_add` where the surrounding invariant makes `None` impossible (should justify or simplify).
- `bit`/`set_bit` calls where the index isn't validated against the type's bit width.

**Examples**:
- "\`\`\`suggestion\n    let lamports = additional_lamports.unwrap_or_default().saturating_add(rent);\n\`\`\`" — replacing `let lamports = rent + additional_lamports.unwrap_or(0);` [doublezerofoundation/doublezero-solana#16](https://github.com/doublezerofoundation/doublezero-solana/pull/16)
- "Seems impossible for this checked subtraction to be None, right?" — in `fn try_collect_integration_rewards(...)` [doublezerofoundation/doublezero-solana#116](https://github.com/doublezerofoundation/doublezero-solana/pull/116)
- "What is the API for bit and set_bit if the index exceeds the number of bits of the type? Should these be `checked_` operations instead, where you check the index value?" — on `pub fn set_integration_collected(&mut self, index: u16) { self.collected_integrations_bitmap.set_bit(index as usize, true); }` [doublezerofoundation/doublezero-solana#117](https://github.com/doublezerofoundation/doublezero-solana/pull/117)

---

### performance / compute units

**Current guidance** (as of 2026-04-16): Price compute-unit budgets precisely from known costs (e.g. ~1500 CU per bump-seed iteration for find-program-address) but pad with a few thousand CU of headroom because the added priority-fee cost is negligible and op prices can change. Eliminate avoidable syscalls (an extra `Rent::get`) and avoid redundant iterators/RPC shapes; prefer the smaller-scope Solana-native approach (iterate the two relevant epochs) over broader scans.

**What to look for in a diff**:
- A precise CU limit with no headroom for op-price changes.
- Avoidable syscalls (extra `Rent::get`) or duplicate iterators.
- Broad RPC scans (e.g. `ReverseSlotRange`) where a targeted per-epoch `getLeaderSchedule` fetch suffices.

**Examples**:
- "I think we can price out the CU very precisely. But I would leave some room for error or if the price of certain operations happen to change... Adding an extra few thousand CU doesn't cost that much more when setting priority fees" — around `compute_unit_limit += 2_500;` [doublezerofoundation/doublezero-solana#34](https://github.com/doublezerofoundation/doublezero-solana/pull/34)
- "I think just using getLeaderSchedule for a particular (slot of) epoch and validator ID for each epoch we care about would be simpler. Then we definitely don't need `ReverseSlotRange`" [doublezerofoundation/doublezero-solana#30](https://github.com/doublezerofoundation/doublezero-solana/pull/30)
- "Simplify by removing this (and avoid another rent syscall)." [doublezerofoundation/doublezero-solana#16](https://github.com/doublezerofoundation/doublezero-solana/pull/16)

---

### access control & authority

**Current guidance** (as of 2026-04-16): Separate roles from payers: let an admin/authority account be **read-only** and take a distinct writable payer account so the authority key never has to fund accounts. Route authority checks through the shared `VerifiedProgramAuthority` / upgrade-authority helpers. Make privileged limits (deposits, fees) configurable in the program config rather than hardcoding constants.

**Why it matters**: Coupling the authority signer to funding forces a privileged key to sign spends, and hardcoded limits require a redeploy to change.

**What to look for in a diff**:
- An admin/authority account marked writable and used as the funder — split out a separate payer.
- Bespoke authority checks instead of `VerifiedProgramAuthority` / upgrade-authority helpers.
- Hardcoded deposit/fee constants that belong in the program config.

**Examples**:
- "Can we actually have a separate payer account? Admin can be read-only and that allows the admin key to provide a separate payer account" — on `// - 1: Admin (also funds the new account).` [doublezerofoundation/doublezero-solana#113](https://github.com/doublezerofoundation/doublezero-solana/pull/113)
- "Maybe remove this and have this as a configurable SOL deposit in the program config" — on `pub const LAMPORTS_PER_SOL: u64 = 1_000_000_000;` [doublezerofoundation/doublezero-solana#16](https://github.com/doublezerofoundation/doublezero-solana/pull/16)

---

### error handling & custom errors

**Current guidance** (as of 2026-04-16): In tests, match the **exact** error variant a failing instruction produces rather than a wildcard: the system program reverts with `ProgramError::Custom(0)` for an already-created account, so assert on that instead of `_`. Emit a clear `msg!` and return a specific `ProgramError` (`InvalidInstructionData` / `InvalidAccountData`) on the failure branch.

**Why it matters**: A generic error with no `msg!` context leaves the offchain reader unable to tell why an instruction actually reverted.

**What to look for in a diff**:
- Test error assertions on `_` where the produced variant is known (e.g. `ProgramError::Custom(0)` for an already-created account).
- Failure branches returning a vague error with no `msg!` context.
- Wrong `ProgramError` variant for the failure kind (data vs. account).

**Examples**:
- "Can change `_` to `ProgramError::Custom(0)`" [doublezerofoundation/doublezero-solana#113](https://github.com/doublezerofoundation/doublezero-solana/pull/113)
- "All I meant was we can match exactly to that error variant because that's what the system program reverts with for an already-created account" [doublezerofoundation/doublezero-solana#113](https://github.com/doublezerofoundation/doublezero-solana/pull/113)

---

### events / logging

**Current guidance** (as of 2025-08-16): Program logs (`msg!`) are for the offchain reader's benefit — keep them meaningful but don't over-engineer wording. Renaming a log purely for cosmetics is unnecessary; add a code comment instead if clarity is the goal. Test error paths by reading program logs and simulating transactions before executing on a validator.

**What to look for in a diff**:
- Cosmetic-only `msg!` rewording where a code comment would serve better.
- New instruction whose error paths weren't validated via log inspection / transaction simulation.

**Examples**:
- "Oh I see. I figured it was clear what was happening with the log already. Let's just write a comment if you feel it's necessary to make things clearer with this program log" — on `msg!("Initialized user AccessRequest {}", service_key);` [doublezerofoundation/doublezero-solana#30](https://github.com/doublezerofoundation/doublezero-solana/pull/30)
- "Reading the program logs of the Solana program tests + simulating transactions before executing in a local validator to confirm" [doublezerofoundation/doublezero-solana#34](https://github.com/doublezerofoundation/doublezero-solana/pull/34)
