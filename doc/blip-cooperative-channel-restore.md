```
bLIP: XXX
Title: Cooperative Channel Restore
Status: Draft
Author: Abhijit Das <Sukuna0007Abhi>
Created: 2025-05-31
License: CC0
```

## Abstract

A protocol extension that lets a node recover lost channel state from its
counterparty instead of force-closing. Both sides negotiate a feature bit; the
recovering node proves identity with a signed nonce; the healthy peer hands
back the latest signed commitment. The channel survives.

## Copyright

This bLIP is licensed under the CC0 license.

## Motivation

SCB recovery works, but the outcome is always the same: the channel dies.
On-chain fees get burned, CSV delays lock up funds, and both sides have to open
a new channel if they want to keep transacting.

The frustrating part is that the peer *already has everything* needed to restore
the channel — the latest signed commitment transaction and the relevant
revocation secrets. There's no fundamental reason to destroy the channel when
the peer could just send that state back.

This bLIP adds that path. When both sides opt in, the recovering node gets its
latest commitment handed back, verifies it against the known funding keys, and
resumes normal operation after a single reconnection. No on-chain footprint.

The key insight that makes this safe: the peer can already unilaterally close
with their latest commitment and take everything owed to them. Handing us our
own signed commitment gives us strictly less power than they already have. The
recovering side verifies the 2-of-2 funding signature, so fabricated state
can't get through.

## Specification

### Feature Bit

| Bits    | Name                                 | Context |
|---------|--------------------------------------|---------|
| 272/273 | `option_cooperative_channel_restore` | IN      |

Set in `init` and `node_announcement`. Both sides must have negotiated this
for the protocol to activate. Otherwise, standard data-loss / SCB behavior
applies — nothing changes.

### TLV Extension on `channel_reestablish`

1. `tlv_stream`: `channel_reestablish_tlvs`
    1. type: 7 (`cooperative_restore`)
    2. data:
        * [`32*byte`:`nonce`]
        * [`signature`:`node_signature`]

`nonce` is 32 bytes of fresh randomness, generated per reconnection attempt.

`node_signature` signs the following message with the node's private key:

```
SHA256d("Lightning Signed Message:" || "cooperative_restore:" || channel_id || nonce)
```

This follows the same `hsmd_sign_message` convention used elsewhere in CLN.
The signature serves two purposes: it proves identity (only the real node
owner can produce it) and it prevents replay (the nonce is random each time).

### New Message: `cooperative_restore_response`

1. type: 41042 (`cooperative_restore_response`)
2. data:
    * [`channel_id`:`channel_id`]
    * [`u64`:`latest_commitment_number`]
    * [`32*byte`:`per_commitment_secret`]
    * [`u16`:`signed_commitment_tx_len`]
    * [`signed_commitment_tx_len*byte`:`signed_commitment_tx`]

`latest_commitment_number` is the commitment being restored — that's
`next_commitment_number - 1` from the healthy peer's point of view.

`per_commitment_secret` is the secret for commitment N-1. The recovering node
needs this for the shachain and to verify the peer isn't lying about the
commitment number.

`signed_commitment_tx` is the fully serialized transaction with the 2-of-2
P2WSH witness already applied (both funding key signatures in the witness
stack). The recovering node does not need to re-sign anything.

### Protocol Flow

```
  Recovering Node                           Healthy Peer
       |                                         |
       |  channel_reestablish                     |
       |  + cooperative_restore {nonce, sig}       |
       |---------------------------------------->|
       |                                         |
       |                      Detects peer behind |
       |                      Verifies nonce sig  |
       |                      Builds commitment   |
       |                      HSM-signs it        |
       |                      Applies 2-of-2      |
       |                                         |
       |  cooperative_restore_response            |
       |  {commit_num, secret, signed_tx}         |
       |<----------------------------------------|
       |                                         |
       |  Verifies everything (see below)         |
       |  Reports to channel manager              |
       |  Disconnects cleanly — no error          |
       |                                         |
       |  Reconnects with restored state          |
       |---------------------------------------->|
       |         Normal operation resumes         |
```

### Recovering Node

When a node detects it is behind (peer's `next_revocation_number` exceeds its
own expectation) and both sides negotiated `option_cooperative_channel_restore`:

- MUST include the `cooperative_restore` TLV with a fresh random nonce and
  valid node signature.
- MUST wait for `cooperative_restore_response` before falling back to SCB.
- If any unexpected message arrives instead, MUST fall back to SCB.

Upon receiving `cooperative_restore_response`:

- MUST verify `channel_id` matches.
- MUST verify `per_commitment_secret` produces a valid public key.
- MUST deserialize `signed_commitment_tx` and verify:
  - Input 0 spends the correct funding outpoint.
  - The obscured commitment number (extracted from locktime + sequence)
    matches `latest_commitment_number` after XOR with the channel's
    commitment number obscurer.
  - Both signatures in the 2-of-2 witness validate against the known
    LOCAL and REMOTE funding pubkeys.
- MUST report the restored state (commitment number, per-commitment secret,
  remote signature, signed tx) to its channel manager for persistence.
- MUST disconnect *without* sending an `error`, so the peer doesn't
  interpret it as a force-close request. The channel manager will
  automatically reconnect with the restored state.

If any step fails — bad signature, wrong outpoint, malformed tx, whatever —
MUST fall back to standard SCB force-close. The cooperative path is strictly
best-effort.

### Healthy Peer

When a node detects its peer is behind and the `channel_reestablish` includes
a `cooperative_restore` TLV:

- MUST verify `node_signature` against the peer's `node_id`. If it fails,
  ignore the restore request entirely and proceed with normal data-loss
  handling (send a warning).
- If valid:
  - SHOULD build the latest commitment transaction for the recovering peer
    using `channel_txs()` (or equivalent).
  - SHOULD verify its own stored remote commitment signature against the
    constructed transaction before proceeding.
  - MUST sign the transaction using its own funding key via HSM.
  - MUST construct the 2-of-2 multisig witness (peer's stored sig + own sig,
    ordered by funding pubkey) and apply it to the transaction.
  - MUST derive the per-commitment secret for N-1 via `revoke_commitment()`.
  - MUST send `cooperative_restore_response`.
- After sending, SHOULD issue a `warning` (not `error`) and keep the
  connection open. The recovering node will disconnect on its own once it has
  processed the response.

## Backward Compatibility

The TLV type is odd (7), so non-supporting nodes ignore it per BOLT #1.
Message type 41042 is in the experimental range (>32768), so it's treated as
an unknown message by nodes that don't recognize it. No behavior change for
any existing node or channel type.

A node that supports this feature paired with a node that doesn't will simply
never enter the cooperative restore path — the feature check gates everything,
and the fallback is the exact same SCB behavior that exists today.

## Security Considerations

### Can the peer send us a fake commitment?

No. We verify both 2-of-2 witness signatures against the funding pubkeys
that were established at channel open. The peer would need to forge a
signature for *our* funding key to fabricate a state, which requires
breaking ECDSA on secp256k1.

The funding outpoint check is equally important — even a valid-looking
transaction that doesn't spend the actual funding output is caught.

### Can someone replay an old restore request?

No. The nonce is 32 bytes of fresh randomness, and the signature covers
`channel_id || nonce`. A captured `channel_reestablish` from a previous
session has a different nonce and cannot be reused.

### Can the peer give us an old (but real) commitment?

They could in theory send commitment N-5 instead of N, but the recovering
node verifies the obscured commitment number embedded in the transaction's
locktime and sequence fields. If it doesn't match what the peer claims in
`latest_commitment_number`, verification fails and we fall back to SCB.

There's a subtler point here: the recovering node can't independently verify
that the commitment number is the *latest* one (since it lost state). But
the peer has no incentive to provide an older state — they'd be giving us a
commitment they've already revoked, which we could theoretically use to
penalize them. Rational peers always send the latest.

### DoS via repeated restore requests

A malicious peer could repeatedly connect with stale state to force the
responding side into expensive HSM signing operations. Implementations
SHOULD rate-limit restore attempts per channel.

## Reference Implementation

CLN (Core Lightning): [feature/cooperative-channel-restore](https://github.com/nickygenesis/NEWHH/tree/feature/cooperative-channel-restore)
