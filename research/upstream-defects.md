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

---

# Wave-0 model-diff findings (DRAFT — needs owner confirmation before filing)

**DRAFT.** These 11 entries come from a breadth-first model-diff pass
(2026-07-16), firsthand-cited but not yet owner-reviewed. Do not file upstream
until each is confirmed. Every entry is a case where the guide is on the correct
side; the defect is in the upstream source. Numbering continues the dossier
(11-21). Ground-truth pins: zcash/zips `69610984` (protocol.tex `662dc87`);
orchard HEAD `bef8a27`; librustzcash `e30517e4`; zebra-crosslink `6d02a1b`;
tfl-book `fe6e1d6f`; voting-circuits `4c39abd`; vote-sdk `cb915f5`; RFC 9591;
ePrint 2024/436.

Two are `medium` (both crosslink code bugs); the rest are `low` (spec/impl
divergences and stale citation apparatus).

---

## 11. zcash/librustzcash — NU6.3 Mainnet activation height hardcoded where ZIP-258 says TBD

**DRAFT. Severity: low.**

**Title:** `MainNetwork::activation_height` pins an NU6.3 Mainnet height
(3,428,143) while ZIP-258 leaves Mainnet TBD

**Body:**

`consensus.rs:496` returns `NetworkUpgrade::Nu6_3 => Some(BlockHeight(3_428_143))`
— a concrete Mainnet activation height — but `zip-0258.md:68-70` gives
`ACTIVATION_HEIGHT (NU6.3)` as `Testnet: 4134000` / `Mainnet: TBD`. The Testnet
side agrees on both (`consensus.rs:529` = `4_134_000` = `zip-0258.md:69`), so the
divergence is strictly the Mainnet height. Likely a benign pre-finalization
placeholder rather than a true bug: an implementation ahead of an unfinalized
spec. The Ironwood Guide already reads the spec correctly and flags this
(`ironwood-guide.tex:2050-2055`). Not a duplicate of the existing ZIP-258 mention
in this dossier (entry 2 is the `ScanSummary` counters gap).

Observed at librustzcash `e30517e4` (`components/zcash_protocol/src/consensus.rs:496,529`);
zcash/zips `69610984` (`zip-0258.md:68-70`).

---

## 12. zcash/zips — protocol.tex Action Statement omits the NU6.3 cross-address input the deployed circuit enforces

**DRAFT. Severity: low.**

**Title:** §7.1.5.4 Action Statement still specifies the pre-Ironwood 9-element
primary input (conditions A1-A9), omitting the tenth flag ZIP-229/258 and the
deployed circuit mandate

**Body:**

The formal Action Statement in `protocol.tex` specifies a 7-logical / 9-flattened
primary input ending at `enableOutputs` (`protocol.tex:8008-8014`; pnote
`8103-8107` lists exactly 9 field elements as `(F_q)^9`), with conditions
stopping at [A8] Enable spend / [A9] Enable output (`8082-8086`). An adversarial
`awk` over `protocol.tex:7980-8140` returns zero tokens matching
cross-address / Ironwood, even though NU6.3 integration has begun elsewhere in
the file (`IronwoodPool` defined; `isnusixthree` toggle at `103-104`). The
deployed circuit adds the flag as public input index 9: orchard `bef8a27`
`src/circuit.rs:88` `const DISABLE_CROSS_ADDRESS: usize = 9;`,
`synthesize_cross_address_checks` (`920-960`) enforcing four affine-coordinate
equalities, gated at `1052-1053` behind `supports_cross_address_restriction`
(true only for `OrchardCircuitVersion::Ironwood`, `1599-1601`). The ZIPs mandate
it: `zip-0229.md:200,210,448` define `enableCrossAddress` as flag bit 2 "(new at
NU6.3)"; `zip-0258.md:91-94` requires every Orchard-pool Action be created with
cross-address transfers disabled, "enforced by the Action circuit verifying key".
The gap is acknowledged as deferred: `zip-0258.md:146-148` ("Many changes are
required throughout the specification ... not spelled out here"); the owning
circuit ZIP-2006 is a 273-byte `Reserved` stub; ZIP-229/258 are Draft and NU6.3
is unactivated. The Ironwood Guide documents the flag correctly (Def A10,
`ironwood-guide.tex:1517-1525`; remark `1528-1547`, noting the detail is
reconstructable only from the implementation).

Observed at zcash/zips `69610984` (protocol.tex `662dc87`, `§7.1.5.4`); orchard
`bef8a27` (`src/circuit.rs:76,88,920-960,1052-1053,1599-1601`).

---

## 13. zcash/librustzcash — zip321 CHANGELOG claims zero-valued transparent outputs are consensus-rejected

**DRAFT. Severity: low.**

**Title:** `zip321` CHANGELOG asserts a per-output positivity consensus rule for
transparent outputs that the protocol spec does not impose

**Body:**

`components/zip321/CHANGELOG.md:49` (in the `[0.7.0] - 2026-04-23` entry) states
"Zero-valued transparent outputs are rejected by the Zcash consensus rules, and
so payments to transparent addresses with the `amount` parameter explicitly set
to zero are now disallowed." The protocol spec imposes no per-output positivity
consensus rule: `protocol.tex:4091`'s sole constraint is "The remaining value in
the `transparentTxValuePool` MUST be nonnegative"; the transaction consensus
block (`13560-13600`) and transparent-output description (`4130-4165`) constrain
only `txOutCount` and the coinbase no-transparent-outputs rule, and transparent
outputs are "encoded as in Bitcoin" (`13334`/`13447`) with no per-output nonzero
rule (the `[0, MAX_MONEY]` range at `6392` applies to Sapling output
descriptions). So an individual zero-valued transparent output is
consensus-valid; it is rejected only by mempool standardness (dust) policy —
corroborated by Zebra's `IsDust` standardness error
(`zebrad/src/components/mempool/storage.rs:139-140`). The impl documentation
therefore contradicts the spec. The Wallet Guide independently identifies the
CHANGELOG assertion as inaccurate (divergence D7, `wallet-guide.tex:5335-5352`).

Observed at librustzcash `e30517e433` (`components/zip321/CHANGELOG.md:49`);
zcash/zips `69610984` (`protocol.tex:4091`).

---

## 14. zcash/zips — ZIP 227 cites ZIP 209 under a stale title (stray "Shielded")

**DRAFT. Severity: low.**

**Title:** ZIP 227 references "ZIP 209: Prohibit Out-of-Range Shielded Chain
Value Pool Balances"; the stray "Shielded" contradicts ZIP 209's own header and
ZIP 226's citation

**Body:**

`zip-0227.rst:688` cites "ZIP 209: Prohibit Out-of-Range **Shielded** Chain Value
Pool Balances", but `zip-0209.rst:4`'s own header reads "Prohibit Out-of-Range
Chain Value Pool Balances" (no "Shielded"), and `zip-0226.rst:446` cites the same
ZIP without "Shielded". A half-completed rename: the historical title was
"Prohibit **Negative Shielded** Chain Value Pool Balances" (still in the QED-it
fork — `zip-0209.rst:4` and `zip-0227.rst:688` both read that). Upstream changed
"Negative" -> "Out-of-Range" and dropped "Shielded" in the ZIP-209 header and in
ZIP-226's citation, but in `zip-0227.rst:688` changed only "Negative" ->
"Out-of-Range", leaving the stray "Shielded". Same class as dossier entry 5
(stale ZIP-2005 title). The ZSA Guide glosses it as a benign fork-vs-upstream
"one reference title" difference (`zsa-guide.tex:320-321`) — worth upgrading to a
flagged defect there.

Observed at zcash/zips `69610984` (`zip-0227.rst:688` vs `zip-0209.rst:4` /
`zip-0226.rst:446`); QED-it/zips fork for the half-rename corroboration.

---

## 15. zebra-crosslink — fat-pointer deserializer reads a reversed slice range and panics on every valid input

**DRAFT. Severity: medium.**

**Title:** `FatPointerToBftBlock{,2}::try_from_bytes` reads the u16 length prefix
from `bytes[76 - 32..2]` (= `bytes[44..2]`, reversed), panicking on every input
that clears its own length guard

**Body:**

The serializer writes the u16 signature-count immediately after the 44-byte vote
field (offset 44): `zebra-chain/src/block/header.rs:164-165`
(`buf.extend_from_slice(&self.vote_...)` then
`buf.extend_from_slice(&(self.signatures.len() as u16).to_le_bytes())`;
byte-identical copies at `lib.rs:1887-1888` and `malctx.rs:171-172`). The
deserializer reads a reversed range — all three copies contain
`let len = u16::from_le_bytes(bytes[76 - 32..2].try_into().unwrap()) as usize;`
(`header.rs:178`, `lib.rs:1900`, `malctx.rs:184`). `76 - 32 = 44`, so the slice
is `bytes[44..2]` (start 44 > end 2). The length guard one line above
(`if bytes.len() < 76 - 32 + 2 { return None; }` at `header.rs:174` /
`lib.rs:1896` / `malctx.rs:180`) returns `None` only when `len < 46`, so every
input that clears it (`len >= 46` — including the minimal 46-byte null fat pointer
that `to_bytes()` emits) reaches the reversed slice and panics. Each function is
prefixed with `#[allow(clippy::reversed_empty_ranges)]` (`header.rs:172`,
`lib.rs:1894`, `malctx.rs:178`), silencing the exact lint that flags reversed
ranges. Empirically reproduced: `&bytes[76 - 32..2]` on a 46-byte `Vec` panics
`slice index starts at 44 but ends at 2`. The intended range is `bytes[44..46]`.
No callers exist at this commit (`try_from_bytes` appears exactly 3 times, all as
the `pub fn` definitions), so this is a latent panic — but the round-trip
`to_bytes()` -> `try_from_bytes()` is provably broken for every non-rejected
input, and fires the moment deserialization is wired in. The Crosslink Guide
describes the fat pointer and its conversion (`crosslink-guide.tex:2290-2303`)
but does not flag the bug.

Observed at zebra-crosslink `6d02a1b80f896d08f923e39b2505f0565efb5787`
(`zebra-chain/src/block/header.rs:172,178`; `zebra-crosslink/src/lib.rs:1894,1900`;
`zebra-crosslink/src/malctx.rs:178,184`).

---

## 16. zebra-crosslink — decided-block handlers finalize from the tip header while `finalization_candidate()` returns the deepest

**DRAFT. Severity: medium.**

**Title:** `finalization_candidate()` returns `headers.last()` (deepest) but both
tenderlink decided-block handlers compute the new final PoW hash from
`headers.first()` (tip); the reconciling assertion is commented out

**Body:**

Two code paths disagree on which bc-header of a decided `BftBlock` identifies the
newly finalized PoW block. `chain.rs:124-125`:
`pub fn finalization_candidate(&self) -> &BcBlockHeader { &self.headers.last()... }`
— returns the deepest header, given the documented tip-first in-memory order
that is "reversed from the specification" (`chain.rs:52-54`, quoting the spec's
"deepest first"; built tip-first, so `last()` = deepest). But `lib.rs:543` and
`lib.rs:809` both compute
`let new_final_hash = new_block.headers.first().expect("at least 1 header").hash();`
— the tip header. The reconciling assertion is disabled at `lib.rs:545`
(`// assert_eq!(new_final_height.0, new_block.finalization_candidate_height);`).
Spec: tfl-book `construction.md` @ `fe6e1d6f:420-427` — `headers_bc` is
"zero-indexed, deepest first" and `snapshot(B) := B.headers_bc[0]`, so the
finalization point derives from the deepest header; the live handlers use the
tip. Corroboration that `finalization_candidate_height` is the deepest (min)
height: `viz.rs:3028-3041` (`min_hdr_h = finalization_candidate_height`,
`max_hdr_h = ... + headers.len() - 1`), so the disabled `assert_eq!` would fail
for σ>1 — the tip is `max`, the candidate is `min`. With σ=3 the two selections
are three heights apart, so the implementation over-finalizes toward the tip on
those paths. The same first/last split recurs in live viz code (`viz.rs:974`
`last()` vs `999`/`1013` `first()`). The Crosslink Guide states this accurately
(`crosslink-guide.tex:2335-2344`).

Observed at zebra-crosslink `6d02a1b` (`chain.rs:52-54,124-125`;
`lib.rs:543,545,809`; `viz.rs:974,999,1013,3028-3041`); tfl-book `fe6e1d6f`
(`construction.md:420-427`).

---

## 17. zcash/zips — ZIP 312 cites stale RFC 9591 appendix letters (off by one)

**DRAFT. Severity: low.**

**Title:** ZIP 312 labels RFC 9591 Trusted Dealer Key Generation as "Appendix B"
and Random Scalar Generation as "Appendix C"; in the published RFC these are
Appendices C and D

**Body:**

`zip-0312.rst:444` cites `[#frost-tdkg]` as "Appendix B: Trusted Dealer Key
Generation" and `:445` cites `[#frost-randomscalar]` as "Appendix C: Random
Scalar Generation". In published RFC 9591 (June 2024, local `rfc9591.txt`):
`:1686` "Appendix C. Trusted Dealer Key Generation", `:1944` "Appendix D. Random
Scalar Generation". An "Appendix A. Schnorr Signature Encoding" (`:1616`) was
inserted in the final RFC, shifting every appendix letter by one relative to
`draft-irtf-cfrg-frost-11`, against which the ZIP was written. So tdkg is labeled
B but is really C, and randomscalar is labeled C but is really D — a reader
following letter B lands on Schnorr Sig Gen/Verification. Only the human-readable
titles are stale; the URL anchors (`#name-trusted-dealer-key-generati`,
`#name-random-scalar-generation`) still resolve. Same root cause as the
§7.3->§7.5 mislabel (entry 20). The FROST Guide is correct: `frost-guide.tex:871`
cites Appendix C for trusted-dealer keygen (matching RFC 9591) and only catches
the analogous section mislabel at `1343-1345`, not these appendix letters.

Observed at zcash/zips `69610984` (`zip-0312.rst:444-445`); RFC 9591
(`rfc9591.txt:1616,1686,1944`).

---

## 18. ePrint 2024/436 — Fig. 5 challenge-hash argument order self-inconsistent

**DRAFT. Severity: low.**

**Title:** In the Rerandomized-FROST construction figure, `Sign'`/`Combine`
compute `c <- H_sig(PK-bar, R, m)` while `Verify` computes
`c <- H_sig(PK-bar, m, R)`

**Body:**

Paper ePrint 2024/436, Fig. 5 (`rerand-2024-436.txt`): `Sign'` (`467-468`) and
`Combine` (`483-484`) use `Hsig(PK-bar, R, m)`; `Verify` (`501-502`) uses
`Hsig(PK-bar, m, R)`; caption `:510`; the reduction's random oracle queries
`Hsig` on `(PK, m, R)` (`:598`). `Hsig` is defined `{0,1}* -> Z_p` (`:447`), so
argument order is significant — taken literally a signature produced by
`Sign'`/`Combine` would not verify under `Verify`. Both orderings also differ
from RFC 9591 §4.6 / RedDSA's fixed order `(R, PK, m)` that the deployed `reddsa`
crate uses, so there is no deployment impact: a preprint typo with no normative
weight. The FROST Guide quotes both orderings and reads the mismatch as a
presentational slip (`frost-guide.tex:1119-1129`).

Observed at ePrint 2024/436 (`rerand-2024-436.txt:447,467-468,483-484,501-502,598`).

---

## 19. ePrint 2024/436 — construction figure cross-referenced as "Figure 4" but captioned "Fig. 5"

**DRAFT. Severity: low.**

**Title:** The Rerandomized-FROST construction (captioned Fig. 5) is referred to
as "Figure 4" twice in Section 5; "Figure 4" is overloaded with the
unforgeability game

**Body:**

Caption `rerand-2024-436.txt:510` "Fig. 5. Rerandomized-FROST ... Differences
with plain FROST are highlighted in a grey box." Section 5 prose points at that
construction as "Figure 4" twice: `:402` "We highlight the changes to plain FROST
in grey in Figure 4." and `:404` "We show the exact details of Rerandomized-FROST
in Figure 4." Both descriptions match the Fig. 5 caption verbatim. Meanwhile
Figure 4 is genuinely a different figure — `:350` "Fig. 4. The (static)
unforgeability games ..." — correctly referenced at `:387` and Def. 6 (`:390`).
A grep of every in-text "Figure N" confirms "Figure 5" never appears, so the
construction's true number is never cited and "Figure 4" is overloaded. The FROST
Guide is accurate: construction header `frost-guide.tex:1089` cites "Figure 5", a
footnote at `:1128` notes the paper twice mislabels it "Figure 4", and `:1165`
refers to the game as "its Figure 4".

Observed at ePrint 2024/436 (`rerand-2024-436.txt:350,387,390,402,404,510`).

---

## 20. zcash/zips — ZIP 312 cites the wrong RFC 9591 section for "Removing the Coordinator Role"

**DRAFT. Severity: low.**

**Title:** ZIP 312 cites RFC 9591 "Section 7.3: Removing the Coordinator Role";
the material is §7.5 (§7.3 is "Nonce Reuse Attacks")

**Body:**

`zip-0312.rst:441` cites `[#frost-removingcoordinator]` as "RFC 9591 ... Section
7.3: Removing the Coordinator Role". In RFC 9591 (`rfc9591.txt`): `:1423` "7.3.
Nonce Reuse Attacks", `:1442` "7.5. Removing the Coordinator Role" (TOC
corroborates: `:89` 7.3 Nonce Reuse, `:91` 7.5 Removing the Coordinator Role).
The printed number "7.3" is stale draft-vs-final drift; the URL anchor
`#name-removing-the-coordinator-ro` targets the correct 7.5 heading. Footnote
consumed at `zip-0312.rst:113-114`. The FROST Guide flags this correctly
(`frost-guide.tex:1343-1347`).

Observed at zcash/zips `69610984` (`zip-0312.rst:441`); RFC 9591
(`rfc9591.txt:1423,1442`).

---

## 21. voting-circuits — README and an in-code "Spec" note say 16 proposals/round; circuit and chain enforce 15

**DRAFT. Severity: low.**

**Title:** `MAX_PROPOSAL_AUTHORITY` documentation claims 16 proposals per round,
but bit 0 is a permanent sentinel so the deployed circuit and chain cap at 15

**Body:**

voting-circuits `4c39abd`, `src/delegation/README.md:300`:
"`MAX_PROPOSAL_AUTHORITY`: `2^16 - 1 = 65535`. A 16-bit bitmask authorizing voting
on all 16 proposals." — asserts 16. Contradicted within the same repo by
`src/vote_proof/circuit.rs:124-127`: "valid values are 1-15. Bit 0 is permanently
reserved as the sentinel/unset value and is rejected by the non-zero gate in
`AuthorityDecrementChip` (`q_cond_6`). This means a voting round supports at most
15 proposals, not 16." (also `circuit.rs:143` `MAX_PROPOSAL_ID: usize = 16`;
`:156` "Only bits 1-15 correspond to usable proposals"). The deployed cap is 15:
vote-sdk `cb915f5`, `x/vote/types/keys.go:48` `const MaxProposals = 15` ("bit 0
reserved as a sentinel ... leaving bits 1-15 usable"), enforced at
`x/vote/types/msgs.go:42-43`. The Voting Guide states 15 correctly
(`voting-guide.tex:835-840`) but does not flag this README-vs-circuit
self-contradiction, and its claim that the quantization gap "is the only
spec-versus-code disagreement this volume records" (`voting-guide.tex:332,741`)
overlooks this second one — a guide gap to fix alongside filing this defect.

Observed at voting-circuits `4c39abd` (`src/delegation/README.md:300`;
`src/vote_proof/circuit.rs:124-127,143,156`); vote-sdk `cb915f5`
(`x/vote/types/keys.go:48`, `x/vote/types/msgs.go:42-43`).

---

# Wave-1 model-diff findings (DRAFT — needs owner confirmation before filing)

**DRAFT.** These 11 entries come from the Wave-1 breadth-first model-diff pass
(2026-07-16), firsthand-cited but not yet owner-reviewed. Do not file upstream
until each is confirmed. Numbering continues the dossier (22-32). Ground-truth
pins unchanged: zebra-crosslink `6d02a1b` (verified HEAD =
`6d02a1b80f896d08f923e39b2505f0565efb5787`); orchard `bef8a27`; tfl-book
`fe6e1d6f`; voting-circuits `4c39abd`; vote-sdk `cb915f5`. All paths under
zebra-crosslink are `zebra-crosslink/zebra-crosslink/…` unless noted `zebra-chain/…`
/ `zebra-state/…` / `zebra-consensus/…`.

Severity spread: **2 high, 4 medium, 5 low** — a step up from Wave-0 (whose
ceiling was medium). Both highs and all four mediums are zebra-crosslink code
bugs in the BFT / finalization path; several cluster on one file (`lib.rs`).

**Dedup notes.** This wave's raw yield was 16 findings; four pairs/triples were
folded: notarization (raw 6+13+15 → **22**), fin-extraction depth off-by-one
(raw 4+10 → **23**), `finalization_candidate_height = 0` (raw 5+14 → **26**). A
16th raw finding (voting-circuits README "16 proposals") is a **duplicate of
Wave-0 entry 21** and is *not* re-filed here; see the note after entry 32.

---

## 22. zebra-crosslink — BFT notarization verification enforces neither roster membership nor the 2/3 supermajority (and accepts an empty signature set)

**DRAFT. Severity: high.**

**Title:** Fat-pointer / notarization verification only batch-verifies ed25519
signatures against keys carried in the fat pointer itself — no roster check, no
2/3-voting-power threshold, and an empty signature set verifies as valid

**Body:**

The decided-BFT-block handler `new_decided_bft_block_from_malachite`
(`lib.rs:499-620`, invoked via `ClosureToPushDecidedBlock` at `lib.rs:1234-1247`,
which crosslink-finalizes the block through `CrosslinkFinalizeBlock` at `:581`)
gates acceptance on exactly one cryptographic check: `lib.rs:531-535`,
`// TODO: check public keys on the fat pointer against the roster` immediately
followed by `if fat_pointer.validate_signatures() == false { error!(…); panic!(); }`.
There is no summation of voting power and no 2/3 comparison anywhere in the
function; a whole-crate grep for `two-thirds|supermajority|2/3|threshold|quorum`
finds only PoW `difficulty_threshold`.

`validate_signatures()` (`lib.rs:1933-1943`, byte-identical copy at
`malctx.rs:217-227`) builds an `ed25519_zebra::batch::Verifier`, and for each
`(vote, signature)` from `self.inflate()` queues `(vk_bytes =
vote.validator_address.0, sig, msg = vote.to_bytes())`, returning
`batch.verify().is_ok()`. `inflate()` (`lib.rs:1922-1932`) sets
`vote.validator_address = MalPublicKey2(MalPublicKey::from(s.public_key))` — the
verification key is read from the *same* `FatPointerSignature2 { public_key:
[u8;32], vote_signature: [u8;64] }` (`lib.rs:1989-1991`) that carries the
signature. So the check is tautological under attacker-chosen keys: it proves
only "these are valid signatures by these keys", never that the keys are roster
members or that their weight meets 2/3. Independently, an **empty** signature set
verifies as `Ok(())`: with `self.signatures == []` the batch is empty, so (ed25519-zebra
4.1.0 `batch.rs` `verify()`) `B_coeff = Scalar::ZERO`, `check =
vartime_multiscalar_mul(once(&0), once(&B)) = identity`, and
`identity.mul_by_cofactor().is_identity()` is true — `null()` (`lib.rs:1878-1883`)
constructs exactly such a pointer.

The sibling assertion `validate_bft_block_from_malachite_already_locked`
(`lib.rs:790-820`) adds no protection — it checks only prev-block fat-pointer
linkage (`:798-807`) and that the final hash exists at some PoW height
(`:809-819`), then returns `TMStatus::Pass`. `voting_power` /
`validators_at_current_height` are used only for the malachite-bug roster
workaround (`:464`, `:510`), reward apportionment (`:654-718`), and roster-update
commands (`:988-1074`) — never to validate a fat pointer.

This is the load-bearing check the guide's Assured-Finality safety proof depends
on: `crosslink-guide.tex:1611-1622`/`2440-2443` require "at least a two-thirds
absolute supermajority of the epoch's voting units to have been cast for a
proposal P", tracing verbatim to tfl-book `construction.md` @ `fe6e1d6f:129`,
`156-161`. The guide is faithful to the spec; the deployed handler has no
enforcement behind it. (The guide flags only the roster-check TODO at `2331-2333`,
not the missing threshold or the empty-batch gap.)

**Failure mode.** Forge `FatPointerToBftBlock2` for target block `B` (blake3 =
`H`): set `vote_for_block_without_finalizer_public_key[0..32] = H` (clears the
`:523` hash-binding check), then for each slot generate a fresh ed25519 keypair
`(sk_i, pk_i)` never in the roster, build `vote` with `validator_address = pk_i`,
sign `msg = vote.to_bytes()` with `sk_i`, and push `FatPointerSignature2 {
public_key: pk_i, vote_signature: sig_i }`. `validate_signatures()` batch-verifies
each `sig_i` against `pk_i` and returns `true`; a roster minority holding <2/3, or
an empty set entirely, passes identically. The target block is crosslink-finalized
on a forged notarization. Real-world reachability of this handler depends on
`tenderlink` (a private git dep, quorum-enforced upstream of the predicate today),
but the verification step itself is provably incomplete the moment it faces
untrusted input.

Observed at zebra-crosslink `6d02a1b` (`lib.rs:499-620,523-535,1878-1883,1922-1943,
1989-1991`; `malctx.rs:217-227`); ed25519-zebra 4.1.0 `batch.rs verify()`; tfl-book
`fe6e1d6f` (`construction.md:129,156-161`).

---

## 23. zebra-crosslink — the block that gets finalized is one confirmation shallower than the code's own computed candidate (σ−1, not σ)

**DRAFT. Severity: high.**

**Title:** The decided-block handler finalizes `headers.first()` (= tip−σ+1)
instead of the spec's `snapshot(B) = headers_bc[0]⌈¹` (that header's parent, at
tip−σ); `propose_new_bft_block` computes the correct candidate at tip−σ and then
discards it

**Body:**

`propose_new_bft_block` computes the correct finalization candidate:
`finality_candidate_height = tip_height.sub(bc_confirmation_depth_sigma)` = T−σ
(`lib.rs:379-381`) and fetches `candidate_hash` at exactly that height
(`lib.rs:414-416`). But it then uses `candidate_hash` only as the
`FindBlockHeaders` anchor (`lib.rs:425-429`). `FindBlockHeaders` returns headers
*strictly after* the anchor and in ascending order: `zebra-state
src/service/read/find.rs:392-397` sets `start_height = intersection_height + 1`,
`:414` `height_range = start..=final` ascending, `:485-490` asserts "the list
must not contain the intersection hash". With `MAX_FIND_BLOCK_HEADERS_RESULTS =
160` (`constants.rs:110`) ≥ σ and σ = 3 (`chain.rs:220`), `headers = [T−σ+1 … T]`,
so the anchor block (T−σ) is *excluded* and `headers[0]` is the deepest returned
header at T−σ+1. `new_decided_bft_block_from_malachite` then finalizes
`new_final_hash = new_block.headers.first().hash()` = T−σ+1 (`lib.rs:543-544`) and
issues `CrosslinkFinalizeBlock(new_final_hash)` (`lib.rs:579-587`). The spec
finalizes `snapshot(B) := headers_bc[0]⌈¹_bc` — the *parent* of the deepest
header, at depth σ (tfl-book `construction.md` @ `fe6e1d6f:420-428`;
`candidate(H) := lca(snapshot(LF(H)), H⌈^σ)` at `:434`; the fin-depth lemma
`:490-495`, `:510-513` requires the finalized point at depth ≥ σ for Assured
Finality). So the code finalizes the deepest *header itself* (depth σ−1), one
block too shallow, and the correcting assert `lib.rs:545`
(`// assert_eq!(new_final_height.0, new_block.finalization_candidate_height)`) is
commented out.

This is distinct from Wave-0 entry 16 (the `headers.first()` vs
`finalization_candidate() = headers.last()` accessor disagreement): the off-by-one
persists even if that first/last split is reconciled, because the correct target
is `headers.first().parent`, not either end of the vector. The guide's own
header-order bullet carries a TODO admitting the direction is unresolved
(`crosslink-guide.tex:~2345`), while `1725-1731` states the correct rule ("The
snapshot is the parent of the deepest supplied header").

A related second-order effect: `is_improved_final` (`lib.rs:404`) gates on
`cand_height > latest` (height-only), comparing T−σ against the previously
recorded T′−σ+1, so finality only advances when the tip grows by ≥2 blocks — if
PoW yields exactly one block per BFT round, no proposal is emitted and finality
stalls by a block until a 2-block gap appears.

**Failure mode.** With σ = 3 and PoW tip at height T, every decided BFT block
finalizes T−2 (2 confirmations) rather than the intended candidate/snapshot at
T−3 (3 confirmations). T−2 is not ⪯ H⌈^σ = T−3, violating the σ-confirmation
invariant the Assured Finality proof depends on — every finalization is one
confirmation shallower than the protocol's safety margin, and the code holds the
correct answer (`candidate_hash` = T−3) and throws it away.

Observed at zebra-crosslink `6d02a1b` (`lib.rs:379-381,404,414-429,543-545,
579-587`; `zebra-state src/service/read/find.rs:392-397,414,485-490,526`;
`constants.rs:110`; `chain.rs:220`); tfl-book `fe6e1d6f`
(`construction.md:420-428,434,490-495,510-513`).

---

## 24. zebra-crosslink — consensus-path callbacks `panic!()` instead of returning an error (liveness/DoS)

**DRAFT. Severity: medium.**

**Title:** The tenderlink historical-block closure is an unconditional
`panic!()`, and the decided-block handler `panic!()`s on fat-pointer
hash-mismatch / signature-failure — both abort the node instead of rejecting

**Body:**

`tfl_service_main_loop` (spawned at `service.rs:207`) unconditionally spawns
`tenderlink::entry_point` at `lib.rs:1205` with a set of closures — the spawn
sits in the plain block opened at `:1116`, with no `TEST_MODE`/`cfg(test)`/validator
gate between `:1099` and `:1205` (the only `TEST_MODE` branch, `run_tfl_test` at
`:1098`, is separate). Two closures abort the process:

1. **Historical-block closure**, `lib.rs:1248-1252`:
   `ClosureToGetHistoricalBlock(Arc::new(move |height| Box::pin(async move {
   panic!(); })))` — the body ignores `height` and is an unconditional panic stub.
   A complete working alternative, `async fn get_historical_bft_block_at_height`
   (`lib.rs:843-861`, bounds-checks height, clones the block, builds the fat
   pointer, returns `Option`), sits a few hundred lines away and has **zero
   callers** (repo-wide grep) — dead code.

2. **Decided-block handler** `new_decided_bft_block_from_malachite`
   (body of `ClosureToPushDecidedBlock` at `:1234-1247`): `panic!()` at
   `lib.rs:529` when `fat_pointer.points_at_block_hash() != new_block.blake3_hash()`,
   and `panic!()` at `:534` when `fat_pointer.validate_signatures() == false`.

That this is a defect, not a simplification, is confirmed by the sibling
`ClosureToValidateProposedBlock` at `lib.rs:1220-1233`, which returns
`tenderlink::TMStatus::Fail` on undeserializable input — the codebase knows the
graceful-reject pattern; these two callbacks do not use it.

**Failure mode.** When tenderlink asks this node for a historical decided BFT
block (normal catch-up sync of a lagging peer), the closure panics and the node
crashes — a remotely-reachable liveness failure with a ready alternative left
unused. Separately, any decided value delivered with a malformed/invalid fat
pointer crashes the node via `:529`/`:534` rather than being rejected.

Observed at zebra-crosslink `6d02a1b` (`lib.rs:843-861,529,534,1205,1220-1233,
1234-1252`; `service.rs:207`). Distinct from Wave-0 entries 15 (`try_from_bytes`
reversed range) and 16 (fin `first`/`last`): this is the panic-on-failure pattern
in registered consensus callbacks.

---

## 25. zebra-consensus / zebra-crosslink — BFT voting power (roster) is mutated by staking transactions that are neither stake-backed nor per-action authorized

**DRAFT. Severity: medium.**

**Title:** A zero-input/zero-output `VCrosslink` staking transaction passes
`has_inputs_and_outputs` and adds arbitrary `voting_power` to an
attacker-chosen finalizer key — no locked ZEC, no signature by the target key,
power-of-10 rule unenforced

**Body:**

Design (nutshell.md / guide) requires the roster to be both value-backed and
key-authorized: `zebra-crosslink book/src/design/nutshell.md:96,98,102,104` — a
finalizer's voting weight is the sum of staking positions, each needing a
finalizer verification key + staked ZEC, with an unstaking/redelegation key
authorizing changes; `:86` amounts must be powers of 10 ZAT;
`crosslink-guide.tex:2386-2388` "Crosslink's finality vote is stake-weighted".

The implementation binds none of this. (1) `zebra-consensus
src/transaction/check.rs:124-130` `has_inputs_and_outputs` returns `Ok(())` for
any `VCrosslink` with `staking_action.is_some()` (comment "TODO: real staking
transactions with inputs/outputs") — a zero-input/zero-output tx is accepted.
(2) `lib.rs:948-965` `push_staking_action_from_cmd_str` builds exactly that tx:
`inputs: Vec::new()`, `outputs: Vec::new()`, both shielded `None`,
`staking_action = StakingAction::parse_from_cmd(cmd_str)`. (3) `lib.rs:988-1061`
`update_roster_for_cmd` Add path: `amount = action.val`; `member.voting_power +=
amount` or `roster.push(MalValidator::new(add_key.0, amount))` where `add_key =
action.target` parsed from the command string — no value binding, no signature by
the target key; the power-of-10 rule is unenforced (grep for
`pow(10`/`ilog10`/`%10` in consensus = none). (4) `lib.rs:697,718` add rewards
straight into `voting_power`. (5) `zebra-consensus src/transaction.rs:573-575`
hard-codes the `VCrosslink` fee ("TODO: remove this complete @Hack").

Crux: `malctx.rs:706-718` `MalValidator` implements
`malachitebft_core_types::Validator`, `voting_power()` returning the field, and
`lib.rs:458-464` serves `validators_at_current_height` as Malachite's current
validator set. So this field *is* the BFT vote weight (and the reward weight,
`lib.rs:654`).

**Failure mode.** Any party broadcasts a zero-value, zero-input `VCrosslink`
staking tx whose `staking_action` adds a large `val` to a finalizer key it
controls. It passes `has_inputs_and_outputs` (no coins required) and
`verify_v5_transaction` (nothing to verify), and every node's
`update_roster_for_cmd` raises that key's `voting_power`. One unbacked,
unauthenticated transaction can seize a supermajority of the roster.

Observed at zebra-crosslink `6d02a1b` (`zebra-consensus src/transaction/check.rs:124-130`,
`src/transaction.rs:573-575`; `lib.rs:458-464,654,697,718,948-965,988-1061`;
`malctx.rs:706-718`; `book/src/design/nutshell.md:86,96,98,102,104`).

---

## 26. zebra-crosslink — `BftBlock.finalization_candidate_height` is hard-coded 0 on every proposed (and therefore every decided) block, and committed into the block's BLAKE3 identity

**DRAFT. Severity: medium.**

**Title:** `propose_new_bft_block` passes the literal `0` as the
`finalization_candidate_height` argument to `BftBlock::try_from`, discarding the
tip−σ it just computed; the field is serialized into the block hash and the
reconciling assert is disabled

**Body:**

`propose_new_bft_block` computes the real candidate `finality_candidate_height =
tip_height.sub(bc_confirmation_depth_sigma)` = tip−σ (`lib.rs:379-391`) and uses
it for the header request (`:414`) and the `is_improved_final` gate (`:404`), but
threads the literal `0` into the block: `lib.rs:443-449`, `BftBlock::try_from(params,
internal.bft_blocks.len() as u32 + 1, internal.fat_pointer_to_tip.clone(), 0,
headers)` — the 4th positional arg binds to `finalization_candidate_height`
(signature `chain.rs:129-135`). `try_from` validates only `headers.len() ==
bc_confirmation_depth_sigma` (`chain.rs:136-140`), logs `error!("not yet
implemented: all the documented validations")` (`:142`), and echoes the value
verbatim into the struct (`:148`). So `0` flows through unreconciled. The field
is documented "The height of the PoW block that is the finalization candidate"
(`chain.rs:70-71`) and is serialized on the wire (`chain.rs:83`
`write_u32(self.finalization_candidate_height)`, deserialized `:97`) and into the
block's BLAKE3 identity (`From<&BftBlock> for Blake3Hash`, `chain.rs:166-178`).
The decided handler carries the block through unchanged (`lib.rs:577`), so every
*decided* block also advertises `0`.

The reconciling consistency check `lib.rs:545`
(`// assert_eq!(new_final_height.0, new_block.finalization_candidate_height)`) is
commented out; with the field pinned at 0 it reduces to "(height of
`headers.first()`) == 0", unsatisfiable off genesis — which is why it is
disabled. This is a distinct root cause from Wave-0 entry 16 (which attributed the
disabled assert to the tip=first vs deepest=last split at σ>1): the field being
hard-coded 0 makes the assert unsatisfiable even at σ=1 where first==last.

That 0 is a bug, not intent, is corroborated by `zebrad/tests/crosslink.rs:687-691`
(the test author populates the field with a real
`pow_blocks[0].coinbase_height().0`) and by the commented-out backfill at
`viz.rs:830-843` treating 0 as a known "unset" sentinel to repair.

**Failure mode.** Given tip H and σ=3, the proposer builds a `BftBlock` whose
`finalization_candidate_height` is 0 instead of H−3, committed into the block's
BLAKE3 identity hash, and the documented invariant (assert at `lib.rs:545`) is
permanently unsatisfiable. Any consumer reading the field mislocates the
finalization point at genesis — live today in a dev tool: `viz.rs:3028,3041,3330`
anchor decided blocks at height 0.

Observed at zebra-crosslink `6d02a1b` (`lib.rs:379-391,404,414,443-449,545,577`;
`chain.rs:70-71,83,97,129-135,136-148,166-178,220`; `viz.rs:830-843,3028,3041,3330`;
`zebrad/tests/crosslink.rs:687-691`).

---

## 27. zebra-crosslink — the finalization path implements none of the book's `updatefin` no-rollback safety logic (no descendant check, no rollback on state error, no hazard record)

**DRAFT. Severity: medium.**

**Title:** `is_improved_final` gates only on strictly-greater height, the decided
handler sets `latest_final_block` unconditionally before the state call and never
rolls it back on `Err`, and no finalization-safety hazard is ever recorded

**Body:**

The book's `updatefin` (tfl-book `construction.md` @ `fe6e1d6f:477-486`) is: `N
:= candidate(ch); if N ⪰ localfin then localfin ← N; else keep old, and if N ⋠
localfin record a finalization-safety hazard` (`:471-473`, hazard record includes
`ch` + fin history, `:488`). The guide reproduces this (def `xl-updatefin`,
`crosslink-guide.tex:1977-1994`) and claims the deployed `latest_final_block`
mirrors the book's node-local `fin_i` state machine (`4147-4153`).

The deployment implements none of the safety logic. `is_improved_final`
(`lib.rs:403-404`, in the *propose* path) = `latest.is_none() || cand_height >
latest.0` — height-only, no descendant/ancestry check. The *decide* handler
`new_decided_bft_block_from_malachite` sets `internal.latest_final_block =
Some((new_final_height, new_final_hash))` **unconditionally** (`lib.rs:579`),
before the state call and with no ⪰/descendant check; on state `Err` it only
`error!(?err)` (`:581-593`) and leaves `latest_final_block` set — no rollback.
`tfl_set_finality_by_hash` (`lib.rs:1476`) is likewise unconditional ("TODO:
sanity checks"). A grep across `zebra-crosslink/src` for
`hazard`/`rollback`/`descendant`/`conflicts` logic returns zero hits. The state
layer offers no protection either: `zebra-state non_finalized_state.rs:264-292`
`crosslink_finalize` uses `find_chain(|c| c.height_by_hash(hash).is_some())` —
*any* chain, no descendant-of-previous check — returning `None` if absent, and
`write.rs:342-380` returns `Err("Couldn't find finalized block")` on
None-and-not-in-DB, which the crosslink side only logs.

This is orthogonal to the guide's already-noted "Linearity/Extension/LFS rules
not enforced" (those are proposal/block-validity rules, `crosslink-guide.tex:2329-2330`);
the `updatefin` no-rollback check is a separate fin-extraction mechanism the guide
does not flag as dropped.

**Failure mode.** A conflicting candidate at strictly greater height (a bc-reorg
deeper than σ, or a subverted/equivocating BFT — the exact break Crosslink's fin
logic is meant to survive) passes the height-only gate; `latest_final_block` is
overwritten with no descendant check and no hazard record. When that hash is on a
fork already pruned/uncommitted, `CrosslinkFinalizeBlock` returns `Err` but
`latest_final_block` still points at the never-committed block — the service
reports a finalized tip inconsistent with the committed chain and silently omits
the Assured-Finality hazard the book requires.

Observed at zebra-crosslink `6d02a1b` (`lib.rs:403-404,579,581-593,1476`;
`zebra-state non_finalized_state.rs:264-292`, `write.rs:342-380`); tfl-book
`fe6e1d6f` (`construction.md:471-473,477-486,488`).

---

## 28. zebra-crosslink — `FatPointerToBftBlock{,2}::zcash_deserialize` pre-allocates from a 2-byte attacker length, bypassing `TrustedPreallocate` (memory-amplification DoS)

**DRAFT. Severity: low.**

**Title:** The fat-pointer deserializer runs `Vec::with_capacity(len)` on a
2-byte attacker-controlled signature count (up to ~6.3 MB) before reading a
single signature byte, hand-rolling around Zebra's `TrustedPreallocate`
DoS-hardening

**Body:**

`FatPointerToBftBlock::zcash_deserialize` (`zebra-chain
src/block/header.rs:251-252`) reads `let len = reader.read_u16::<LittleEndian>()?;`
then immediately `let mut signatures: Vec<FatPointerSignature> =
Vec::with_capacity(len.into());` — the allocation runs before any signature byte
is read (identical duplicate `FatPointerToBftBlock2` at `lib.rs:1972-1973`).
`FatPointerSignature = { public_key: [u8;32], vote_signature: [u8;64] }`
(`header.rs:127-133`), size 96, so `with_capacity(0xFFFF)` reserves 65535·96 =
6,291,360 bytes.

This bypasses the codebase's own DoS convention:
`zebra-chain src/serialization/zcash_deserialize.rs:79-99`
`zcash_deserialize_external_count` checks `external_count > T::max_allocation()`
and returns `SerializationError::Parse("Vector longer than max_allocation")`
*before* its `Vec::with_capacity` (`:95`); the trait docstring (`:178-183`) calls
blind preallocation of a generic `Vec<T>` "a DOS vector". The idiomatic
`impl<T> ZcashDeserialize for Vec<T>` (`:23-32`) routes through that checked path
via CompactSize, but the FatPointer deserializer hand-rolls a raw `u16` read +
`Vec::with_capacity`, so `FatPointerSignature` never uses `TrustedPreallocate` and
no `max_allocation` bound applies. `Block::zcash_deserialize`'s
`.take(MAX_BLOCK_BYTES)` (`serialize.rs:190`) does not help: the allocation
happens on the 2-byte `len` alone, before signature bytes are read.

Reachability is v5+ only: `Header::zcash_deserialize` calls the fat-pointer
deserializer when `logical_version >= 5` (`serialize.rs:124-126`), and
`check_version` (`:36-62`) accepts version 5. Normal Zcash blocks are v4
(`ZCASH_BLOCK_VERSION = 4`), so current exposure is the Crosslink/regtest network;
if Crosslink ships it becomes every block-header parse.

**Failure mode.** A peer sends a v5 header whose 44-byte fat-pointer stub is
followed by `len = 0xFFFF` then EOF. The deserializer reads the stub, reads
`len = 65535`, allocates `Vec::with_capacity(65535)` = 6.29 MB, then the loop's
first `read_exact` of 96 bytes fails with `UnexpectedEof` and the Vec is dropped —
~6.3 MB transient allocation from a 46-byte input (≈136,769× amplification), per
header, pre-validation.

Observed at zebra-crosslink `6d02a1b` (`zebra-chain src/block/header.rs:127-133,
251-252`; `lib.rs:1972-1973`; `zebra-chain src/serialization/zcash_deserialize.rs:23-32,
79-99,178-183`; `zebra-chain src/block/serialize.rs:36-62,124-126,190`).

---

## 29. zebra-crosslink — `TF::read_from_bytes` panics on corrupt/adversarial `.zeccltf` files instead of returning `Err`

**DRAFT. Severity: low.**

**Title:** `read_from_bytes` (contract `Result<Self, String>`) slices with the
file-supplied `instrs_o` offset without any bounds or ordering check, so a
malformed offset panics

**Body:**

`test_format.rs:376` `pub fn read_from_bytes(bytes: &[u8]) -> Result<Self, String>`
is contracted to signal failure gracefully. The write side computes a valid
offset `instrs_o = align_up(size_of::<TFHdr>() + self.data.len(),
align_of::<TFInstr>())` (`:332-336`), but the read side never bounds-checks the
raw file field `tf_hdr.instrs_o` before two slices:
`<[TFInstr]>::ref_from_prefix_with_elems(&bytes[tf_hdr.instrs_o as usize..], …)`
(`:382-383`) and `let data = &bytes[size_of::<TFHdr>()..tf_hdr.instrs_o as usize];`
(`:392`). `TFHdr` (`magic[u8;8] + u64 + u32 + u32`, `repr(C)`) has
`size_of == 24` (`:9-16`).

Non-test reachability: `TF::read_from_file` (`viz.rs:3751`, with graceful `Err(err)
=> error!(…)` at `:3766`) and `read_instrs` (`test_format.rs:675`) both do a plain
`std::fs::read` then `read_from_bytes`, with no validation guard.

**Failure mode.** A file with `instrs_o = 0` (< `size_of::<TFHdr>() = 24`) makes
`bytes[24..0]` a reversed range → panic "slice index starts at 24 but ends at 0";
`instrs_o` greater than the file length → out-of-bounds slice panic "range start
index N out of range for slice of length M". Any tooling loading an untrusted or
corrupt crosslink-test-data file aborts instead of returning `Err`.

Observed at zebra-crosslink `6d02a1b` (`test_format.rs:9-16,332-336,376,382-383,392,
675`; `viz.rs:3751,3766`).

---

## 30. zebra-crosslink — `TF` write/read `data` field does not round-trip (padding is captured into `data`)

**DRAFT. Severity: low.**

**Title:** `write()` pads the data region up to `align_of::<TFInstr>()` before the
instruction table, but `read_from_bytes` reconstructs `data` as
`bytes[24..instrs_o]`, which includes that padding — so `read(write(tf)).data !=
tf.data` whenever `data.len() % 8 != 0`

**Body:**

Write pads and emits zero bytes between data and the instruction table:
`test_format.rs:332` `instrs_o_unaligned = size_of::<TFHdr>() + self.data.len()`;
`:333` `instrs_o = align_up(instrs_o_unaligned, align_of::<TFInstr>())`; `:344`
writes `self.data`; `:347-352` if `instrs_o > instrs_o_unaligned` writes
`align_size` zero bytes. Read reconstructs `data` running all the way to
`instrs_o`, capturing that padding: `:392` `let data =
&bytes[size_of::<TFHdr>()..tf_hdr.instrs_o as usize];`, `:397` `data: data.to_vec()`.
`size_of::<TFHdr>() = 24` (multiple of 8); `align_of::<TFInstr>() = 8`
(`kind:u32, flags:u32, data:TFSlice{o:u64,size:u64}, val:[u64;2]`, `:46-53`), so
`instrs_o > instrs_o_unaligned` exactly when `self.data.len() % 8 != 0`, producing
`8 − (data.len() % 8)` trailing zero bytes.

Masked today because instruction payloads are fetched by *absolute* offset, never
from the `tf.data` vec: `tf_read_instr` (`:442`) reads each payload via
`instr.data_slice(bytes)` → `TFInstr::data_slice` (`:144-146`) →
`TFSlice::as_byte_slice_in` (`:30-32`) `&bytes[o..o+size]` indexing the full
buffer; the driver `read_instrs(internal_handle, &bytes, &tf.instrs)` (called at
`:682`) passes `bytes`, not `tf.data`. No round-trip test exists (no `#[test]` /
`mod tests` in the file), so nothing catches the inequality.

**Failure mode.** Any `TF` whose `data` length is not a multiple of 8 round-trips
with trailing zero bytes appended to `.data`; a re-serialize of the read-back `TF`
would grow the file further and can desync the stored absolute instruction
offsets. Functionally latent only because payloads are fetched by absolute offset.

Observed at zebra-crosslink `6d02a1b` (`test_format.rs:9-16,30-32,46-53,144-146,
325-352,392-397,442,682`).

---

## 31. zebra-crosslink — `BftBlock`'s documented stateless validations are unimplemented, and the deserialize path bypasses the constructor entirely

**DRAFT. Severity: low.**

**Title:** `BftBlock::try_from` (documented as the "only way to construct" a
`BftBlock`, performing four stateless validations) checks only the header count,
hard-codes `version:1`, logs "not yet implemented", and `zcash_deserialize`
never calls it

**Body:**

The contract (`chain.rs:33-38,56`) lists four validations by the "sole
constructor" `try_from` — header count = σ, known version, header ordering by
`previous_block_hash`, PoW solutions validate — and calls it "the safest design",
"the only way to construct". The implementation (`chain.rs:136-151`) checks only
`headers.len()` (else `IncorrectConfirmationDepth`), then `error!("not yet
implemented: all the documented validations")` (`:142`) and hard-codes
`version: 1` (input version ignored). Validations 2/3/4 are absent.
`finalization_candidate()` is annotated `UNVERIFIED` (`chain.rs:123`).
`zcash_deserialize` (`chain.rs:92-118`) builds the struct literal directly
(`:110-116`) — never calls `try_from`; its own `TODO: Ensure deserialization
delegates to BftBlock::try_from` (`:48`) — with only a `header_count > 2048`
sanity cap (`:99-104`), so on the network path not even the count check runs. All
fields are `pub` (`chain.rs:62-74`), refuting "only way to construct" (direct
literals at `lib.rs:551`, `viz.rs:2667/2932`).

The network accept path (`service.rs:172-174` / `lib.rs:1227` →
`validate_bft_block_from_malachite` → `..._already_locked`, `lib.rs:790-820`)
checks only prev-block fat-pointer linkage (`:798-807`) and that
`headers.first()` is a known PoW block (`:809-819`), then returns `Pass`; a
crate-wide grep for `pow`/`equihash`/`check_solution` finds no PoW-solution
verification of BftBlock headers anywhere. (The guide documents this stub
accurately, so the guide is not wrong; filed here as the underlying defect.)

**Failure mode.** A decided/proposed `BftBlock` whose `headers` are not a valid
PoW chain (bad PoW solutions, wrong parent-linkage order, unexpected version) is
accepted: `try_from` validates none of these, and the deserialize path skips
`try_from` so even the count check is absent. The remaining σ−1 headers are never
PoW-verified — latent while only `headers.first()` drives fin-extraction, but the
documented safety property does not hold.

Observed at zebra-crosslink `6d02a1b` (`chain.rs:33-38,48,56,62-74,92-118,123,
136-151`; `lib.rs:551,790-820,1227`; `service.rs:172-174`; `viz.rs:2667,2932`).

---

## 32. zebra-crosslink — `finalization_candidate()` returns the chain tip; the `chain.rs` header-order docstring is inverted

**DRAFT. Severity: low.**

**Title:** The docstring claims the in-memory header order is "reversed from the
specification (built tip-first)", but the sole producer yields ascending /
deepest-first order matching the spec — so `finalization_candidate() =
headers.last()` returns the tip (T), not the σ-confirmed snapshot (~T−σ)

**Body:**

The sole producer, `propose_new_bft_block` via `FindBlockHeaders`, yields
ascending / deepest-first headers: `lib.rs:414-439` anchors at
`finality_candidate_height = tip−σ`; `zebra-state src/service/read/find.rs:414`
`height_range = start..=final` with `start = intersection+1 = tip−σ+1`, `final =
min(tip−σ+160, tip) = tip`, iterated ascending (`:526`); read handler
`service.rs:1720-1743` returns them unreversed. So `headers[0]` = tip−σ+1
(deepest), `headers.last()` = tip. This *matches* the spec's deepest-first
convention (tfl-book `construction.md` @ `fe6e1d6f:420` "exactly σ bc-headers,
zero-indexed, deepest first"; `:427` `snapshot(B) := B.headers_bc[0]⌈¹` = the
deepest header's parent).

Yet `chain.rs:52-54` asserts the in-memory order is "reversed from the
specification (built tip-first)" — factually inverted vs the sole real producer.
Consequently `finalization_candidate()` (`chain.rs:124-126`, `&self.headers.last()`)
returns the *tip* (height T, 0 confirmations) rather than the ~σ-confirmed
snapshot near T−σ; off by σ−1 = 2 blocks. Corroboration: the wired-up validation
path (`lib.rs:809`) uses `headers.first()` (deepest = spec's `headers_bc[0]`), so
the live code and the dead accessor disagree and the dead `last()` one is the
inverted/wrong end. Latent: a grep for `.finalization_candidate(` has zero
callers. This resolves the guide's explicit open TODO
(`crosslink-guide.tex:~2345`, "pin down which end of the header vector is the
snapshot"); the guide itself makes no false claim, it is correctly hedged.

**Failure mode.** `finalization_candidate()`, documented as "the finalization
candidate for this block", returns the block at the chain tip (height T, 0
confirmations) instead of the ~σ-confirmed snapshot near T−σ. Any future caller
that trusts this accessor (e.g. a real validity/finalization path replacing the
current stub) would finalize the tip, σ−1 blocks too shallow. Latent today
because the method has no callers.

Observed at zebra-crosslink `6d02a1b` (`chain.rs:52-54,124-126,220`;
`lib.rs:414-439,809`; `zebra-state src/service/read/find.rs:414,526`,
`service.rs:1720-1743`); tfl-book `fe6e1d6f` (`construction.md:420,427`).

---

**Not re-filed — duplicate of Wave-0 entry 21.** A 16th Wave-1 raw finding
(voting-circuits `src/delegation/README.md:300` "all 16 proposals" vs the
circuit/chain cap of 15) is the same defect as dossier entry 21; the Wave-1
firsthand cross-checks (voting-circuits `4c39abd`
`src/vote_proof/circuit.rs:123-143`, `gadgets/authority_decrement.rs:214,315`,
test `proposal_id_zero_fails` `circuit.rs:2599-2608`; vote-sdk `cb915f5`
`x/vote/types/keys.go:43,48`, `msgs.go:127-128`) strengthen entry 21's evidence
but add no new defect.
