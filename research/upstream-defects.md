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
