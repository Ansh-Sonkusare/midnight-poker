# Midnight Poker — Domain Glossary

## Game

- **Poker variant**: Texas Hold'em
- **Table model**: Single contract manages multiple simultaneous games, keyed by game_id
- **Max players per table**: 6
- **Game start**: Auto-starts when N players seated (2-6). Platform submits initial deck.
- **Game end**: Fixed number of hands. Settles after final hand payout.
- **Chip model**: Custom Midnight fungible token for buy-in and payouts
- **Buy-in**: Variable within min/max per table. Chips committed before each hand.
- **Ante**: Equal ante from all seated players per hand. No blinds.
- **Actions**: Standard Hold'em (check, call, raise, fold, all-in)
- **Payout**: Winner-takes-all pot. Split pot on tied hands.
- **Disconnect**: Auto-fold after block-based timeout.
- **Platform fee**: Per-hand fee deducted from each player's stack.

## Deck & Cards

- **Deck scheme**: Merkle tree commitment. 52 leaf hashes hash(card_value || blinding_factor) → single 32-byte Merkle root stored on-chain per hand.
- **Deck generation**: Trusted setup ceremony. Platform generates deck + ZK proof of valid shuffle.
- **Shuffle proof**: ZK Snark proving exactly 52 distinct cards in a valid permutation. Published on-chain per hand.
- **Card reveal (progressive)**: Community cards are unsealed round-by-round by submitting card_value + blinding_factor + Merkle proof (sibling hashes along the path to root). The contract verifies the proof against the stored Merkle root. Hole cards encrypted to the player's public key using Midnight encryption gadgets. Decryption happens in the player's wallet/proof server via platform UI.
- **Card reveal (end-game)**: All 52 cards revealed with their Merkle proofs → anyone can replay every hand and independently verify winners.
- **Fairness guarantee**: Contract cannot modify deck mid-game (would break ZK proof). All reveals verified against Merkle root. Post-game anyone can replay.

## Midnight-Specific

- **Public state (ledger)**: Merkle root per hand (32 bytes), game state machine (player seats, game phase, pot per hand, revealed community cards per hand).
- **Private state**: Hole cards (encrypted to recipient), per-player blinding factors.
- **Witness functions**: `encrypt_hole_cards`, `decrypt_hole_cards`, `reveal_card`, `verify_shuffle`
- **Circuit**: Fixed at deploy. Supports 6 max seats. Proving keys generated at deploy time.
- **Proving keys**: Distributed with dApp or downloaded on-demand.
- **UI**: Platform's frontend runs on the player's machine, triggers wallet/proof server for decryption and transaction signing.

## Trust Model

- **Platform**: Generates deck each hand, knows all cards during play. Constrained by ZK proofs — cannot modify deck mid-hand, cannot lie about reveals.
- **Players**: See only their hole cards + community cards. Verify fairness post-game.
- **Verification**: Post-game, anyone can open all commitments and replay every hand on-chain. Contract code is public.
