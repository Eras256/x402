# Scheme: `upto` on `Stellar`

## Versions supported

- ❌ `v1` is not supported.
- ✅ `v2`

## Supported Networks

[CAIP-2](https://namespaces.chainagnostic.org/stellar/caip2) identifiers:
- `stellar:pubnet`: Stellar mainnet
- `stellar:testnet`: Stellar testnet

> [!IMPORTANT]
> **Verification status (`contract` profile).** All three properties below
> are closed against real `stellar:testnet` behaviour, each backed by a
> settled transaction hash, not simulation alone. Full evidence:
> [`conformance/RESULTS.md`](https://github.com/Eras256/Periplo/blob/main/conformance/RESULTS.md)
> in the reference implementation, settled tx
> [`cc46374e...`](https://stellar.expert/explorer/testnet/tx/cc46374e34f70ff479ccf919d55df33d0bf1a05e1c7479fa8f90dac596c5d218).
>
> 1. `require_auth_for_args` accepts a root argument tuple of `(authorization,)`
>    while the [SEP-41] `transfer` rides as a sub-invocation for `max_amount`.
>    **Closed.** Confirmed via `inspectAuthEntry` on a real simulation: root
>    call `argCount=1`, one sub-invocation.
> 2. The pull-and-refund sequence fits within Soroban's per-transaction read,
>    write, instruction and memory limits. **Closed.** `2,026,530` instructions
>    of a `400,000,000` ceiling, `392` read bytes and `680` write bytes.
> 3. `temporary()` storage TTL can always cover `deadline_ledger − current_ledger`
>    under the `MAX_WINDOW_LEDGERS` bound. **Closed.** The settled nonce
>    entry's `liveUntilLedgerSeq`, read back from RPC, exceeded `deadline_ledger`.
>
> `extra.uptoProfile` and `extra.settlementContract` are proposed field names.
> Happy to align with whatever convention the maintainers prefer.

## Summary

`upto` on Stellar authorizes a transfer of **up to** a maximum amount, with the
settled amount fixed at settlement time. Like [`exact` on
Stellar](../exact/scheme_exact_stellar.md), the client signs [Soroban
authorization entries][auth-entry-signing] rather than a full transaction, and
the facilitator sponsors network fees.

Unlike `exact`, the settled amount is unknown when the client signs. Two
conformant profiles close that gap while preserving all four [core `upto`
properties](./scheme_upto.md#core-properties-must):

| Profile | Settlement path | Buyer accounts | Ships a contract | Facilitator binding |
| --- | --- | --- | --- | --- |
| **`contract`** | `UptoSettlement`, atomic pull-and-refund | G- and C-accounts | Yes | Bound to one named facilitator |
| **`stateless`** | `UptoSettlement` (stateless), atomic `approve` + `transfer_from` | G- and C-accounts | Yes, a different stateless contract | Facilitator-agnostic |

Both profiles ship a contract and are equally conformant. They differ in
settlement mechanism, not in whether Soroban's `require_auth_for_args` trick
is used, and neither is a universal default: § Choosing between `contract`
and `stateless` (under Profile `stateless`, below) states the deployment
trade-off and the topology each one fits.

> [!NOTE]
> [SEP-41] Soroban tokens only; classic assets are out of scope. Amounts are
> `i128` in the token's own precision. USDC on Stellar uses **7 decimals**.

## Why a SEP-41 allowance is insufficient

`approve` / `transfer_from` measured against the four core properties:

| Core property | Allowance behaviour | Conformant |
| --- | --- | --- |
| 4. Maximum amount | The allowance is a ceiling. | ✅ |
| 2. Time-bound | `expiration_ledger` bounds validity. | ✅ (no `validAfter`) |
| 3. **Recipient binding** | `transfer_from` lets the spender choose **any** `to`. Nothing the client signed constrains the destination. | ❌ |
| 1. **Single-use** | An allowance is a standing balance, drawable across many calls until exhausted or expired. | ❌ |

Recipient binding is decisive: it would let a compromised facilitator redirect
funds, which is the risk Core Property 3 exists to eliminate. Enforcing both
properties requires code that runs at settlement. That is what the `contract`
profile provides.

> A facilitator implementing `upto` on Stellar via `approve` / `transfer_from`
> does not satisfy Core Properties 1 or 3 and MUST NOT advertise `upto` support.

## Profile `contract`: protocol flow

1. Client requests a resource; server responds `402` with `amount` set to the
   authorized **maximum**, plus `extra.areFeesSponsored`, `extra.uptoProfile`
   and `extra.settlementContract`.
2. Client builds an `Authorization` (below) and an invocation of
   `UptoSettlement.settle`, then simulates to identify required auth entries.
3. Client signs the auth entries: the root invocation restricted to
   `(authorization,)`, plus the [SEP-41] `transfer` sub-invocation for
   `max_amount`. Expiration is `currentLedger + ceil(maxTimeoutSeconds /
   estimatedLedgerSeconds)`. Use the current network estimate for
   `estimatedLedgerSeconds` where available; fall back to `5`.
4. Client retries with the base64 XDR in `payload.transaction`.
5. Resource server calls `/verify`. At this phase `requirements.amount` carries
   the **maximum**.
6. Resource server serves the request, meters consumption, and calls `/settle`
   with `requirements.amount` set to the **actual charge**.
7. Facilitator re-verifies independently, rebuilds the transaction with its own
   account as source, and passes `actual_amount` as the unsigned second argument
   to `settle`.
8. Facilitator simulates, derives fee and fresh Soroban resource data, signs,
   submits, and polls for confirmation.

`/settle` MUST perform full verification independently and MUST NOT assume prior
verification.

## The `Authorization` struct

```rust
#[contracttype]
#[derive(Clone)]
pub struct Authorization {
    pub from: Address,           // buyer
    pub to: Address,             // MUST equal requirements.payTo (Property 3)
    pub asset: Address,          // SEP-41 token
    pub max_amount: i128,        // ceiling (Property 4)
    pub valid_after_ledger: u32, // Property 2
    pub deadline_ledger: u32,    // Property 2
    pub nonce: BytesN<32>,       // Property 1
    pub facilitator: Address,    // binds the settling party
}
```

Every field is covered by the client's signature. `facilitator` binds settlement
to one operator, mirroring `witness.facilitator` in the [EVM
profile](./scheme_upto_evm.md). It prevents an intercepted payload being settled
elsewhere.

**Ledger sequences, not timestamps.** Stellar auth entries expire by
`signatureExpirationLedger`. `valid_after_ledger` and `deadline_ledger` are
ledger sequences derived client-side from `maxTimeoutSeconds`. At the default
`60` and ~5-second ledgers, an authorization is valid for roughly 12 ledgers.
Implementations MUST NOT convert timestamps to ledger sequences by assuming a
fixed interval over long horizons.

## Both G- and C-account payers

`Authorization.from` and `Authorization.facilitator` MAY be either a
G-account (`G...`) or a C-account (`C...`) Stellar address.
`require_auth_for_args`/`require_auth` dispatch transparently to either the
protocol-defined Ed25519 account signature or the C-account's own
`__check_auth`. The contract does not special-case either kind; nothing in
`settle()` assumes a G-account payer. (See also
[#3134](https://github.com/x402-foundation/x402/pull/3134)'s more extensive
treatment of G-/C-account signing adapters, which informed this section.)

Facilitators MUST treat the credential's `signature` field as opaque. For a
G-account it has the protocol-defined account-signature shape. For a
C-account its `ScVal` type and contents are defined entirely by that
account's `__check_auth`. Facilitators MUST NOT require an Ed25519 shape for
a C-account credential or attempt to reproduce wallet-specific verification
off-chain. Enforcing-mode simulation is the authoritative check (§
Facilitator verification rules, rule 5).

A C-account cannot be a transaction source. When building the candidate
invocation for simulation, clients MUST use a separate, funded G-account as
the simulation source when `from` is a C-account. Clients MAY do the same for
a G-account payer, for consistency. This source only produces a valid
simulation. It is never included in the signed authorization tree and need
not be the eventual facilitator.

A C-account's spending policy sees `max_amount` at signing time, not the
later `actual_amount`. This is true under every `upto` profile on Soroban,
since the settled amount is unknown by construction when the client signs.
Clients MUST surface a wallet's policy rejection rather than weakening or
rewriting the signed authorization tree.

## The `UptoSettlement` contract

```rust
pub fn settle(env: Env, authorization: Authorization, actual_amount: i128)
```

`actual_amount` is supplied by the facilitator and is **deliberately excluded**
from what the client signs. The contract enforces:

```rust
// 1. Authorize the client for the authorization only (excludes actual_amount).
authorization.from.require_auth_for_args((authorization.clone(),).into_val(&env));
authorization.facilitator.require_auth();

// 2. Time bounds.
let ledger = env.ledger().sequence();
if ledger < authorization.valid_after_ledger { panic_with_error!(&env, Error::NotYetValid); }
if ledger > authorization.deadline_ledger   { panic_with_error!(&env, Error::Expired); }

// 4. Ceiling.
if actual_amount < 0 || actual_amount > authorization.max_amount {
    panic_with_error!(&env, Error::AmountExceedsMaximum);
}

// 1. Single use.
let key = DataKey::Nonce(authorization.nonce.clone());
if env.storage().temporary().has(&key) { panic_with_error!(&env, Error::AuthorizationConsumed); }
env.storage().temporary().set(&key, &authorization.deadline_ledger);
env.storage().temporary().extend_ttl(&key, ttl, ttl);

// 3. Recipient binding: `to` comes from the signed struct.
let token = token::Client::new(&env, &authorization.asset);
token.transfer(&authorization.from, &env.current_contract_address(), &authorization.max_amount);
if actual_amount > 0 { token.transfer(&env.current_contract_address(), &authorization.to, &actual_amount); }
let refund = authorization.max_amount - actual_amount;
if refund > 0 { token.transfer(&env.current_contract_address(), &authorization.from, &refund); }
```

**`require_auth_for_args` is what makes `upto` expressible on Soroban.** A plain
`require_auth()` authorizes the invocation with its full argument list,
including `actual_amount`. That forces the client to know the charge at
signing time, collapsing `upto` into `exact`. Restricting the authorized
tuple to `(authorization,)` decouples the ceiling from the charge.

**Why pull-and-refund.** The client's signed sub-invocation is
`transfer(from, UptoSettlement, max_amount)`. Auth entries commit to **exact**
sub-invocation arguments, so the contract cannot instead call
`transfer(from, to, actual_amount)`. The mismatch fails authorization. All legs
execute in one transaction: there is **no custody window**. Costs, stated plainly:

- Up to **three** token transfers per settlement. The refund leg is skipped when
  `actual_amount == max_amount`, the payout leg when `actual_amount == 0`.
  Implementations MUST verify the flow fits Stellar's [per-transaction
  limits](https://lab.stellar.org/network-limits).
- The client must hold **`max_amount`**, not `actual_amount`, at settlement.
  Resource servers SHOULD keep ceilings tight.
- The buyer needs a **trustline** to the asset first.

**Nonce storage and TTL.** Soroban entries can be evicted, which appears to risk
replay. It does not, because **the deadline dominates the nonce**: the expiry
check runs before the nonce check, so an entry only needs to survive until
`deadline_ledger`. Implementations MUST size the TTL to cover
`deadline_ledger − currentLedger` and MUST reject windows exceeding the
contract's maximum supported TTL. This is why `temporary()` storage is correct
and cheaper than `persistent()`.

## `PaymentRequirements`

```json
{
  "scheme": "upto",
  "network": "stellar:testnet",
  "amount": "50000000",
  "asset": "CBIELTK6YBZJU5UP2WWQEUCYKLPU6AUNZ2BQ4WWFEIE3USCIHMXQDAMA",
  "payTo": "GBHEGW3KWOY2OFH767EDALFGCUTBOEVBDQMCKU4APMDLQNBW5QV3W3KO",
  "maxTimeoutSeconds": 60,
  "extra": {
    "areFeesSponsored": true,
    "uptoProfile": "contract",
    "settlementContract": "CDLZFC3SYJYDZT7K67VZ75HPJVIEUVNIXF47ZG2FB2RMQQVU2HHGCYSC"
  }
}
```

- `amount` is **phase-dependent** per [Core Property
  5](./scheme_upto.md#5-phase-dependent-amount-semantics-in-paymentrequirements):
  the authorized **maximum** at `/verify`, the **actual charge** at `/settle`.
- `extra.areFeesSponsored`: currently always `true`, matching `exact`.
- `extra.uptoProfile`: `"contract"` or `"stateless"`.
- `extra.settlementContract`: REQUIRED for both profiles. It is the canonical
  `UptoSettlement` deployment address for the selected `uptoProfile` on this
  network. `contract` and `stateless` are different contracts; implementations
  supporting both MUST deploy and configure both addresses.

## `PaymentPayload` `payload` field

```json
{
  "transaction": "AAAAAgAAAABriIN4poutFUmHfB6FbFJu8GgXoPPTGQWREqFpPfvO1AAA...",
  "authorization": {
    "from": "GBHE...", "to": "GBHE...", "asset": "CBIE...",
    "maxAmount": "50000000",
    "validAfterLedger": 0, "deadlineLedger": 58291204,
    "nonce": "9f2c1a...",
    "facilitator": "GA7QYNF7SOWQ3GLR2BGMZEHXAVIRZA4KVWLTJJFC7MGXUA74P7UJVSGZ"
  }
}
```

`transaction` is the base64 XDR of a transaction with a single
`invokeHostFunction` calling `UptoSettlement.settle`, carrying signed auth
entries.

`authorization` is **advisory only**. Facilitators MUST derive every enforced
value from the XDR auth entries, never from this field, and MUST reject a
mismatch between the two.

## Facilitator verification rules (MUST)

**1. Protocol.** `x402Version` is `2`. Both `payload.accepted.scheme` and
`requirements.scheme` are `"upto"`. Networks match. `extra.uptoProfile` is
supported and matches between payload and requirements.

**2. Transaction structure.** Exactly one `invokeHostFunction` operation, type
`hostFunctionTypeInvokeContract`, contract equal to `extra.settlementContract`,
function `"settle"`. The decoded `Authorization` MUST satisfy: `to` equals
`requirements.payTo`; `asset` equals `requirements.asset`; `facilitator` equals
the facilitator's own address; `max_amount` equals the **verify-phase**
`requirements.amount`; `valid_after_ledger <= currentLedger < deadline_ledger`;
and `deadline_ledger - currentLedger <= ceil(maxTimeoutSeconds /
estimatedLedgerSeconds)`.

**3. Authorization entries.** Signed entries for `authorization.from`.
Credential type MUST be address-based authorization (`sorobanCredentialsAddress`,
or a newer address-credential variant on networks where it is active), never
source-account implicit authorization. The credential address MUST be either
`Address::Account` (G-account) or `Address::Contract` (C-account) and MUST
equal `authorization.from`; its `signature` field MUST be treated as opaque
(§ Both G- and C-account payers). The authorized root argument tuple MUST be
exactly `(authorization,)`. An entry covering `actual_amount` indicates a client
that has fixed the charge and MUST be rejected. Exactly one `subInvocation`:
`transfer(from, settlementContract, max_amount)` on `requirements.asset`. All
required signers present. Entry expiration within the `maxTimeoutSeconds` bound.

**4. 🚨 Facilitator safety.** The client-supplied transaction source and
operation source MUST NOT be the facilitator. The facilitator MUST NOT be
`authorization.from`, and MUST NOT appear as a signer in any client-supplied auth
entry. Simulation MUST emit balance changes consistent **only** with the payer
decrease, recipient increase, and the transient contract legs. **No others.**
These mirror [`exact` §4](../exact/scheme_exact_stellar.md#4--facilitator-safety)
and are equally load-bearing.

**5. Simulation.** Re-simulate against current ledger state at both `/verify` and
`/settle`; simulation MUST succeed; at `/settle`, events MUST confirm a net
transfer to `authorization.to` of exactly the settle-phase `requirements.amount`.

> [!WARNING]
> Soroban's simulator **records** `require_auth()` without verifying signatures.
> Simulation is not authorization verification. A payload with absent or invalid
> signatures can simulate successfully. Verify signatures explicitly.

## Settle-time verification

At `/settle`, `requirements.amount` carries the **actual** amount, which may be
below the signed ceiling. Facilitators MUST:

1. **Verify the client's signature against `authorization.max_amount`**, not
   `requirements.amount`. The client signed the ceiling.
2. Validate `0 <= requirements.amount <= authorization.max_amount`.
3. Invoke `settle` with `actual_amount = requirements.amount`.
4. Re-check `deadline_ledger`. Metering happens between verify and settle.

> A facilitator enforcing `max_amount === requirements.amount` at settle time
> rejects all partial settlements and breaks the scheme. The equality check in
> rule 2 applies **only** to `/verify`.

## Fees, throughput and settlement response

Fee handling is identical to [`exact` on
Stellar](../exact/scheme_exact_stellar.md#transaction-fees): derive from a fresh
settle-time simulation (`simulationResourceFee + inclusionBuffer`, buffer ≥ 100
stroops), refresh Soroban resource data, never reuse the client's bid. Because
`upto` runs up to three transfers, implementations SHOULD set a higher default
`maxTransactionFeeStroops` than `exact` and reject with
`invalid_upto_stellar_payload_fee_exceeds_maximum` when exceeded.

Facilitators SHOULD use **channel accounts**. The facilitator is the transaction
source, so its sequence number is the bottleneck under bursty agent traffic.

`SettlementResponse` follows the `upto` extension defined in
[`scheme_upto_evm.md` §3](./scheme_upto_evm.md#3-settlementresponse-schema-extension):
the base schema plus `amount`, the **actual** amount charged in atomic units
(may be `0`).

```json
{ "success": true, "transaction": "a1b2...", "network": "stellar:testnet",
  "payer": "GBHE...", "amount": "1858000" }
```

## Error codes

`upto` on Stellar uses the standard x402 error codes defined in the
[x402 specification](../../x402-specification-v2.md#9-error-handling), plus two
that carry over from its scheme and its network:

- **`invalid_upto_stellar_payload_settlement_exceeds_amount`**: attempted to
  settle for more than the authorized maximum. Mirrors
  `invalid_upto_evm_payload_settlement_exceeds_amount`.
- **`invalid_upto_stellar_payload_fee_exceeds_maximum`**: the
  simulation-derived settlement fee exceeds `maxTransactionFeeStroops`. Mirrors
  `invalid_exact_stellar_payload_fee_exceeds_maximum`.

Every rejection MUST carry a non-null `reason`.

## Profile `stateless`

An alternative to `contract`, contributed by
[Iam0TI](https://github.com/Iam0TI) via
[x402-foundation/x402#3134](https://github.com/x402-foundation/x402/pull/3134),
with a reference implementation at
[`0d1026/Rialto`](https://github.com/0d1026/Rialto/tree/mvp/contracts/upto-settlement).
Full protocol flow, wire formats, and facilitator verification rules: see that
PR's `specs/schemes/upto/scheme_upto_stellar.md`. This section summarizes the
mechanism and states its trade-off against `contract` plainly. It does not
restate every MUST from the source PR, which remains the authoritative
reference for implementing this profile.

**Mechanism.** `UptoSettlement` under this profile is *stateless*. It holds no
funds at any point and stores no per-request state on-chain. The client signs
one authorization binding `(payTo, asset, maxAmount, validAfter, deadline,
expirationLedger, salt, autoRevoke)`. That covers everything except the
eventual charge. A `token.approve(from, UptoSettlement, maxAmount,
expirationLedger)` sub-invocation sits nested beneath it. If
`autoRevoke = true`, a second `approve(from, UptoSettlement, 0, 0)`
sub-invocation atomically clears any unused remainder. At settlement, the
contract grants itself that allowance, satisfied by the pre-signed
sub-invocation, then calls `transfer_from(self, from, payTo, amount)`
directly: one real token movement, buyer to recipient, with the contract
acting purely as authorized spender rather than an intermediate holder.

```rust
pub fn settle(
    env: Env, from: Address, pay_to: Address, asset: Address,
    max_amount: i128, valid_after: u64, deadline: u64,
    expiration_ledger: u32, salt: BytesN<32>, auto_revoke: bool,
    amount: i128,
) -> i128
```

**Replay protection is Soroban's own.** No app-level nonce or TTL management.
Single use comes entirely from the protocol-level nonce every signed
authorization entry already carries, consumed on first use and scoped to the
entry's own `signatureExpirationLedger`. This removes an implementation-bug
class that `contract`'s `temporary()`-storage nonce has to actively defend
against: sizing `extend_ttl` to cover `deadline_ledger − currentLedger` (§
Nonce storage and TTL, above). `stateless` carries no such risk, because
there is no author-managed TTL to size in the first place.

**The trade-off, stated plainly.** `stateless` settles cheaper: real settled
transactions from both profiles' own testnet deployments, each independently
checked against Horizon. `fee_charged`, and the decoded on-chain `settle`
call arguments, confirm `stateless`'s two transactions are a genuine partial
settlement (`300,000` of a `1,000,000` signed ceiling) and a genuine maximum
settlement (`500,000` of `500,000`), not just labels taken on trust:

| | `contract` (partial) | `stateless` (partial) | `stateless` (maximum) |
| --- | --- | --- | --- |
| `fee_charged` | 42,872 stroops | 32,731 stroops | 30,585 stroops |
| Transaction | [`cc46374e...`](https://stellar.expert/explorer/testnet/tx/cc46374e34f70ff479ccf919d55df33d0bf1a05e1c7479fa8f90dac596c5d218) | [`2ea539f6...`](https://stellar.expert/explorer/testnet/tx/2ea539f67aa8daff06698a93279345c88c71dd41a5a77f77d65335f4113a11d5) | [`b76b45a3...`](https://stellar.expert/explorer/testnet/tx/b76b45a383f7e54991e3fad6beb0143b7b78c2bff25e929305fe911241869763) |

It also never holds a balance, even transiently within a transaction. In
exchange, settlement is **facilitator-agnostic**: no facilitator identity is
bound into the signed tree, so any party holding the signed authorization
entries, not only the resource server's intended facilitator, can submit
settlement, for up to the full signed `maxAmount`, to the signed `payTo`.
`payTo` and `maxAmount` are still fully bound. This is not a redirection or
overcharge risk. It is a risk that a leaked or multiply-forwarded
authorization gets settled by an unintended party, for more than the actual
metered usage, before the intended facilitator acts. Deployments that need
settlement restricted to one operator MUST enforce that off-chain (e.g. the
resource server only ever forwards payloads to one trusted facilitator
endpoint).

**A behavior worth documenting precisely.** SEP-41 `approve` replaces rather
than adds to an existing allowance. If a buyer uses `autoRevoke = false` and
leaves a remainder allowance from one authorization (permitted by design: see
the source PR's § `PaymentPayload` `payload` field), a **subsequent**,
unrelated `stateless` authorization settling against the same `(from, asset,
UptoSettlement)` triple will silently overwrite that remainder via its own
fresh `approve()` call. The practical exposure is small. The remainder was
already unreachable except through another `settle()` call requiring its own
fresh signature. But implementations SHOULD document this rather than let
"the remainder survives until `expirationLedger`" read as a more durable
guarantee than it is.

**Choosing between `contract` and `stateless`.** Neither profile is a
universal default. The right choice depends on deployment topology.
`contract` fits a **federated or multi-facilitator** deployment, where more
than one facilitator may plausibly see the same signed payload
(catalog-driven discovery routing, a resource server trying more than one
facilitator, a facilitator that receives a payload for `/verify` but never
proceeds to `/settle`), and containing a leaked-authorization risk matters
more than settlement cost. `stateless` fits a **single-operator** deployment,
one resource server, one trusted facilitator, a controlled channel between
them, where that risk is already minimized operationally and the lower cost
and simpler implementation are worth more. Implementers choose per
deployment. This spec does not declare one profile conformant and the other
not.

**A third design, named here but not specified.** Both profiles above ship
a shared `UptoSettlement` contract that every buyer's authorization is
checked against. A pure self-enforcement design is possible instead: a
buyer's own C-account `__check_auth` enforces all four core properties
itself (recipient allowlist, cap, ledger window, consumed-nonce set), and the
client authorizes a direct SEP-41 `transfer(from, to, actual_amount)` with no
intermediary contract at all. That is a genuinely different value
proposition from either profile above: zero shared-contract trust surface,
since there is nothing between the buyer's own account and the token
transfer for either `contract` or `stateless` to be a dependency of. It is
also the narrowest. It only ever works for a buyer running a
policy-enforcing custom account, with no path for a plain G-account payer,
unlike either documented profile. It is not written up as a third profile
here because, unlike `contract` and `stateless`, nobody has yet built and
settled a real transaction against it. This paragraph records the design
space, not a specified and evidenced option.

## Security considerations

**Facilitator cannot redirect funds.** `to` is inside the signed struct and read
from the auth entry, never from the advisory payload field.

**Facilitator cannot overcharge.** `actual_amount > max_amount` panics. Even a
fully compromised facilitator is bounded by the signed ceiling.

**Facilitator can undercharge or not settle.** Inherent to `upto` on every
network: the resource server chooses the charge. `upto` bounds the buyer's
downside; it does not guarantee the seller's revenue.

**Unsettled authorizations lock nothing.** No funds move at signing time. The
exposure is the buyer keeping `max_amount` liquid until `deadline_ledger`,
which is why short `maxTimeoutSeconds` values are preferable.

**Contract balance is transient by design.** Any implementation allowing a
balance to persist across transactions has introduced custody. Implementations
SHOULD assert a zero contract balance at the end of `settle`.

**Nonce generation.** Clients MUST use a cryptographically random 32-byte
`nonce`. A predictable nonce lets an observer pre-consume it, denying service.

## Out of scope

Per [`upto` core](./scheme_upto.md#out-of-scope): multi-settlement, streaming,
recurring payments and open-ended allowances. On Stellar, `batch-settlement`
additionally requires a Soroban escrow, a voucher store, double-spend prevention
and its own audit; it is deliberately deferred.

## Appendix

`upto` reuses `exact`'s transport, fee sponsorship, auth-entry signing model and
facilitator safety rules verbatim. The differences are confined to the settlement
target (contract vs. token), the authorized argument tuple
(`require_auth_for_args` vs. `require_auth`), phase-dependent `amount` semantics,
and the `amount` field in the settlement response. Implementations SHOULD share
verification code between the two schemes where these do not diverge.

As in `exact`, clients authorize via [auth-entry signing][auth-entry-signing]
rather than full transaction signing: no client [sequence number] is spent, both
C- and G-accounts are supported, and fee sponsorship is required. Full
transaction signing is not supported for `upto`.

[SEP-41]: https://stellar.org/protocol/sep-41
[auth-entry-signing]: https://developers.stellar.org/docs/build/guides/freighter/sign-auth-entries
[sequence number]: https://developers.stellar.org/docs/learn/glossary#sequence-number
