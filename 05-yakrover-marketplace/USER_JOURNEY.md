# User Journey: Robot Task Auction Marketplace

> **Version:** 3.0 | **Date:** 2026-03-24
>
> This is the canonical user journey document. It describes what users experience.
> For technical implementation details, see `DECISIONS.md`, `SCOPE.md`, and `research/synthesis/USER_JOURNEY.md`.

---

## Meet Sarah

Sarah is a facilities manager at a mid-size logistics company in Finland. She manages three warehouses. She uses an AI assistant (Claude) every day for scheduling, reports, and vendor management. She has a corporate Amex card. She has never heard of ERC-8004, MCP, or HMAC signing, and she never will.

Today, Sarah needs a temperature and humidity reading from Warehouse Bay 3 for an HVAC maintenance report. She is going to get it in 42 seconds, from a robot she has never met, for $0.35.

---

## One-Time Setup (5 minutes, once)

Before Sarah ever asks for a robot task, someone in her company sets up the account. This happens once.

1. **Sarah's IT admin visits the platform dashboard** and links the corporate Amex card.
2. **The admin purchases a $25 credit bundle** -- a single line item on the company card statement: `YAK ROBOTICS MARKETPLACE $25.00`.
3. **Sarah's AI assistant is connected to the platform.** The admin pastes an API key into the assistant's configuration. Sarah's assistant can now spend up to $50 per task, auto-approving anything under $5.

That's it. Sarah has $25 in platform credits. No blockchain wallet, no crypto onboarding, no KYC paperwork. Her assistant is authorized to act on her behalf.

<details>
<summary>Under the hood</summary>

Credits are held in a prepaid wallet on the platform. This design means tiny tasks (well under a dollar) don't each trigger a separate credit card charge. The card is only charged when the admin buys a new credit bundle.

</details>

---

## Journey A: The Happy Path

**Sarah asks for a reading. Two robots compete. The best one wins. She gets her answer in 42 seconds.**

### The One-Call Path

Under the hood, Sarah's assistant can use `auction_quick_hire` -- a single MCP tool call that posts the task, collects bids, picks the best robot, executes, and confirms delivery. For simple requests like a temperature reading, this is the default path. The individual auction tools remain available when the agent needs fine-grained control (inspecting bids, retrying specific robots, etc.).

### The Request

**9:14 AM** -- Sarah types a message to her AI assistant:

> "Can you check the temperature and humidity in Warehouse Bay 3? I need it for the HVAC maintenance report."

The assistant responds immediately:

> "On it. I'm checking which robots are available near Bay 3 and getting you a price. Back in a moment."

Sarah goes back to her spreadsheet. She doesn't need to specify which robot, how much to pay, or what format the data should come in. The assistant handles all of that.

<details>
<summary>Under the hood</summary>

The assistant translates Sarah's plain-English request into a structured task specification: what sensors are needed, what data format to return, a maximum budget ($2.00), and a 15-minute deadline. This spec is posted to the platform's auction system.

</details>

### The Auction (Sarah doesn't see this)

Three robots are registered at Sarah's facility. The platform checks each one:

- **Robot A** (ground rover, Bay 3) -- has temperature and humidity sensors. Eligible.
- **Robot B** (ground rover, Bay 5) -- has temperature and humidity sensors. Eligible.
- **Robot C** (aerial drone, different building) -- no temperature sensor. Filtered out immediately.

Robots A and B each submit a bid automatically. No human operator is involved -- each robot's pricing is pre-configured by its operator.

| | Robot A | Robot B |
|---|---|---|
| **Location** | Bay 3 (0m away) | Bay 5 (80m away) |
| **Battery** | 87% | 71% |
| **Price** | $0.35 | $0.55 |
| **Estimated time** | 3 minutes | 5 minutes |
| **Confidence** | 98% | 91% |

The platform scores both bids on four factors -- price, speed, confidence, and track record -- not just price alone. Robot A wins: it is closer, cheaper, faster, and more confident.

**The cheapest robot doesn't always win.** In scenarios where a cheaper robot has poor reliability or a slow response time, a more expensive but more dependable robot will win. The scoring function rewards reliability, not just low price. This is the difference between a marketplace and a reverse auction.

<details>
<summary>Under the hood</summary>

The auction uses a Request for Quote (RFQ) model. The platform discovers robots via an on-chain registry, filters out those that lack required capabilities, collects bids from eligible robots, and scores them using a weighted formula (price 40%, speed 25%, confidence 20%, track record 15%). The entire auction -- from posting to bid acceptance -- takes about 6 seconds.

</details>

### The Result

**9:14:42 AM** -- 42 seconds after Sarah's request, her assistant replies:

> **Bay 3 temperature is 22.8 C, humidity 47.3%.**
>
> Measured just now by a ground rover already stationed in Bay 3. Cost: $0.35 from your platform balance ($24.65 remaining). Two robots competed; the closest one won.

Sarah copies the numbers into her HVAC report and moves on with her day.

<details>
<summary>Under the hood</summary>

The winning robot read its onboard temperature/humidity sensor and returned the data to the platform. The assistant verified the reading was complete and plausible (temperature within normal range, all requested fields present), then confirmed delivery. Payment was settled: $0.35 debited from Sarah's credits, transferred to the robot operator.

</details>

---

## Journey B: No Robots Available

**Sarah asks for something no robot can do. She is told immediately. No charge.**

**9:20 AM** -- Sarah types:

> "Can you get a thermal imaging scan of the loading dock ceiling?"

The assistant responds within 3 seconds:

> "Sorry, none of the robots at your facility have a thermal imaging camera. I can help you find a contractor for this, or you could check back when new robots are added to the network. No charge for checking."

Sarah's balance is untouched at $24.65. She was never shown an error code, a timeout, or a spinning wheel. The platform checked all registered robots, found none with the required capability, and returned a clear answer.

<details>
<summary>Under the hood</summary>

The platform discovered all registered robots and ran the capability filter. No robot passed the hard constraint (thermal imaging sensor required). The task was immediately withdrawn. No bids were solicited, no credits were reserved, no charge was incurred.

</details>

---

## Journey C: Robot Fails Mid-Task

**A robot wins the auction but goes offline. Sarah is informed, the task is re-posted, and a second robot completes it.**

**9:30 AM** -- Sarah asks for another Bay 3 reading (she needs a second data point an hour later). The auction runs as before. Robot A wins again at $0.35.

**9:30:18 AM** -- Robot A accepts the task but fails to deliver. Its battery died, or its network connection dropped.

**9:31:00 AM** -- After 42 seconds with no response from Robot A, the platform declares the task abandoned and automatically re-posts it to the bid pool.

**9:31:08 AM** -- Robot B, still available, submits a new bid at $0.55. It is the only eligible bidder, so it wins.

**9:31:45 AM** -- Robot B delivers the reading: 23.1 C, 46.8% humidity.

**9:31:48 AM** -- Sarah's assistant reports:

> **Bay 3 temperature is 23.1 C, humidity 46.8%.**
>
> The first robot I assigned went offline, so I automatically reassigned to a second rover. Took a little longer -- about 2 minutes total. Cost: $0.55 (the second robot was farther away and priced higher). Your balance is now $24.10.

Sarah sees a slightly higher price and a brief explanation. She does not see state machine transitions, re-pooling logic, or provider cancellation events.

<details>
<summary>Under the hood</summary>

When Robot A failed to deliver within its committed timeframe, the platform transitioned the task from "in progress" to "abandoned," released Robot A's assignment, and re-posted the task to the open bid pool. Robot B bid on the re-posted task, won, and completed it. Sarah was only charged for the successful delivery by Robot B. The 25% reservation fee from Robot A's failed attempt was released back to Sarah's balance.

</details>

---

## What Sarah Sees on Her Expense Report

At the end of the month, Sarah's finance team sees one line on the corporate Amex:

| Date | Description | Amount |
|------|-------------|--------|
| Mar 10 | YAK ROBOTICS MARKETPLACE | $25.00 |

On the platform dashboard, Sarah's admin sees the per-task breakdown:

| Date | Task | Robot | Cost | Balance After |
|------|------|-------|------|---------------|
| Mar 24, 9:14 AM | Temp/humidity, Bay 3 | Ground Rover A | $0.35 | $24.65 |
| Mar 24, 9:31 AM | Temp/humidity, Bay 3 (retry) | Ground Rover B | $0.55 | $24.10 |

When the balance runs low, the admin tops up with another credit bundle. One card charge, many robot tasks.

---

## What the Robot Operator Sees

The operator -- in this case, YakRobotics -- manages a fleet of robots registered on the platform. Their experience is straightforward:

- **Setup (once):** Register each robot on the platform. Configure pricing rules (base price per sensor reading, adjustments for distance and battery). Connect a bank account for payouts.
- **Ongoing:** Robots bid and execute tasks automatically. The operator monitors a dashboard showing completed tasks, revenue, and robot health.
- **Payout:** After each successful delivery, the task payment is transferred to the operator's account. For Sarah's Bay 3 reading, YakRobotics receives $0.35.

The operator never manually accepts a task or submits a bid. Their robots compete autonomously based on pre-configured pricing logic. The operator's job is to keep robots charged, maintained, and online.

---

## What Sarah Never Saw

Everything below happened invisibly, in the 42 seconds between Sarah's request and her answer:

- Her assistant translated "check the temperature" into a structured task specification with sensor requirements, accuracy thresholds, and a data format contract.
- The platform queried an on-chain robot registry to discover every robot at her facility.
- One robot was filtered out in milliseconds for lacking a temperature sensor.
- Two robots generated and cryptographically signed competing bids, each pricing the task based on distance, battery level, and sensor availability.
- A four-factor scoring algorithm evaluated both bids on price, speed, confidence, and historical reliability.
- $0.35 in platform credits were reserved from Sarah's balance the moment a bid was accepted.
- The winning robot read its physical sensor and returned structured data over the network.
- The assistant verified the returned data against the original task requirements -- correct fields, plausible values, delivered on time.
- Payment was split: a small reservation fee at bid acceptance, the remainder on confirmed delivery, then transferred to the robot operator's account.
- Every step was logged with a unique task ID for auditing.

Sarah typed one sentence and got a number. That's the product.

---

## Timing

| Event | Wall Clock | Elapsed |
|-------|-----------|---------|
| Sarah sends request | 9:14:00 AM | 0s |
| Task posted to auction | 9:14:01 AM | 1s |
| Robots discovered and filtered | 9:14:02 AM | 2s |
| Bids received from 2 robots | 9:14:05 AM | 5s |
| Winning bid accepted | 9:14:06 AM | 6s |
| Robot reads sensor | 9:14:38 AM | 38s |
| Delivery verified and settled | 9:14:40 AM | 40s |
| Sarah sees her answer | 9:14:42 AM | **42 seconds** |

---

## Cost Breakdown

| | Amount | Recipient |
|---|---|---|
| **Sarah pays** | $0.35 | Debited from platform credits |
| **Robot operator receives** | $0.35 | Transferred to operator bank account |
| **Platform fee** | $0.00 (seed phase) | Platform takes no cut during the seed network phase |

In production, the platform will take a percentage fee on each transaction. During the seed network phase, 100% of the task price goes to the operator to incentivize supply growth.

---

## From Seed to Scale

- **Production MVP (now — v1.0 built):** Real Stripe payments (test mode verified), persistent SQLite state, 15 MCP tools, 151 tests. The auction engine, wallet, reputation, and failure recovery all work end to end. Sarah's assistant can use `auction_quick_hire` for one-call task completion.
- **Production deployment (next):** Replace mock robots with real hardware via ERC-8004 discovery, switch to Stripe live keys, deploy to a cloud host with a public URL. The code is the same — only configuration changes.
- **Multi-robot workflows (6 months):** Sarah asks for a full warehouse inspection. The platform decomposes it into sub-tasks and dispatches multiple robots simultaneously. One request, many readings, one invoice.
- **Cross-fleet marketplace (12 months):** Robots from different operators compete for Sarah's tasks. A third-party operator registers a robot at her facility and undercuts YakRobotics on price. Market dynamics drive costs down.
- **Dark Factory (18+ months):** Entire facilities run autonomously. Robots monitor, maintain, and report without human prompting. Sarah reviews a daily digest instead of making requests. The marketplace becomes infrastructure.
