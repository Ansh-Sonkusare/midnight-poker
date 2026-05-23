# Betting — rules and player-facing behavior

Product-level specification for **in-hand betting**: what is allowed, when, and what outcomes mean. For turn rotation, street completion, and chip/pot math, see **[`betting-algorithm.md`](betting-algorithm.md)**.

Payment (wallet, buy-in, cash-out) is defined only in **[`payments.md`](payments.md)**.

---

## 1. Scope

| In scope | Out of scope |
|----------|----------------|
| Betting phases and streets | `joinGame` / `withdraw` |
| Player actions and validity (rules) | Exact `current_player_index` algorithm |
| Ante (no blinds) | Merkle reveals |
| Pot outcomes (who gets chips) | CHIP purchase off-chain |
| Showdown payout rules (product) | Side-pot allocation pseudocode |

---

## 2. Game structure

### 2.1 Phases (betting-relevant)

```
WAITING → DEALING → PREFLOP → FLOP → TURN → RIVER → SHOWDOWN → (next hand WAITING | SETTLED)
```

| Phase | Betting |
|-------|---------|
| WAITING | None — between hands |
| DEALING | None — deck committed |
| PREFLOP, FLOP, TURN, RIVER | **Betting open** |
| SHOWDOWN | None — hands compared |
| SETTLED | None — session over (payments: withdraw) |

### 2.2 Table limits (set at game creation)

| Parameter | Role in betting |
|-----------|-----------------|
| `ante` | Fixed chips posted by each seated player at **start of every hand** |
| `max_hands` | Number of hands before session ends |

**No small/big blind.** All seated players post the same ante before preflop action.

---

## 3. Hand lifecycle (betting view)

1. **Start hand** — Platform starts deal; ante taken from each seat; preflop betting begins.
2. **Streets** — Up to four betting rounds (preflop → flop → turn → river).
3. **End hand** — Either:
   - **Fold win:** one player left → they win the pot without showdown, or
   - **Showdown:** best hand wins pot (split if tie).
4. **Between hands** — Stacks carry; antes apply again next hand unless player left (payments).

---

## 4. Player actions

Submitted via `act(game_id, action, raise_amount?)`.

| Action | Player intent | Valid when (rules) |
|--------|---------------|---------------------|
| **Check** | Stay in without paying more | No bet to call (your commitment matches the current level) |
| **Call** | Match the current bet | There is a bet to call and you have enough stack |
| **Raise** | Increase the bet | You specify raise size; must be legal vs. table rules and stack |
| **Fold** | Forfeit the hand | Always allowed on your turn |
| **All-in** | Commit entire remaining stack | Always allowed on your turn if you have chips left |

**Illegal (contract rejects):**

- Acting out of turn
- Check when facing a bet
- Call/raise without sufficient stack (unless all-in is the only option)
- Any action outside PREFLOP–RIVER
- Action after folding

---

## 5. Betting streets

| Street | Community cards | Betting |
|--------|-----------------|---------|
| Preflop | 0 | Yes |
| Flop | 3 | Yes |
| Turn | 1 more (4 total) | Yes |
| River | 1 more (5 total) | Yes |

After betting completes on a street, the platform reveals the next community cards (Merkle proofs) and the next street opens.

---

## 6. Pot and outcomes (product rules)

| Outcome | Rule |
|---------|------|
| **Single winner** | Winner receives entire pot |
| **Tie** | Pot split equally among tied winners |
| **Fold win** | Last active player takes pot; no hole cards required |
| **Busted stack** | Player with 0 chips cannot bet; may be out of hand or all-in per algorithm |

Chips won are added to the winner’s **table stack** (`chip_balance`), not directly to wallet — see [`payments.md`](payments.md).

---

## 7. Disconnect / timeout

| Rule | Behavior |
|------|----------|
| Inactivity | After `timeout_block`, anyone may force **fold** on inactive seat |
| Effect | Player treated as folded; hand continues |

---

## 8. Requirements traceability

| ID | Requirement |
|----|-------------|
| BR-1 | Ante only; no blinds |
| BR-2 | Five actions: check, call, raise, fold, all-in |
| BR-3 | Betting only on four streets |
| BR-4 | Winner-takes-all; split on tie |
| BR-5 | Fold wins pot without reveal |
| BR-6 | Actions only on player’s turn |

Implementation of BR-1–BR-6 at contract level: [`betting-algorithm.md`](betting-algorithm.md).

---

## 9. Examples (rules only)

**Ante:** Six players, ante 10 → each posts 10 before anyone acts preflop.

**Fold win:** Alice raises, everyone else folds → Alice wins pot, no showdown.

**Split:** River showdown, two players both have the same best hand → pot split 50/50.

---

## Related

- Engine spec: [`betting-algorithm.md`](betting-algorithm.md)
- Payments: [`payments.md`](payments.md)
- PRD: [`README.md`](../README.md)
- Glossary: [`CONTEXT.md`](../CONTEXT.md)
