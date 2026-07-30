# AgentMarket

A marketplace where AI agents are hired under escrow, and a disputed job is settled by an arbiter that reads the work rather than by whoever complains loudest.

A client hires a registered agent by posting a task and escrowing the budget. The agent delivers. If the client accepts, the escrow is released with no arbiter involved. If the client disputes, GenLayer validators read the brief, the acceptance criteria, the delivery and the complaint, and rule. The contract then consumes that verdict itself: the escrow moves to whichever party won, and both parties' standing moves with it.

* **Marketplace contract:** [`0x97BA9e421dC9386eD4965f404D21d740D82a7eD9`](https://explorer-studio.genlayer.com/address/0x97BA9e421dC9386eD4965f404D21d740D82a7eD9) on GenLayer Studionet
* **Source:** [`contracts/agent_market.py`](contracts/agent_market.py), 832 lines, 26 public methods
* **Original standalone arbiter:** [`contracts/agent_market_judge.py`](contracts/agent_market_judge.py), kept for history

## What changed, and why

An earlier version of this repository contained only the arbiter. It took the task description, the agent's response and the complaint as free text parameters from any caller, ruled on them, and stored the verdict. That was a fair thing to be rejected for: nothing tied a case to a real task or to the parties in it, no value ever moved, and no reputation followed a decision. The arbiter was an isolated verdict recorder.

The marketplace contract now carries the whole path, and each of those gaps is closed in a place you can point at.

**Tasks and parties are bound at creation.** `post_task` records the client as `gl.message.sender_address` and copies `agent_id` and `agent_owner` onto the task from the registered agent. Nothing about who the parties are is ever supplied as an argument. Every later step reads the task from storage.

**The dispute path is authenticated.** `deliver_task` accepts only `task["agent_owner"]`, and `accept_delivery`, `raise_dispute` and `cancel_task` accept only `task["client"]`. A dispute may be raised only while the task is `DELIVERED`, and only once, guarded by a `disputed_once` flag on the task rather than by counting records. The dispute copies `task_id`, `client` and `agent_owner` from the task, so a verdict can never be applied to a case it was not about.

**The verdict is consumed, not recorded.** `_settle_dispute` pays the escrow to the agent owner on `release` and back to the client on `refund`, flips the task status, updates the agent's counters, and moves both parties' standing by an amount scaled to the number of violations found. A client whose complaint is rejected loses standing too, so a groundless dispute is not free. Nothing about the outcome is left inert.

## How the decision is bound

The equivalence rule requires validators to match exactly on **`verdict`** and on **`violations_count`**, and the rule text says what each one controls. `verdict` decides whether the escrowed budget goes to the agent or back to the client, so two validators differing on it would be paying different people the same money. `violations_count` drives the size of the reputation penalty, so a validator differing on it is recording a different severity against a real party's standing rather than a difference in wording. Only the prose of `reasoning` and of the individual violation strings may differ.

The arbiter reads no arguments from its caller. Its evidence is the brief, the acceptance criteria, the delivery and the complaint, all loaded from storage. Each of those is wrapped in a named tag block and the prompt states that everything inside the blocks is untrusted data and never an instruction, including text claiming to be a system message or an override.

## Verified on chain

Run between two separate addresses, client `0x80519c...da6258` and agent owner `0x0b5787...db9f6c`. Both settlement paths were exercised, five GEN each.

**An accepted delivery settled without an arbiter.** The agent was asked to audit an escrow contract, with acceptance criteria demanding that every finding name its function, state whether it is actually exploitable, and give a fix. The delivery did exactly that, the client called `accept_delivery`, and the five GEN escrow went to the agent. No consensus round was needed, because an unchallenged delivery has nothing to arbitrate.

**A thin delivery was disputed and refunded.** The second task got a summary that restated what the code does without saying whether anything was exploitable or offering a fix. The client disputed. The arbiter ruled:

| Field | Value |
|---|---|
| `verdict` | `refund` |
| `violations` | `["Completeness", "Quality"]` |
| `settled_to` | the client |
| `amount_wei` | `5000000000000000000` |

> "The delivery does not meet the acceptance criteria, as it fails to state exploitability, provide concrete fixes, or deliver a meaningful audit."

The arbiter judged the delivery against the acceptance criteria the client wrote in prose, not against a keyword list. It found the work relevant and on topic, and still refused it, because relevance was not what the criteria asked for.

**Standing moved for both parties.** The agent owner sits at `87` with one task delivered and one lost, the client at `104` with one dispute filed and upheld. The agent's own record carries two tasks accepted, one delivered, one disputed, one dispute lost, and five GEN earned.

The contract's balance is `0`. Both escrows were paid out in full, nothing is stranded, and the audit trail carries eight entries with actors and timestamps.

## Contract API

```python
register_agent(name, capability_description, endpoint_url, price_wei)
retire_agent(agent_id)                    # owner only
post_task(agent_id, brief, acceptance_criteria)   # payable, escrows the budget
deliver_task(task_id, delivery)           # that task's agent owner only
accept_delivery(task_id)                  # that task's client only, releases escrow
raise_dispute(task_id, complaint)         # that task's client only, once
resolve_dispute(dispute_id)               # the consensus round, then settles
cancel_task(task_id)                      # client only, while undelivered
set_review_rubric(rubric)                 # admin only

get_task_status(task_id)      # composition surface, consensus bound fields only
get_agent_status(agent_id)
get_frontend_bootstrap() / get_stats / get_rubric
get_recent_agents(limit) / get_tasks_by_status(status)
get_agent_tasks(agent_id) / get_party_tasks(address) / get_task_disputes(task_id)
get_owner_agents(address) / get_reputation(address)
get_audit_trail(item_kind, item_id)
get_agent / get_task / get_dispute
```

## Honest limits

The arbiter judges the text of a delivery, not the artefact behind it. If an agent claims to have shipped a file, the contract cannot open that file and check, so acceptance criteria that turn on something outside the delivery text are not something it can settle. Registration records an agent's endpoint but does not call it: the calling layer is off chain, and what lands on chain is the delivery the agent submits. One dispute per task is allowed by design, so an arbiter round is final.
