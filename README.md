# percolator-nft

Position NFT wrapper for Percolator — mint transferable Token-2022 NFTs representing open perpetual futures positions, with **true escrow custody** of the underlying position while wrapped.

## Architecture

```
percolator-nft (this program)
  ├── Reads portfolio state directly from Percolator portfolio accounts (no CPI for reads)
  ├── SPL Token-2022 (mint/burn position NFTs, decimals=0, supply=1, TransferHook)
  ├── PositionNft PDA (links NFT mint → portfolio + market_id)
  └── CPIs the Percolator core ONLY for custody:
        • Mint  → B-3 TransferPortfolioOwnership (tag 72): escrow portfolio.owner → mint-authority PDA
        • Burn  → UnwrapEscrowedPortfolio       (tag 82): release portfolio.owner → burning holder
```

**Why a wrapper?**
- Core program stays lean — no Token-2022 dependency in the core BPF binary
- Independent upgradability — iterate on NFT logic without touching core
- Security isolation — NFT bugs can't affect core funds
- Same pattern as `percolator-stake`

## Custody model (escrow-at-mint, "frozen while wrapped")

- **Mint** takes true custody: it CPIs the core's B-3 `TransferPortfolioOwnership` (tag 72) to set `portfolio.owner` = this program's mint-authority PDA. While wrapped, the original owner can no longer operate the position (trade / reduce / close / withdraw). This closes the OTC pre-transfer drain window.
- **Transfer** moves only the bearer token; the position stays escrowed (the hook reassigns no ownership — it only *gates* the transfer).
- **Burn / EmergencyBurn** release custody: they CPI the core's `UnwrapEscrowedPortfolio` (tag 82) to return `portfolio.owner` to the burning holder. Because the portfolio is owned by a program PDA while wrapped, only one NFT can exist per portfolio at a time.

## Instructions

| Tag | Name | Caller | Description |
|-----|------|--------|-------------|
| 0 | `MintPositionNft` | position owner | Mint an NFT for an open leg; escrows the portfolio (12 accounts) |
| 1 | `BurnPositionNft` | NFT holder | Burn the NFT, release escrow to the holder (10 accounts) |
| 2 | `SettleFunding` | NFT holder | Refresh the NFT's funding snapshot (holder-only since GH#5) |
| 3 | `GetPositionValue` | anyone | Read-only; emits raw leg fields via `msg!` logs (read with `simulateTransaction`) |
| 4 | `ExecuteTransferHook` | Token-2022 (CPI) | Transfer-hook gate; not called directly |
| 5 | `EmergencyBurn` | NFT holder | Burn + release for a closed/liquidated/slot-reused position (10 accounts) |
| 6 | `RepairExtraMetas` | anyone | Rewrite an NFT's ExtraAccountMetaList to the current layout (permissionless, deterministic) |

(See the per-instruction doc-comments in `src/instruction.rs` for the exact account lists and flags.)

## PDA Seeds

- **PositionNft**: `["position_nft", portfolio_account, market_id_le_bytes]` (keyed on the per-position `market_id`, not the reused `asset_index` — see #108)
- **MintAuthority**: `["mint_authority"]` (program-wide; signs mint-to, mint-close, and the escrow/unwrap CPIs)
- **ExtraAccountMetaList**: `["extra-account-metas", nft_mint]`
- **NftRegistry** (owned by the core wrapper): `["nft_registry", market_group]` under the wrapper program id

## Portfolio layout

The NFT program mirrors the v17 `PortfolioAccountV16Account` layout (`src/slab_types_v16.rs`) to read position state directly without a read-CPI. The mirror is byte-exact against the engine struct; offsets are pinned by compile-time `const_assert!`s.

## Transfer Hook

The Token-2022 TransferHook is a **stateless gate** — it performs no ownership reassignment and no margin/health check. On each transfer it validates: amount == 1, the caller is a genuine Token-2022 `TransferChecked`/`TransferCheckedWithFee` of this mint (#103 rejects plain `Transfer`), the source/dest ATAs, the PositionNft PDA derivation, the per-market NftRegistry, the slot-reuse anchor (`market_id`), and the leg transfer gate (active leg + not locked/stale/resolved/mid-close). Any failure rejects the transfer; the position stays escrowed regardless of where the NFT is held.

## Build and Test

```bash
# Build BPF binary
cargo build-sbf

# Unit tests (pure logic)
cargo test
```

## Security Notes

- `forbid(unsafe_code)` enforced
- Portfolio owner verified against the known Percolator wrapper program IDs (devnet + mainnet), fail-closed
- Position ownership verified before minting; mint escrows the portfolio to a program PDA
- The transfer hook gates transfers (active/clean leg, registry, market_id anchor); it reassigns no ownership
- Per-market NftRegistry validated at mint so a minted NFT is born transferable (#109)
- Burn/EmergencyBurn close the NFT, mint, ATA, and ExtraAccountMetaList PDAs and return rent to the holder; EmergencyBurn also recovers when the core has already closed the underlying portfolio (#131)
