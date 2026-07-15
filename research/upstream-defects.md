# Upstream defects found while writing the Sync Guide

Verified firsthand 2026-07-15. Ready-to-file issue bodies; a fourth defect
(`RawTransaction.height` mempool contradiction) is already tracked as
[zcash/librustzcash#1484](https://github.com/zcash/librustzcash/issues/1484).

---

## 1. zcash/zips — ZIP-307 grace-period condition inverted

**Title:** ZIP 307: second Canopy-onward lead-byte condition rejects 0x01
during the grace period

**Body:**

The compact-note trial-decryption pseudocode in `zip-0307.rst` contains two
`[Canopy onward]` lead-byte checks:

```
- [Canopy onward] if height <  CanopyActivationHeight + ZIP212GracePeriod
                  and leadByte ∉ {0x01, 0x02}, return ⊥
- [Canopy onward] if height <  CanopyActivationHeight + ZIP212GracePeriod
                  and leadByte ≠ 0x02, return ⊥
```

Both test `height < CanopyActivationHeight + ZIP212GracePeriod`. As written,
the second line rejects `leadByte = 0x01` *during* the grace period,
contradicting the first line and defeating the grace period entirely (no
pre-ZIP-212 note would ever decrypt). The second condition is evidently
intended to be the post-grace rule:

```
if height ≥ CanopyActivationHeight + ZIP212GracePeriod and leadByte ≠ 0x02,
return ⊥
```

which matches the deployed behaviour (`zip212_enforcement` in
librustzcash: `GracePeriod` accepts both lead bytes, `On` requires `0x02`).

Observed at zcash/zips `69610984`.

---

## 2. zcash/librustzcash — `ScanSummary` has no Ironwood counters

**Title:** `ScanSummary` lacks spent/received counters for the Ironwood pool

**Body:**

Struct `ScanSummary` (`zcash_client_backend/src/data_api/chain.rs:440`)
carries `spent_sapling_note_count` / `received_sapling_note_count` and, under
`feature = "orchard"`, the Orchard pair — but no Ironwood counterparts. With
NU6.3 introducing the Ironwood pool (ZIP 258) and wallets scanning Ironwood
outputs via the extended compact-block format, per-pool scan telemetry for
Ironwood appears to be an oversight; callers consuming `ScanSummary` (e.g.
progress reporting) will silently under-count activity in the new pool.

Observed at zcash/librustzcash `e30517e433`.

---

## 3. zingolabs/zaino — advertises nonexistent lightwallet protocol v0.5.0

**Title:** `LightdInfo.lightwalletProtocolVersion` hard-codes "v0.5.0",
which does not exist upstream

**Body:**

Both backends hard-code the advertised protocol version:

- `packages/zaino-state/src/backends/fetch.rs:2097`
- `packages/zaino-state/src/backends/state.rs:2776`

```rust
lightwallet_protocol_version: "v0.5.0".to_string(),
```

The canonical `zcash/lightwallet-protocol` repository's latest tagged
release is `v0.4.1`; no `v0.5.0` tag exists (and lightwalletd leaves field
18 unpopulated). Zaino's vendored proto snapshot is an intermediate state
(Ironwood fields present) that matches no published protocol version, so
clients keying behaviour on this field would be misled. Either the canonical
repo should mint the version Zaino ships, or Zaino should advertise the
version it actually implements.

Observed at zingolabs/zaino `0e057e22`.

---

# Upstream defects found while writing the ZSA Guide

Verified firsthand 2026-07-15 at zcash/zips `69610984`, QED-it/zips (zsa1)
`fd71419a`, QED-it/orchard (zsa1) `cf801a5d`, QED-it/librustzcash (zsa1)
`3f9b53fc`.

---

## 4. zcash/zips — ZIP-227 cites a ZIP-226 anchor that no longer exists

**Title:** ZIP 227: SigHash consensus rule references
`[#zip-0226-txiddigest]`, but ZIP 226 no longer contains a TxId Digest
section

**Body:**

`zip-0227.rst:474` states the SigHash is computed "with the modifications
described in ZIP 226 [#zip-0226-txiddigest]", and the footnote at line 693
links `zip-0226.rst#txid-digest`. Upstream ZIP 226 contains no "TxId
Digest" section (the v6 digest material lives in the now-withdrawn ZIP
246), so the reference is dangling. Either point the rule at ZIP 246 (with
its Withdrawn caveat) or at whatever ZIP respecifies the v6 digests.

Observed at zcash/zips `69610984` (`zip-0227.rst:474,693`).

---

## 5. zcash/zips — ZIP-226 cites ZIP 2005 under a stale title

**Title:** ZIP 226: ZIP 2005 is cited as "Orchard Quantum Recoverability";
its title is "Ironwood Quantum Recoverability"

**Body:**

`zip-0226.rst:45` reads "ZIP 2005 (Orchard Quantum Recoverability) is
expected to deploy before any ZSA activation", but `zip-2005.md`'s header
titles the ZIP "Ironwood Quantum Recoverability" (Status: Proposed). The
stale name predates the Ironwood renaming and misleads readers about which
pool the recoverability changes target.

Observed at zcash/zips `69610984` (`zip-0226.rst:45` vs `zip-2005.md`
header).

---

## 6. zcash/zips — valueBurn upper bound: "less than" vs "≤"

**Title:** Withdrawn ZIP 230's `valueBurn` bound ("less than
MAX_BURN_VALUE") contradicts ZIP 226's rule (`v ≤ MAX_BURN_VALUE`)

**Body:**

ZIP 226's burn consensus rule 2 requires
`v > 0 and v <= MAX_BURN_VALUE` (`zip-0226.rst:224`), while ZIP 230's
AssetBurn field table says `valueBurn` is "checked by consensus to be
non-zero, and less than MAX_BURN_VALUE" (`zip-0230.rst:377-380`),
excluding equality. The reference implementation rejects only amounts
strictly greater than `MAX_BURN_VALUE = (1 << 63) - 1`, i.e. permits
equality, agreeing with ZIP 226 (QED-it/orchard
`src/bundle/burn_validation.rs:16,100` @ `cf801a5d`). ZIP 230 is
Withdrawn, but any respecification inheriting its field table inherits
the off-by-one wording.

Observed at zcash/zips `69610984`; QED-it/orchard `cf801a5d`.

---

## 7. zcash/zips + QED-it/orchard — empty non-finalizing IssueAction:
implementation rule without a ZIP-227 consensus rule

**Title:** `IssueActionWithoutNoteNotFinalized` has no counterpart MUST in
ZIP 227

**Body:**

The reference implementation rejects an `IssueAction` with no notes unless
it is finalized (`IssueActionWithoutNoteNotFinalized`, QED-it/orchard
`src/issuance.rs:216-219` @ `cf801a5d`). ZIP 227's consensus-rule list
(`zip-0227.rst:470-500`) contains no such MUST, and ZIP 230 merely notes
`nNotes` may be zero "to only finalize". A ZIP-conformant validator would
accept an empty, non-finalizing action that the reference implementation
rejects — a consensus-splitting divergence if nodes disagree on which
artifact is normative. Either ZIP 227 should add the rule or the
implementation should drop the check.

Observed at zcash/zips `69610984`; QED-it/orchard `cf801a5d`.

---

## 8. QED-it forks — code diverges from upstream ZIP-226/246 as rebased

**Title:** ZSA fork stack lags upstream ZIP text: split-note ψ^nf
derivation, lead byte, and ZIP-246 digest tree

**Body:**

Three code-versus-upstream divergences at the current pins, each blocking
byte-compatibility with the upstream texts as rebased in 2026:

1. **Split-note ψ^nf.** Upstream ZIP 226 derives
   `ψ^nf = ToBase(PRFexpand_{rseed_nf}(0x0A || ρ_old))`, domain byte
   `0x0A` reserved by ZIP 2005 (`zip-0226.rst:297`); the fork ZIP samples
   ψ^nf uniformly at random on F_{q_P} (zips-zsa `zip-0226.rst:293` @
   `fd71419a`); the implementation uses the ordinary PSI byte `0x09`
   keyed by a fresh `rseed_split_note` (QED-it/orchard
   `src/circuit.rs:316-319`, `src/note.rs:417-425`; QED-it/zcash_spec
   `src/prf_expand.rs:89` @ `d5e84264`).
2. **Lead byte.** Upstream ZIP 230/226 replaced `0x03` with the
   `{{LEADBYTE}}`/`{{ZSALEADBYTE}}` placeholders (ZIP 2005 claims `0x03`);
   the implementation hard-codes `NOTE_VERSION_BYTE_V3 = 0x03`
   (QED-it/orchard `src/primitives/zcash_note_encryption_domain.rs:49`).
3. **ZIP-246 digest tree.** `librustzcash-zsa`'s txid omits the T.1f fee
   field ("TODO: Factor this out … when implementing ZIP 246 in full",
   `transaction/txid.rs:256`) and the T.6 memo branch (five-branch
   `to_hash`, `transaction/txid.rs:415-460`), and inserts a non-spec
   per-action-group memos digest with personalization `ZTxIdOrcActMHash`
   (QED-it/orchard `src/bundle/commitments.rs:27`).

All three are known interim states, but none is tracked in the upstream
ZIP text or fork issue trackers as far as the pinned trees show.

Observed at QED-it/orchard `cf801a5d`, QED-it/librustzcash `3f9b53fc`,
QED-it/zips `fd71419a`, zcash/zips `69610984`.

---

# Upstream defects found while writing the Crosslink Guide

Verified firsthand 2026-07-15 at tfl-book `fe6e1d6f` (upstream HEAD
unchanged; repository transferred to `github.com/daira/tfl-book`, rendered
at `daira.github.io/tfl-book`) and crosslink-spec `8edc3e94`.

---

## 9. tfl-book — Final Agreement safety theorem carries the wrong proof

**Title:** Security analysis: the proof under "Safety Theorem: Final
Agreement of Π_bft implies Assured Finality" is a verbatim duplicate of
the Prefix Agreement proof

**Body:**

In `src/design/crosslink/security-analysis.md`, the proof text under
"Safety Theorem: Final Agreement of Π_bft implies Assured Finality"
(lines 87-91) is byte-identical to the proof of "Safety Theorem: Prefix
Agreement of Π_bc at confirmation depth σ implies Assured Finality"
(lines 73-77): it invokes "Prefix Agreement at confirmation depth σ" and
never uses Final Agreement. The theorem is the design's load-bearing
safety claim for the case of a subverted PoW layer, and it currently has
no written proof; the intended route presumably runs through the
Linearity rule (cf. the sketch at `construction.md:592`). The defect
persists on the rendered book (`daira.github.io/tfl-book`, fetched
2026-07-15).

Observed at tfl-book `fe6e1d6f` (`security-analysis.md:73-77` vs
`87-91`).

---

## 10. informalsystems/crosslink-spec — first `add_pow` block collides
with the genesis hash

**Title:** `add_pow` assigns `PoWHashed(pow_store.size())`, so the first
added PoW block duplicates the genesis hash and self-parents

**Body:**

In `validity.md`, `PoWGenesis` is `{hash: PoWHashed(1), parent:
PoWHashed(0)}` and `add_pow` assigns `hash:
PoWHashed(pow_store.size())`. With genesis pre-loaded, the first
`add_pow` in every trace therefore mints `{hash: PoWHashed(1), parent:
PoWHashed(1)}` — a phantom, self-parented duplicate of the genesis hash;
all later hashes are fresh (the store grows strictly). The merge commit
`8edc3e9` ("added +1 to make bft blocks unique") fixed the analogous
collision on the BFT side only (`try_add_bft` uses
`BFTHashed(length()+1)`).

Impact: confined to the root. `prefix_inv` cannot be spuriously
satisfied (non-root hashes are unique, and descent from the universal
root is vacuous), so the published model-checking result stands; but
`sigmaConfirmed(genesis, ·, 1)` accepts the phantom block as a
confirmation-tail witness at the verification scope
(`confirmation_depth = 1`, stores ≤ 8), so checked coverage differs
slightly from the intended semantics. A model artifact, not a soundness
gap; the fix is to seed PoW hashes past the genesis value as the BFT
side already does.

Observed at crosslink-spec `8edc3e94` (`validity.md`: `PoWGenesis`,
`add_pow`, `sigmaConfirmed`, `isDescendant`).
