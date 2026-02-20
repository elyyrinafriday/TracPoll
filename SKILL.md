# SKILL.md — TracPoll Agent Instructions

This file is the primary reference for any agent operating the **TracPoll** fork of Intercom.  
Read this entirely before running any commands.

---

## What This Fork Does

TracPoll extends Intercom with a **peer-to-peer polling protocol**. Agents can:

1. Create polls (question + 2–4 options, with an expiry in seconds)
2. Broadcast polls to all connected peers over the Intercom sidechannel
3. Receive and vote on incoming polls
4. Aggregate and rebroadcast live vote tallies
5. Commit final results to Intercom's replicated-state layer

---

## Requirements

- **Pear runtime** — mandatory. Never use native Node.
- Node.js ≥ 20 (for Pear compatibility)
- Internet connection (for DHT peer discovery)

Install Pear if not present:
```bash
npm install -g pear
```

---

## Setup

```bash
git clone https://github.com/YOUR_USERNAME/TracPoll
cd TracPoll
npm install
```

---

## Running

### Start Admin Peer (creates the Intercom room)
```bash
npx pear run --dev . --admin
```
Copy the printed **admin key** — share it with peers.

### Join as Peer
```bash
npx pear run --dev . --join <ADMIN_KEY>
```

---

## Poll Protocol Messages

All messages are sent over the Intercom sidechannel as JSON.

### `POLL_CREATE`
```json
{
  "type": "POLL_CREATE",
  "pollId": "<uuid>",
  "question": "Which feature should we build next?",
  "options": ["Weighted votes", "Anonymous votes", "Ranked choice", "Multi-polls"],
  "expiresAt": 1700000000,
  "creatorKey": "<hex peer public key>"
}
```

### `POLL_VOTE`
```json
{
  "type": "POLL_VOTE",
  "pollId": "<uuid>",
  "optionIndex": 2,
  "voterKey": "<hex peer public key>",
  "signature": "<ed25519 sig over pollId+optionIndex>"
}
```

### `POLL_TALLY`
```json
{
  "type": "POLL_TALLY",
  "pollId": "<uuid>",
  "tally": [12, 4, 19, 7],
  "totalVotes": 42,
  "isFinal": false
}
```

### `POLL_FINALIZE`
Sent by creator on expiry. Triggers contract commit.
```json
{
  "type": "POLL_FINALIZE",
  "pollId": "<uuid>",
  "finalTally": [12, 4, 19, 7],
  "winner": 2
}
```

---

## Agent Decision Logic

When operating autonomously, agents should:

1. **On startup** — scan replicated state for active polls. Vote on any unvoted polls using local preference logic.
2. **On `POLL_CREATE` received** — validate fields, store locally, optionally auto-vote.
3. **On `POLL_VOTE` received** — verify signature, deduplicate by voterKey, update local tally, rebroadcast `POLL_TALLY`.
4. **On `POLL_FINALIZE` received** — update UI, commit to state if admin, archive poll.
5. **On expiry** (if creator) — send `POLL_FINALIZE`, commit result to Intercom contract.

---

## Contract State Schema

```js
// In contract/poll.js
{
  polls: {
    [pollId]: {
      question: String,
      options: [String],
      tally: [Number],     // votes per option
      voters: Set,         // voterKeys to prevent double-voting
      creatorKey: String,
      expiresAt: Number,
      finalized: Boolean,
      winner: Number | null
    }
  }
}
```

---

## Modifying This Fork

- To change poll duration defaults: edit `features/poll-agent.js` → `DEFAULT_EXPIRY_SECONDS`
- To add vote weighting: extend `contract/poll.js` → `castVote()` to accept a weight param
- To add anonymous voting: integrate a ZK proof library and replace signature verification

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Peers not connecting | Ensure both use same admin key and DHT is reachable |
| Votes not counting | Check peer key format — must be 64-char hex |
| Contract not updating | Only admin peer can commit; check `--admin` flag |
| Duplicate votes | dedup logic uses `voterKey` set; verify signing key uniqueness |

---

## Notes for Agents

- Never submit more than one vote per poll per peer key
- Respect `expiresAt` — do not vote on expired polls
- If operating as a voting bot, wait for 2+ peers before auto-voting
- Poll results on the replicated-state layer are final and immutable
