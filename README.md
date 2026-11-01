# x402-agent-shopper - an autonomous x402 buyer that shops three strangers' APIs

An autonomous agent is given one goal —  *"what time is it in Mumbai, and is the seller safe to buy from?"* — and it has to **find** sellers, **check** them, **pay** them and come back with a **verifiable result**, without a human in the loop and without knowing any of them in advance.

Two of the three hops belong to servers we do not own. That is the point.

<img width="1260" height="678" alt="01-shopper" src="https://github.com/user-attachments/assets/c1b9caa4-34a2-42b2-84c1-95d03a7f4dc4" />
<img width="1260" height="678" alt="04-shopper-dry" src="https://github.com/user-attachments/assets/0074ae45-80ed-43fd-be6a-b40fa62be6d5" />

```
01 DISCOVER  8 candidates for "timezone"                       bazaar
   └ geospatial.openverbs.com $0.0020  ·  agentsvc.io $0.0020  ·  2s.io $0.0025
02 SELECT    geospatial.openverbs.com  Resolve the timezone… $0.0020
03 PAY       geospatial.openverbs.com · $0.0020 · settled      230ms
   └ 402 header · x402 v2 · 1 offer(s) · also MPP rail "evm"
04 RESULT    Asia/Kolkata +05:30
05 DISCOVER  trust checks for geospatial.openverbs.com         6 found
06 SELECT    trust-agent.io                                    $0.0050
07 PAY       trust-agent.io · $0.0050 · settled                412ms
08 RESULT    verdict: clean · 0.91 · 4 signals
09 CHECK     ERC-8004 identity #1 on base-sepolia              0x21fd…e235
10 PAY       our seller · $0.050 · settled                     18ms
11 RESULT    certificate 9c1f… · buyer 0x21fd…e235
12 ATTEMPT   agent402.tools wants $0.030 for one image         over policy
13 REFUSE    price above per-call cap · 30000 > 20000          stopped
TOTAL SPENT  $0.057   ·  3 sellers touched  ·  2.6s
```

The whole errand costs about five cents. That total is the punchline: the money is not the hard part any more, the *contract* is.

> **The name.** `x402-agent-shopper` is the buyer half of a pair. Its sibling is [`x402-trust-merchant`](../x402-trust-merchant) — the shop that sells the agent a trust check and a signed certificate. Same errand, opposite side of the counter.

---

## The claim this repo makes

> **MCP is the handshake. x402 is the money. Neither tells the agent what it just bought.**

An agent can discover a tool, read a price, sign a payment and receive bytes — and still not know whether those bytes are the thing it asked for. This project is the smallest honest program that demonstrates all four states: discovery, payment, settlement evidence and
result verification, including what happens when they disagree.

---

## Quickstart

```bash
git clone <this repo> && cd x402-agent-shopper
npm install                 # one dependency: viem

node run.mjs --fixtures     # the whole tape, no network, no money
node run.mjs --dry          # real listings, real 402s, real signature, no money
npm run seller &            # our own storefront on :8787
node run.mjs --dry          # now hop 3 has something to buy
PRIVATE_KEY=0x… node run.mjs --live   # the demo — capped at $0.50 by default
```

Node 22+. Nothing else. No framework, no database, no build step.

### Modes, and why there are three

| Mode | Network | Money | Use it for |
|---|---|---|---|
| `--fixtures` | none | none | the stage, when venue Wi-Fi dies mid-sentence |
| `--dry` | real | none | rehearsal — proves the sellers still look like this today |
| `--live` | real | real, capped | the actual demo |

`--fixtures` is not a mock. It replays JSON captured from live endpoints, and
`test/offline.mjs` proves it cannot reach the network: it replaces `globalThis.fetch`
with a landmine that throws. If a fixture ever needs the network, that test fails loudly.

---

## What happens on each hop

| Hop | Who | What is bought | How it is known to be real |
|---|---|---|---|
| 1 | `geospatial.openverbs.com` — a stranger | timezone for 19.076, 72.8777 | `402` → EIP-3009 signature → `200` + settlement header |
| 2 | `trust-agent.io` — a second stranger | a risk verdict on hop 1's seller | same, plus the verdict carries its own evidence list |
| 3 | **our** seller on `:8787` | an ed25519-signed certificate | signature verifies offline, with no call back to the seller |
| — | `erc-8004` registry (testnet) | nothing — a read | `ownerOf` and `tokenURI` on a real contract |

Hop 3 exists on purpose: the demo has to show that *we* can be the shop too, or it is
a story about shopping rather than a story about building.

---

## Architecture

```
                        ┌──────────────────────────────┐
                        │           run.mjs            │
                        │  goal → find → check → pay   │
                        │      → verify → receipt      │
                        └───────┬──────────────┬───────┘
                                │              │
             ┌──────────────────┘              └─────────────────┐
             ▼                                                   ▼
   ┌───────────────────┐                              ┌────────────────────┐
   │ lib/discovery.mjs │  Bazaar index (HTTP)         │  lib/guard.mjs     │
   │ lib/registry.mjs  │  ERC-8004 (JSON-RPC)         │  Budget.approve()  │
   └─────────┬─────────┘                              │  ./STOP kill switch│
             │                                        └─────────┬──────────┘
             ▼                                                  │ every signature
   ┌────────────────────────────────────────────┐               │ goes through here
   │             lib/x402.mjs                   │◄──────────────┘
   │  readChallenge → chooseAccept → sign       │
   │  X-PAYMENT / PAYMENT-SIGNATURE / MPP       │
   └───────┬──────────────────────┬─────────────┘
           │                      │
           ▼                      ▼
  ┌────────────────┐    ┌──────────────────────────────┐
  │ 3rd-party 402  │    │ seller/server.mjs (:8787)    │
  │ timezone       │    │ /llms.txt                    │
  │ trust verdict  │    │ /.well-known/x402            │
  └────────────────┘    │ /.well-known/schema.json     │
                        │ /buy/certificate  (50000 μ$) │
                        └──────────┬───────────────────┘
                                   ▼
                        ┌──────────────────────────────┐
                        │ seller/certificate.mjs       │
                        │ ed25519 sign · verify offline│
                        └──────────────────────────────┘
```

Everything on the left is somebody else's server. Everything on the right is what a builder has to publish for an agent to be able to buy from them at all.

---

## The parts worth reading

### 1. The budget is a gate, not a log line

`lib/guard.mjs` is 54 lines and it is the only thing standing between an autonomous agent and an unbounded bill. It runs *before* the signature, not after the receipt.

```js
approve({ url, amount, network, payTo }) {
  if (existsSync(this.limits.killSwitch)) {
    throw new Refused('kill switch present', `remove ${this.limits.killSwitch} to resume`);
  }
  if (!this.limits.networks.includes(network)) {
    throw new Refused('network not allowed', `${network} is not in the allowlist`);
  }
  if (BigInt(amount) > this.limits.maxPerCall) {
    throw new Refused('price above per-call cap', `${amount} > ${this.limits.maxPerCall}`);
  }
  if (this.spent + BigInt(amount) > this.limits.maxTotal) {
    throw new Refused('session budget exhausted', `${this.spent + BigInt(amount)} > ${this.limits.maxTotal}`);
  }
  this.spent += BigInt(amount);
  return true;
}
```

`Refused` carries `.reason` and `.detail`, so the refusal is a *sentence a human can read on a projector*, not a stack trace:

```
12 ATTEMPT   agent402.tools wants $0.030 for one image   over policy
13 REFUSE    price above per-call cap · 30000 > 20000    stopped
```

The kill switch is a file. `touch STOP` and every future purchase refuses, with no
restart and no code change. On a stage, that is worth more than a config flag.

### 2. The payment header is not standard yet

This is the finding that made the whole project worth building. Real sellers disagree
about where the price lives, how the network is named, and — worst — what the credential
header is called:

- x402 v1 → `X-PAYMENT`
- x402 v2 → `PAYMENT-SIGNATURE`
- MPP → `Authorization: Payment …`

One live seller answers a dual-header request with
`400 ambiguous_payment: provide either x402 (PAYMENT-SIGNATURE) or MPP, not both.`
So the client negotiates instead of guessing:

```js
const attempts = [
  { 'payment-signature': header, 'x-payment': header },
  { 'payment-signature': header },
];
if (challenge.mpp) attempts.push({ authorization: `Payment ${header}` });

let second = null;
for (const credentials of attempts) {
  second = await send(credentials);
  if (second.status !== 400) break;
  const probe = await second.clone().text();
  if (!/ambiguous/i.test(probe)) break;
}
```

Three lines of `for` loop that decide whether the demo works on the seller you meet
five minutes before going on stage.

### 3. The input shape is not in the price

A `402` tells you what something costs. It does not tell you what the tool wants as input. This seller's Bazaar record declares a JSON body `{lat, lon}` on `POST`; the naive `GET ?lat=&lon=` buys a validation error after the money moves. The client posts
what the listing declares:

```js
const r = await pay(buyUrl.toString(), {
  mode: 'live',
  account,
  budget,
  network: NETWORK,
  method: 'POST',
  body: { lat: DEMO.commodity.lat, lon: DEMO.commodity.lon },
});
```

### 4. A seller that accepts money twice is a bug

`usedNonces` in `seller/server.mjs` makes an authorisation single-use. Without it, the
same signed payload replays and the seller hands out two results for one payment. The
contract test asserts it:

```js
const replay = await post('/buy/certificate', body, { 'payment-signature': header });
assert(replay.status === 402, 'replay is refused');
assert(/already used/i.test(replay.body.error), 'replay says why');
```

### 5. The receipt is the product

`lib/tape.mjs` renders every phase as a line with a price, a duration and a verdict.
It is deliberately dumb: no spinner, no colour that carries meaning, no hidden state.
If it is not on the tape, it did not happen.

### 6. The result is verifiable offline

`seller/certificate.mjs` mints a canonical JSON document, hashes it, signs the hash with
an ed25519 key, and gives the buyer both. `check()` verifies with the public key only —
no callback, no registry, no trust in the seller's availability:

```js
const { certificate, publicKey } = mint({ buyer: account.address, item, result });
const verification = check(certificate, publicKey);   // → { valid: true, digest }
```

A buyer that has to ask the seller "is this yours?" has bought nothing.

---

## Technology

| Layer | Choice | Why |
|---|---|---|
| Runtime | Node 22, ESM | no build step; `node run.mjs` is the whole deploy |
| Chain access | `viem` ^2.21 | typed EIP-712 signing, one dependency |
| Money | x402 over USDC on Base (`eip155:8453`) | sub-cent prices, no account, no invoice |
| Signature | EIP-3009 `TransferWithAuthorization` | the buyer signs, the facilitator pays gas |
| Seller | `node:http`, zero deps | readable in a projector font |
| Result signing | `node:crypto` ed25519 | verify offline, ~20 lines |
| Identity | ERC-8004 (read-only, testnet) | a real registry, not a mock |
| Discovery | x402 Bazaar HTTP index | public, no key, live sellers |
| Tests | hand-rolled asserts | one command, no runner config |

No TypeScript, no framework, no ORM, no Docker. Every file fits on one screen because
the audience has to be able to read it from the back of the room.

---

## Tests

```bash
node test/e2e.mjs        # 12 checks — spawns a seller on :8899, ephemeral wallet, full loop
node test/contract.mjs   # 27 checks — the refusal matrix on :8898
node test/offline.mjs    #  3 checks — fixtures mode with fetch replaced by a landmine
```

What the refusal matrix covers, because "an agent meets failure states long before it
meets a 200":

- malformed / unsigned payment
- valid signature, replayed authorisation
- garbage credential
- underpayment
- wrong network
- kill switch present
- no wallet key in live mode
- unpaid request to a priced route

Each case asserts a status code **and** a machine-readable reason. A `402` with an empty
body is not an error message, it is a shrug.

---

## What the demo does not do

- **Settlement is verified, not broadcast.** The seller validates the EIP-3009 signature
  and returns a settlement receipt. Wiring a real facilitator is one HTTP POST —
  left out so the money path has no third party in it.
- **No testnet coins for attendees**, no attendee wallets, no airdrop.
- **The registry check proves identity, never quality.** A registered seller can still
  take your money and return garbage. That is a slide, not a demo.
- **No LLM in the loop.** The "agent" is a deterministic program. That is deliberate:
  the interesting part is the protocol, and a model would only add latency and doubt.

---

## Fork it and build on it

```bash
git clone <this repo> && cd x402-agent-shopper
npm install
node test/e2e.mjs          # green before you change anything
```

Conventions that keep this readable:

1. **One dependency, and it earns its place.** Adding a framework to a 1,400-line demo
   makes it undemoable.
2. **Every network boundary is a `lib/` module.** `discovery`, `registry`, `x402`, `fixtures`.
   Swap one without touching the tape.
3. **New failure state → new contract test first.** The `test/contract.mjs` matrix is the
   spec for the seller's behaviour.
4. **Fixtures are captures, never hand-written.** If you change a fixture by hand, add the
   `capturedAt` and the command that produced it.
5. **The tape is the UI.** New capability = new line with a price and a verdict; no prose.

### Good first contributions

- **Real facilitator settlement** — replace the local `verifyPayment` with a
  `POST /settle` to a facilitator, keep the signature check as the gate. `lib/facilitator.mjs`
  is the intended seam.
- **Refunds** — a seller that can refuse *after* payment needs a refund path; today the
  contract test only proves refusal *before* money moves.
- **Idempotency keys** — two retries of the same request should produce one certificate,
  not two. The nonce set is half of this.
- **MCP surface** — expose `find`, `check`, `pay` as MCP tools so a model can drive the
  same engine, with the same budget guard.
- **A2A agent card** — the seller already serves `/.well-known/agent-card.json`; make it
  useful (skills, prices, auth schemes) and let another agent negotiate from it.
- **Multi-rail** — support the MPP rail properly instead of only detecting it. Today the
  tape prints `also MPP rail "evm"` and walks away; that is honest, not complete.
- **Spend ledger** — append every `approve()` decision to a JSONL file so a buyer can
  audit an agent's month, not just its session.
- **Caching** — the timezone is the same for a minute; a cache with a price-aware
  freshness policy is a genuine economic feature, not an optimisation.

### Pull requests

Small, single-purpose, with the test output pasted in. If you change a network module,
paste the `--dry` tape before and after. If you add a failure state, paste the new
`test/contract.mjs` line. No changes that make the tape harder to read on a projector.

---

## Environment

Copy `.env.example` to `.env` (gitignored) or export directly:

```
PRIVATE_KEY=              # live mode only; keep it out of shell history
MAX_PER_CALL_MICROS=20000 # $0.02 per call
MAX_TOTAL_MICROS=500000   # $0.50 per session
ALLOW_NETWORKS=eip155:8453,eip155:84532
KILL_SWITCH_FILE=./STOP
SELLER_URL=http://localhost:8787
PORT=8787
PAY_TO=0x1111111111111111111111111111111111111111
SELLER_PRICE_MICROS=50000
```

Run the demo from a wallet that holds a few dollars of USDC on Base and nothing else.
A demo wallet should be disposable by construction.
