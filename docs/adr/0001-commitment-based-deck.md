# Use Merkle tree for deck commitment, not a seed-based scheme

A seed-based deck (one seed deterministically generates the full ordering) would let anyone who learns the seed see all cards. Even if the seed is only revealed to the platform post-game, collusion or key leaks break privacy mid-game.

Each card is hashed as `hash(card_value || blinding_factor)`. The 52 leaf hashes are combined into a single Merkle root (32 bytes) stored on the public ledger. Revealing a card means submitting the card data + Merkle proof (6 sibling hashes) — the contract verifies the reveal against the root.

**1.5 KB → 32 bytes** on-chain per hand vs. storing 52 individual commitments. Post-game all 52 cards are revealed with their Merkle proofs, and anyone can re-hash from leaves → root to verify the entire game.

Note: `hash` here refers to a ZK-friendly hash (e.g., Poseidon), not a general-purpose hash like SHA-256. Poseidon is orders of magnitude more efficient inside ZK circuits.
