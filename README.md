# Club Treasury – VibeChain × MERAZ 7.0 (Round 1)

A decentralized treasury for a club, where **members decide by vote, the contract enforces the rules, and everything is publicly verifiable**. No admin, no owner, and the deployer has no special rights.

| | |
|---|---|
| **Network** | Ethereum Sepolia |
| **Contract address** | `<PASTE ADDRESS AFTER DEPLOY>` ([Etherscan](https://sepolia.etherscan.io/address/<ADDRESS>)) |
| **Live demo** | `<PASTE VERCEL URL>` |
| **Dev environment** | Antigravity |

## 1. Problem we chose

Club money usually sits in one or two people's bank/UPI accounts. Decisions happen in private chats, so months later nobody can say who approved what or where the money went. We built a treasury that answers four questions a judge (or any member) can check on-chain:

| Question | Answer |
|---|---|
| Who may do what? | **Anyone** can contribute and view. **Members** can propose, vote and leave. **Nobody** has admin powers. |
| How is a money decision made? | A member proposes a spend with a stated purpose; members vote; more than half of the members must approve; then anyone can execute it once. |
| Why can't one person misuse funds (even the deployer)? | The contract has no owner/admin/upgrade function. The deployer address isn't stored. Funds move only via an approved proposal, with a minimum of 3 members and a per-spend cap. |
| What can a member verify afterwards? | Every proposal, vote, purpose and payment is on-chain (events and public state) and shown on the dashboard, which works with **no wallet** (read-only). |

## 2. How it works

- **Treasury**: holds Sepolia ETH. Anyone can send funds via `contribute()` or a plain transfer; contributions are tracked per address.
- **Membership**: ≥3 founders are fixed at deployment. After that members change **only by vote** (`AddMember` / `RemoveMember`) or by `leave()`. Membership can never drop below 3.
- **Decisions**: one member = one vote. A proposal passes when YES > 50% of the members **at the time it was created** (snapshot). Members who joined later cannot vote on older proposals (prevents vote-stuffing).
- **Lifecycle**: `Active → Passed → Executed`, or `Rejected / Expired / Cancelled`. Voting window is set at deploy (default 24h). A passed proposal must be executed within 2 days after voting ends.
- **Accountability**: every proposal stores its purpose on-chain; every payment emits `Spent(proposalId, to, amount, purpose)`.

### Safety invariants (all covered by tests)
1. **Never pays more than it holds**: balance is checked at execution, plus a cap per spend (default 50% of the treasury).
2. **No authorisation moves funds more than it was granted for**: the approved amount is fixed in the proposal; `executed` is set before the transfer, so it can run once.
3. **Rejected, expired, cancelled or unauthorised actions never move funds**: `execute` only works in the `Passed` state.

## 3. Architecture

```
MetaMask ──► frontend (ethers v6, static) ──► ClubTreasury.sol (Sepolia)
                 ▲ read-only dashboard via public RPC (no wallet needed)
```
- `contracts/ClubTreasury.sol` – the whole system (≈300 lines, 1 dependency: OpenZeppelin `ReentrancyGuard`).
- `frontend/` – dependency-free static site (HTML/CSS/JS + pinned ethers).
- `test/ClubTreasury.test.js` – unit and attack tests; `contracts/mocks/` – test-only attackers.
- `docs/DESIGN.md` – design justification and threat model.

## 4. Setup

```bash
npm install
cp .env.example .env     # throwaway Sepolia wallet; set FOUNDERS (3+ addresses)
npm test                 # all tests must pass
npm run deploy:sepolia   # prints address, writes frontend/config.js + deployment.json
npm run serve            # http://localhost:5173
```

**Live demo (Vercel):** `cd frontend && npx vercel --prod` (static site, no build step). Commit the updated `frontend/config.js` first.

**Demoing alone?** Add 3 founder accounts to MetaMask (or use 3 teammates); 2 of 3 approve a proposal.

## 5. Design choices and justification

| Choice | Why |
|---|---|
| No owner/admin; deployer not recorded | Meets "no backdoors": the deployer has exactly the same power as any stranger |
| ≥3 founders, never fewer than 3 members | With 1–2 members, one person *is* the club; a minimum of 3 keeps "majority" meaningful |
| Strict majority of a **snapshot** | Fixed target per proposal; later joins or leaves can't swing an open vote |
| Join/leave rights recorded per proposal id | A newly added member can't vote on proposals created earlier |
| Per-spend cap (% of treasury) | Even a colluding majority can't empty the treasury in one transaction |
| Purpose string stored on-chain | Each outflow is tied to a reason and to the decision that authorised it |
| Anyone can execute | Liveness: a passed decision doesn't depend on one person showing up |
| Expiry (voting + 2 days grace) | Old approvals can't be used months later |
| Direct payout with `nonReentrant` + state-before-call | Simple and auditable; a failing recipient only reverts its own execution and funds stay safe |
| Read-only dashboard through public RPC | Outsiders can verify without installing a wallet |

See `docs/DESIGN.md` for the threat model and limitations.
