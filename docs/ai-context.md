# Midnight Poker — AI Development Context

Use this as the source of truth for implementing the Compact contract, TypeScript SDK integration, and dApp.

---

## Contract Overview

Single Compact contract managing multiple Texas Hold'em games. Deck committed via Merkle tree. Platform generates deck + ZK shuffle proof per hand. Cards revealed progressively via Merkle proofs.

**Network**: Midnight testnet → mainnet
**Language**: Compact (TypeScript-based DSL for Midnight)

---

## State Machine

```
GamePhases:
  WAITING → DEALING → PREFLOP → FLOP → TURN → RIVER → SHOWDOWN → SETTLED

Per-game state:

game_id -> {
  phase: GamePhases,
  seats: [Player | null; 6],       // fixed array, null = empty
  pot: Uint<64>,
  hand_number: Uint<16>,
  max_hands: Uint<16>,
  ante: Uint<64>,
  current_player_index: Uint<8>,
  last_raiser_index: Uint<8>,
  community_cards_revealed: Uint<8>, // count of community cards revealed (0/3/4/5)
  deck_root: Field,                  // Merkle root for current hand
  hand_in_progress: bool,
  timeout_block: Uint<32>,          // block after which AFK player auto-folds
  min_buy_in: Uint<64>,
  max_buy_in: Uint<64>,
  per_hand_fee: Uint<64>,
}
```

```
Player state (per seat):
  address: Address,
  chip_balance: Uint<64>,
  hand_commitment: Uint<64>,      // chips committed for current hand
  folded: bool,
  last_action: Action | null,
```

---

## Public Ledger State

```compact
ledger {
  // Contract-wide
  owner: Address,                     // platform address
  token_id: TokenId,                  // custom chip token

  // Game state — single array of N games
  games: GameState[256],              // max concurrent games

  // Player registry
  player_to_game: Map<Address, game_id>,
}
```

---

## Private State (per player per game)

```compact
// Stored in the player's local proof server
private {
  hole_cards: (Field, Field),              // encrypted card value + blinding factor for 2 hole cards
  blinding_factors: [Field; 52],           // full array of blinding factors for current deck
  deck: [Field; 52],                       // full deck ordering (platform only, encrypted for player verification)
}
```

---

## Witness Functions

These run in the ZK circuit. Proving keys are generated at deploy time and distributed with the dApp.

| Witness | Inputs | Outputs | Purpose |
|---|---|---|---|
| `encrypt_hole_cards` | player_pubkey, card_value, blind | ciphertext | Encrypt hole card to player's pubkey |
| `decrypt_hole_cards` | ciphertext | (card_value, blind) | Player decrypts their hole cards locally |
| `reveal_card` | card_value, blind, merkle_proof | verified_reveal | Prove a reveal matches the stored Merkle root |
| `verify_shuffle` | deck[52], merkle_root | valid | Prove the 52 cards form a valid shuffled deck matching root |
| `verify_hand` | hole_cards[2], community_cards[5] | (best_hand_rank, winner) | Evaluate best 5-card hand from 7 cards |

---

## Action Flow

### 1. Create Game
```
Platform → contract: createGame(max_hands, ante, min_buy_in, max_buy_in, per_hand_fee)
```
- Creates a new game entry in WAITING phase
- Assigns `game_id`

### 2. Join Game
```
Player → contract: joinGame(game_id, buy_in_amount)
```
- Transfers chip tokens from player to contract
- Fills first empty seat
- If N ≥ 2 players seated, any player can call `startHand()`

### 3. Start Hand
```
Platform → contract: startHand(game_id, deck_root, shuffle_proof)
```
- Requires: game in WAITING phase, 2+ players
- Platform submits Merkle root + ZK shuffle proof
- For each player: platform calls `encrypt_hole_cards` for their 2 hole cards
- Phase → PREFLOP, ante deducted from each player
- `current_player_index` set to seat 0 (UTG)

### 4. Player Action
```
Player → contract: act(game_id, action, raise_amount?)
```
- `action`: Check | Call | Raise(amount) | Fold | AllIn
- Contract validates: correct phase, your turn, enough chips
- Updates pot, advances `current_player_index`
- If last raiser acted on, advance phase:

### 5. Advance Phase
Automatically triggered after last action in round:
- PREFLOP → FLOP: reveal 3 community cards (platform submits Merkle proofs)
- FLOP → TURN: reveal 1 community card
- TURN → RIVER: reveal 1 community card
- RIVER → SHOWDOWN: evaluate winner

### 6. Showdown
```
Platform → contract: evaluateShowdown(game_id, player_reveals[])
```
- Platform submits hole card reveals for all remaining players
- Contract verifies each reveal against Merkle root
- Contract evaluates hands via `verify_hand` witness
- Payout: winner gets pot (split on ties)
- Deduct per-hand fee from each player's stack
- If `hand_number == max_hands` → SETTLED. Players withdraw.
- Else → return to WAITING for next hand

### 7. Withdraw
```
Player → contract: withdraw(game_id)
```
- Only allowed when game phase == SETTLED
- Transfers remaining `chip_balance` back to player

### 8. Leave Game
```
Player → contract: leaveGame(game_id)
```
- Allowed between hands (WAITING phase)
- Returns their chip_balance

---

## Timeout Handling

Contract tracks `timeout_block = current_block + timeout_duration` at each action window. After that block:
- A call to `foldInactive(game_id, seat_index)` can be made by anyone
- The inactive player is folded
- Game continues

---

## Merkle Tree Structure

```
Depth 6 binary tree for 52 cards:

Level 0: MerkleRoot (32 bytes) ← stored in ledger
Level 1: H(0-25), H(26-51)          // 2 internal nodes
Level 2: H(0-12), H(13-25), ...     // 4 internal nodes
...
Level 6: leaf0..leaf51               // 52 leaves

Each leaf = PoseidonHash(card_value || blinding_factor)

Merkle proof = 6 sibling hashes for any card
Hash function = Poseidon (ZK-friendly, not SHA-256)
```

---

## Chip Token

Custom Midnight fungible token (built using Compact's token standard):
- **Name**: "Poker Chip"
- **Symbol**: "CHIP"
- **Decimals**: 0 (whole chips only)
- **Minting**: Platform mints initial supply
- **Transfer**: Players buy CHIPs from platform (off-chain payment), platform sends CHIPs to player's address on-chain

---

## Platform Responsibilities

| Responsibility | Where |
|---|---|
| Generate deck + Merkle root + shuffle proof per hand | Off-chain (platform server) |
| Encrypt hole cards to each player's pubkey | Off-chain → submit ciphertexts |
| Reveal community cards at each phase | On-chain (Merkle proof submission) |
| Submit winner evaluation at showdown | On-chain (witness proof) |
| Distribute proving keys with dApp | Off-chain (dApp bundle) |
| Cover proving costs | Off-chain (platform pays compute) |

---

## Verification Checklist (Post-Game)

Anyone can verify after game ends:
1. All 52 cards revealed with Merkle proofs
2. Re-hash leaves → check against each hand's Merkle root
3. Verify each reveal happened at the correct phase/position
4. Replay hand evaluations using public community cards + hole cards
5. Confirm payouts match hand results

---

## Compact Implementation Notes

- Use `Poseidon` hash for Merkle tree (Compact stdlib)
- Hand evaluation circuit: rank lookup table for all 10 hand types + kicker comparison
- 6 players × 21 hand combos = 126 comparisons max per showdown
- Proving key size estimation: 100-300 MB per witness function
- Use `ledger` for contract state, `private` for per-user encrypted data
- Token standard from Compact stdlib for CHIP token
- See `midnight://syntax/latest` MCP resource for current Compact syntax
- See `midnight://examples/token` MCP resource for token contract pattern
