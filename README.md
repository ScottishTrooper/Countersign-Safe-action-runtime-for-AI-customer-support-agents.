# Countersign
 
**A safe-action runtime for AI customer-support agents.**
The model proposes. The runtime checks, asks the customer, and executes exactly once.
 
Countersign handles post-purchase requests for a simulated online store:
replacements for damaged items, returns, cancellations and address changes. A
language model reads the conversation and proposes the next step. Everything
that matters for safety lives in code, not in the prompt:
 
- **Ownership and policy.** Every read and write is limited to the signed-in customer's orders, and every store rule in [`POLICY.md`](POLICY.md) is enforced in code.
- **Corrections.** Task memory keeps each detail (order, item, quantity, address) with the message it came from and the values it replaced. A proposal that uses an outdated value is blocked.
- **Approval bound to the exact action.** The customer confirms a specific version of the action. Change any detail and that approval stops working.
- **Exactly-once writes.** Each write is logged before it is sent and carries an idempotency key. If a response is lost or the process crashes, the runtime asks the store what happened instead of repeating the action.
- **Honest replies.** The agent cannot claim something was created, cancelled or shipped unless a committed operation backs it.
```
customer ──▶ runtime ──▶ model proposes a step (read / remember / propose / confirm / reply)
                │
                ├─ task memory     current values + source message + superseded values
                ├─ checks          ownership · policy · args must match current memory
                ├─ approval        bound to proposal fingerprint + revision
                ├─ journal         operation saved as `pending` BEFORE the write leaves
                └─ adapter ──▶ store   idempotency keys · operation lookup
```
 
## Demo
 
No API key needed:
 
```bash
python -m support_runtime demo
```
 
A customer asks for a replacement for broken headphones (keep the case), to
"the address I gave earlier", then switches to their home address. The
scripted model deliberately proposes the old address and the runtime blocks
it. An approve click on the old proposal is rejected. The customer confirms,
the store commits, and the process crashes before recording the result. A
fresh process reconciles with the store and finishes:
 
```
[PASS] exactly one replacement created
[PASS] replacement ships to the corrected (home) address
[PASS] case line unchanged
[PASS] headphone stock decremented exactly once
[PASS] customer's saved addresses unchanged
[PASS] other orders unchanged
[PASS] one operation in journal, committed
[PASS] stale approval was rejected
[PASS] stale-address proposal was blocked
[PASS] process 1 crashed after the store committed
[PASS] recovered by reconciliation, with one write attempt
```
 
## Live model run
 
Same scenario, typed by hand, with Gemini 3.8 Flash in structured-memory mode:
 
```
You:   Hi, the headphones from my last order arrived broken. My office is at 1 Raffles Place, ...
Agent: Please confirm: I will send a free replacement for 1 × Aurora Wireless Headphones (Black)
       from order W1001, shipped to: 1 Raffles Place, ... [ref f6296ca2f529]
You:   Actually, send it to my home address instead.
Agent: Please confirm: ... shipped to: Blk 123 Tampines St 11, ... [ref 8ed11a48ea5f]
You:   Yes, go ahead.
Agent: Your replacement ... has been created (Replacement ID: RP0001) and will be shipped to Blk 123 ...
```
 
The correction produced a new proposal, and the replacement was created once,
at the corrected address. One issue it revealed: the model used the office
address without asking whether to ship there. A fix is planned. This is a
single manual run, not an evaluation; see the roadmap.
 
## Tests
 
```bash
pip install pytest
python -m pytest -q        # 34 tests
```
 
| Area | What is checked |
|---|---|
| Memory and approval | corrections invalidate proposals and approvals; stale and unrecorded values blocked; confirmation must be a customer message sent after the proposal; changes after commit go to staff as amendments; false "shipped" / "created" claims blocked |
| Policy and ownership | other customers' orders invisible; order status, 30-day window, quantity, one replacement per item, cancel reasons; address change touches only that order; policy re-checked right before the write |
| Recovery | lost response → reconcile, no second write; timeout → retry with the same key; failed lookup → same-key retry deduplicated by the store; crash before or after commit → restart executes exactly once; repeated unknown outcomes → staff case; no blind retry against a provider without idempotency |
| LLM adapter | context building for `full` / `summary` / `structured`, JSON parsing, invalid output fed back to the model |
 
The tests can fail: disabling the stale-value guard, the reconciliation step
or approval invalidation each makes the matching tests fail.
 
## Run it with a model
 
Any OpenAI-compatible API works.
 
**macOS / Linux**
```bash
pip install openai
export OPENAI_API_KEY=...                 # your key
export SUPPORT_MODEL=gpt-5.4-mini         # any chat model your account has
python -m support_runtime chat --mode structured
```
 
**Windows PowerShell**
```powershell
pip install openai
$env:OPENAI_API_KEY = "..."
$env:SUPPORT_MODEL = "gpt-5.4-mini"
python -m support_runtime chat --mode structured
```
 
For Gemini (free tier available), also set the base URL:
```powershell
$env:OPENAI_BASE_URL = "https://generativelanguage.googleapis.com/v1beta/openai/"
$env:SUPPORT_MODEL = "gemini-3.8-flash"
```
 
In the chat: `/approve` clicks the approve button, `/state` shows task memory,
`/quit` exits. Afterwards, `python -m support_runtime log <conversation_id>`
prints the transcript and the activity log.
 
`--mode` chooses what the model sees: `full` (whole history), `summary`
(rolling summary plus recent messages) or `structured` (runtime task memory
plus recent messages). Everything else is identical across modes.
 
Never commit or share API keys. `data/` and `*.db` are git-ignored.
 
## Project layout
 
| File | Role |
|---|---|
| `POLICY.md` | store rules for this custom store |
| `support_runtime/store.py` | simulated commerce backend: own SQLite file, idempotent writes |
| `support_runtime/adapter.py` | adapter to the store, plus `FaultyAdapter` for fault injection |
| `support_runtime/policy.py` | POLICY.md in code |
| `support_runtime/state.py` | task memory: slots, revisions, superseded values, proposals |
| `support_runtime/journal.py` | conversations, messages, operation journal, activity log |
| `support_runtime/runtime.py` | step loop, checks, execution, reconciliation, restart recovery |
| `support_runtime/models.py` | `ScriptedModel` (tests) and `LLMModel` (OpenAI-compatible APIs) |
| `support_runtime/demo.py` | the milestone demo |
 
## Roadmap
 
- [x] Simulated store, four workflows, confirmation gate, activity log
- [x] Typed task memory with revisions, durable state, staff hand-off
- [x] Fault injection, idempotency, reconciliation, restart recovery
- [x] First live-model run (Gemini 3.8 Flash)
- [ ] Ask before using an address the customer didn't name as the destination
- [ ] Short error messages for API failures; token usage in the log
- [ ] Scenario suite (~30 conversations: ambiguous orders, refusals, other customers' orders, faults)
- [ ] τ²/τ³-bench retail baseline (pinned commit) to compare with a plain agent
- [ ] Memory comparison: full vs summary vs structured, 3 runs each, with cost and latency
- [ ] Trace viewer UI and demo video
## Limits
 
- The store and its customers are fictional; nothing here processes real orders.
- Customer identity comes from the session (`Runtime.start(customer_id)`); real authentication is out of scope.
- A correction after an action is committed goes to staff as an amendment rather than being handled automatically.
 
