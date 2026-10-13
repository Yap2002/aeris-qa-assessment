# E-commerce Test Plan — MFQ Framework

This is my test plan based on the assessment. I have not run these tests because no application or test environment was provided. All cases are **Not Run**. I would confirm the assumptions and targets below with the team before testing.

## M — Models

### M1. Scope and assumptions

My main concerns are wrong payments, overselling and stock differences between systems. I would test products, the cart, orders, payments, cancellations, shipments, returns and inventory sync with WMS/SC2P.

The question does not give all the business rules. I use the following assumptions for this plan.

| ID | Assumption to confirm |
|---|---|
| A1 | Stock is managed by SKU and warehouse. Customers cannot buy out-of-stock items. Quantities must be positive whole numbers. Adding an item to the cart does not reserve stock. |
| A2 | Creating an order reserves stock. Payment moves reserved stock to sold stock waiting to ship. Shipping reduces physical stock. The order and its reservation must be saved together. |
| A3 | Unpaid orders expire after 15 minutes. Cancellation or expiry releases stock once. A paid order can be cancelled only before the warehouse accepts it. Stock is released after warehouse cancellation is confirmed. The refund is tracked separately. |
| A4 | WMS holds the warehouse stock records. The online store holds reservation records. I would ask what SC2P does and which data it owns. |
| A5 | Messages can arrive twice or in the wrong order. Each message has an ID. Each stock operation also has a stable ID, and updates have version numbers. I would confirm these fields in the API specification. |
| A6 | An order can have several shipments or partial returns. Returned items become available only after the warehouse receives them and confirms they can be sold again. |
| A7 | Payment and refund status can be checked using the same reference number. A timeout does not mean the payment failed. |
| A8 | The numbers in Q are suggested test targets, not test results or confirmed business requirements. I would check them against expected traffic and the test environment. |

I would list unclear rules and ask the team to confirm them. I cannot decide whether a payment or stock test passes without knowing the expected rule. This plan does not cover full promotion, tax or recommendation features. It does check that the server calculates the correct order total.

### M2. End-to-end business flow

```text
Select SKU → Add/update cart → Validate price and stock → Create order + reserve
  ├─ Reservation fails → No payable order / no leaked reservation
  └─ Await payment
       ├─ Cancel / expire → Release reservation → Cancelled
       └─ Payment confirmed → Commit stock → Request fulfillment
            ├─ Cancel before warehouse acceptance → Confirm warehouse cancellation
            │    → Release committed stock + track refund → Cancelled
            └─ Warehouse accepts → Ship (possibly split) → Customer signs
                 → Optional return approval → Receive and inspect
                      ├─ Resalable → Restore sellable inventory
                      └─ Damaged → Keep aside; do not add to available stock
                 → Track refund independently until confirmed
```

Receiving a return, restoring stock and giving a refund are separate steps. Finishing one step must not mark the others as complete. For each external request, I would check its ID, timeout handling and recovery steps.

### M3. Order state machine

I would check payment and shipping status separately from order status. For example, an order can be cancelled while its refund is still pending. One order can also contain both delivered and unshipped items.

| Current state | Event and condition | Next state / effects |
|---|---|---|
| No order | Valid cart, current price accepted, all required quantities reserved | Pending payment; save the order and reservation together |
| Pending payment | Verified successful payment for matching order, currency and amount; reservation valid | Paid; move reserved quantity to committed exactly once |
| Pending payment | Cancel or expiry wins the state transition | Cancelled; release reservation exactly once |
| Cancelled | Late verified payment success | Remain cancelled; record payment and start a refund or review without repeating it; do not ship |
| Paid | Fulfillment request accepted | Processing; do not allow normal cancellation of accepted items |
| Paid | Cancellation requested before acceptance | Cancel pending; check whether acceptance or cancellation happened first before releasing stock |
| Cancel pending | Warehouse confirms cancellation / proves no fulfillment exists | Cancelled; release the cancelled stock once, refund remains separately pending until confirmed |
| Cancel pending | Warehouse proves fulfillment was accepted first | Processing; reject normal cancellation and follow the agreed return or delivery-refusal rules |
| Processing / Partially shipped | Valid shipment of unshipped quantity | Partially shipped or Shipped; adjust inventory only for that shipment |
| Partially shipped / Shipped | Receipt for a known shipment | Update delivered quantities; Signed only when all required quantities are delivered |
| Signed / Eligible delivered line | Return approved within policy | Track line-level return requested, received, inspected and refunded states |
| Any | Duplicate, stale or illegal transition | Do not repeat the change; accept, wait or reject and log the reason |

Key rules: shipped quantity cannot exceed committed quantity; returned quantity cannot exceed delivered quantity minus prior accepted returns; a payment must belong to the correct order. Return-window length and cancellation handling for partially fulfilled orders need business confirmation.

### M4. Inventory state model

For one SKU in one warehouse, I use four numbers:

- **A:** available stock.
- **R:** stock reserved for unpaid orders.
- **C:** stock sold but not yet shipped. I call this committed stock.
- **H:** physical stock that can be sold, including R and C. Damaged goods are not included.

C is not the total number of items ever sold.

In this model, **H = A + R + C**. None of these numbers can be below zero. New deliveries, stock transfers, losses and manual changes must each have a stock record.

| Business event, quantity q | ΔA | ΔR | ΔC | ΔH |
|---|---:|---:|---:|---:|
| Reserve | −q | +q | 0 | 0 |
| Payment commits reservation | 0 | −q | +q | 0 |
| Cancel / expire unpaid reservation | +q | −q | 0 | 0 |
| Confirm cancellation before shipment | +q | 0 | −q | 0 |
| Ship committed goods | 0 | 0 | −q | −q |
| Receive and approve resalable return | +q | 0 | 0 | +q |
| Receive a damaged return and keep it separate | 0 | 0 | 0 | 0 |

Example test data: H=10, A=10, R=0, C=0. Buy two: (A,R,C,H) becomes (8,2,0,10), then (8,0,2,10) after payment, then (8,0,0,8) after shipping. One resalable return produces (9,0,0,9).

I would also check that one request reserves stock only once, the correct SKU and warehouse are updated, and shipped/returned quantities match the records. Processing a message twice must not change stock twice.

### M5. External system interactions

| Interaction | What I would check |
|---|---|
| Customer → online store | User identity, SKU, quantity, order ID and request key. The server checks permissions and calculates prices. |
| Online store ↔ payment service | Order ID, payment/refund reference, amount, currency and callback identity. The final payment result can be checked. |
| Online store → WMS/SC2P | Operation ID, order/item ID, SKU, warehouse, quantity and version. The reply clearly shows accepted or rejected. |
| WMS/SC2P → online store | Message ID, source, order/item ID, version and shipment/return details. The store saves the message before confirming receipt. |
| Stock comparison job ↔ external systems | Stock records cover the same completed updates. Each difference has a reason, repair record and final check. |

I would use the actual API specification for field names and error codes. If reservations are handled by a separate service, I would also test pending orders and the release of reservations left behind by failed orders.

### M6. Inventory synchronization consistency

**Strong consistency:** stock reservation must prevent overselling immediately. For example, if two people try to buy the last item, only one can reserve it. The order and stock update must be saved together, or the system must safely recover the failed step. Messages waiting to be sent must not be lost after a restart.

**Eventual consistency:** the store and warehouse may show different stock for a short time. I suggest a 60-second sync limit during normal operation. If a request times out, check its result or retry with the same operation ID. Do not send a new stock deduction without checking the first one.

**Repeated and out-of-order messages:** check message IDs to find duplicates. Also check operation IDs so the same stock change is not applied twice. An older version must not replace a newer one. If an earlier update is missing, wait for it or check the source system.

**Reconciliation (comparing stock records):** compare each SKU and warehouse after both systems have processed the same set of updates. Include reserved stock, sold stock waiting to ship and updates still in progress. Warehouse physical stock is not the same as online available stock. I suggest checking recent changes every 5 minutes and all stock once a day. Fix the cause of a difference and check again. Running the fix twice must not change stock twice.

**Stock shortage:** if warehouse stock is below the amount already promised to customers, stop new sales for that SKU and alert the team. Do not overwrite reservations or hide the shortage by changing the number to zero.

## F — Functional

### F1. Test groups and priorities

The case groups below cover products/cart, orders/payment, stock sync, shipment/returns and APIs. Each case has a priority, test steps and an expected result.

- **P0:** problems with payments, overselling, permissions or the main buying flow. These must pass before release.
- **P1:** important error handling and recovery cases. These should pass. Any low-risk exception needs the review described in Q5.
- **P2:** small display or compatibility issues. I would test these after the main risks.

Default test data: SKU-A in WH-1 has (A,R,C,H) = (10,0,0,10). SKU-B has separate stock. Users U1 and U2 have separate orders. Reset the data before each case. Unless stated otherwise, buy 2 items.

For each case, check the response, saved order, stock records and messages. For stock sync, keep checking until the time limit. Save request IDs, stock numbers before and after, and timestamps.

An **idempotency key** identifies the same request across retries. For example, sending a payment request twice with the same key must not charge the customer twice.

### F2. Product and cart

| ID | Priority | Setup and action | Expected result |
|---|---|---|---|
| F01 | P1 | Select two variants under one SPU; add SKU-A and SKU-B | Correct SKU IDs, prices and stock displayed; variants and inventory remain independent |
| F02 | P1 | Add 1 unit, update to 2, then remove the cart line | Quantity and server-calculated totals update; cart actions never reserve inventory |
| F03 | P1 | Submit quantity 0, −1, 1.5, an excessive integer and a nonnumeric value | Invalid quantities rejected according to contract; no order or stock change |
| F04 | P1 | Add SKU-A, then change its price or deactivate it before checkout | Checkout checks the item again; changed price requires the agreed confirmation; inactive SKU cannot be ordered |
| F05 | P0 | Tamper with client unit price or total during checkout | Server rejects or recalculates using server prices; tampered amount never becomes payable |

### F3. Checkout, payment and order transitions

| ID | Priority | Setup and action | Expected result |
|---|---|---|---|
| F06 | P0 | Purchase 2 units; pay; accept fulfillment; ship; sign | States follow M3; balances follow M4; correct quantities and payment reference throughout |
| F07 | P0 | Set A=0; attempt to order 1 | Out-of-stock result; no successful reservation, charge or payable order |
| F08 | P0 | With A=10, separately request 10 and 11 units from reset test data | 10 succeeds with A=0/R=10; 11 fails with all balances unchanged |
| F09 | P0 | Create mixed order where one line has insufficient stock; assume all-or-nothing checkout | No payable partial order; any temporary reservations fully released; no stock left reserved by a failed order |
| F10 | P0 | Send identical order request twice with the same key; then reuse key with a different quantity | First pair returns the same business order and single reservation; changed request is rejected without changing data |
| F11 | P0 | Provider declines payment for a reserved order | No committed stock or paid state; reservation remains until retry/cancel/expiry under policy |
| F12 | P0 | Deliver the same successful payment callback twice, one after another and at the same time | One paid transition and one R→C conversion; no repeated stock or shipping action |
| F13 | P0 | Callback has wrong amount, currency or order reference | No paid transition or stock commitment; difference logged for review |
| F14 | P0 | Cancel an unpaid order twice | One release; final (A,R,C,H) (10,0,0,10); state remains Cancelled |
| F15 | P0 | Advance controlled clock just before, at and after 15-minute expiry | No early release; due reservations eventually released once; repeated expiry job has no extra effect |
| F16 | P0 | Send payment success and order expiry at nearly the same time. Test payment first, then expiry first in a second run | If payment wins, keep the order paid and stock committed. If expiry wins, keep it cancelled and release stock; handle the late payment by refund or review. Never ship released stock |
| F17 | P0 | Provider charges but response/callback is lost; query and retry with same reference | Query recovers successful payment; no second charge; eventual single commitment, or controlled refund if order already cancelled |
| F18 | P0 | Attempt shipment before payment, payment on a shipped order, and receipt for an unknown shipment | Reject or delay invalid changes as agreed; do not move back to an older state or change stock |
| F19 | P0 | Cancel a paid order while the warehouse accepts it. Test each action finishing first | If cancellation wins, release C once and track the refund. If acceptance wins, keep the stock assigned and reject normal cancellation |
| F20 | P0 | Refund times out after provider may have processed it; retry/query | One refund at most; refund status stays pending until verified; stock is not restored again by refund retry |

### F4. Dedicated inventory synchronization consistency cases

| ID | Priority | Setup and action | Expected result |
|---|---|---|---|
| F21 | P0 | Fail to save the reservation during order creation | No successful order with missing reservation; transaction/pending workflow resolves without leaked stock |
| F22 | P0 | Fail local commitment after verified payment | Payment result remains saved after restart; retry updates stock once, or issue one refund if the order cannot continue; no untracked paid order |
| F23 | P0 | Crash after local stock commit, before outgoing event delivery | Saved message is delivered after recovery; stock is not deducted again; stock records match after recovery |
| F24 | P0 | External stock operation succeeds but response is lost; retry same operation | One external effect; query/retry resolves unknown status; local and external stock records match |
| F25 | P0 | Deliver duplicate inventory events concurrently and after consumer restart | Apply the stock change only once; duplicates are still detected after restart |
| F26 | P0 | Deliver shipment v3 before required v2, then deliver v2 | Wait for the missing update or check the source; reach the correct final state once; no negative balance or silent event loss |
| F27 | P0 | Deliver old stock snapshot after a newer version | Old snapshot cannot overwrite newer balances or active allocations; old message is logged |
| F28 | P0 | Drop one synchronization event; run reconciliation twice | Missing operation discovered and repaired once; second run makes no further change; final balances match after accounting for reserved and unshipped stock |
| F29 | P1 | Deliver malformed message followed by valid messages for other orders | Invalid message is kept aside with a reason; other orders continue; fixing and resending it has one recorded effect |
| F30 | P0 | Disconnect WMS for 10 minutes while safe existing flows continue; restore connectivity | Pending tasks remain saved after restart; unconfirmed shipment is not shown as complete; waiting messages are processed and stock records match within recovery target |
| F31 | P0 | With A=1, submit 100 concurrent unique orders | Exactly one reservation succeeds in a healthy run; others receive controlled stock rejection; A/R/C never negative |
| F32 | P0 | Reserve SKU-A in WH-1 while SKU-A exists in WH-2 and SKU-B in WH-1 | Only intended SKU/warehouse changes; other warehouses and variants are unchanged |
| F33 | P0 | Inject physical stock shortage below current commitments | Flag affected orders and limit new sales; keep existing reservations and report the shortage |
| F34 | P1 | Restart reconciliation midway through a multi-SKU repair | Saved progress allows the job to continue safely; already repaired operations do not repeat; unresolved items stay visible |

### F5. Shipment and returns

| ID | Priority | Setup and action | Expected result |
|---|---|---|---|
| F35 | P0 | Pay for 3 units; ship 1 then 2 using distinct shipment IDs | First shipment leaves C=2/H=9, second C=0/H=7, A=7 throughout; order goes partially shipped then shipped |
| F36 | P0 | Repeat first shipment callback; then attempt shipment beyond remaining quantity | Duplicate has no effect; excess shipment rejected/flagged; quantities and H remain correct |
| F37 | P1 | Sign one of two shipments, then the other | Delivered quantities add up correctly; full Signed state only after all required quantities delivered |
| F38 | P0 | Ship/sign 2; receive and inspect 1 resalable return; replay receipt | A/H increase once from 8 to 9; remaining quantity allowed for return is 1; refund cannot exceed approved returned value |
| F39 | P0 | Approve a return without receiving it; then receive damaged goods | Approval alone never increases A/H; keep the damaged item separate; do not add it to available stock |
| F40 | P0 | Request return above delivered balance or for another user's line | Rejected without stock/refund change; check the order owner and quantity still allowed for return |
| F41 | P1 | Inspection completes but inventory restoration worker fails; replay recovery | Return stays marked as waiting for stock recovery; add stock once after recovery; no duplicate refund |

### F6. Critical API validations

| ID | Priority | Setup and action | Expected result |
|---|---|---|---|
| F42 | P1 | Leave out required fields; use an invalid option, SKU/warehouse pair, IDs and types | Error response and code match the API specification; no partial data change or internal stack trace |
| F43 | P0 | Supply forged, expired or modified payment/warehouse callback authentication | Rejected before business processing; security event logged without leaking credentials |
| F44 | P1 | Send valid shipment/return callback with missing or conflicting line quantity/version | No partial update; reject or keep the message for review as agreed; log the reason and request ID |
| F45 | P1 | Submit same idempotency key for different customers and retry after time limit for keeping keys | Keep customer requests separate; do not show another customer’s data; later retries must not repeat charges or stock changes |
| F46 | P1 | Force out-of-stock, state conflict, dependency timeout and unexpected server error | Return the agreed error code and whether retry is allowed; show rejected and pending results clearly; do not return secrets |

## Q — Quality

### Q1. Performance tests and suggested targets

I would use a separate test environment with settings close to production. As a starting point, use 10,000 SKUs, 100,000 past orders and 1,000 active users. Let 20% of users buy from a small group of popular products. Warm up for 10 minutes, test normal load for 30 minutes, then double the request rate for 5 minutes. Record the server setup and any limits of the load tool.

These numbers need team agreement. **p95 ≤800 ms** means at least 95% of requests finish within 800 ms. **Throughput** means how many requests or orders the system handles per second.

| ID / Priority | Workload / method | Proposed acceptance criteria |
|---|---|---|
| Q01 / P1 | Send requests at a fixed rate: 500 API requests/s, approximately 70% read, 20% cart and 10% order creation; keep enough stock available | Sustain ≥50 successful orders/s; p95 ≤300 ms for reads, ≤800 ms for order creation, p99 ≤2 s for order creation; unexpected 5xx/timeouts <0.1% |
| Q02 / P0 | Hot SKU: 100 units, 1,000 concurrent unique attempts; separately use F31's one-unit case | Exactly 100 successful reservations in the healthy run; no overselling; expected stock rejection counted separately from technical errors |
| Q03 / P1 | Double baseline traffic for 5 minutes, then restore baseline | No loss of work already reported as successful; overload causes documented controlled rejection; baseline latency/error targets recover within 5 minutes |
| Q04 / P1 | Continue representative load for 2 hours with scheduled reconciliation | Memory, connections and waiting messages do not keep growing without limit; latency remains within agreed target; final stock records match |
| Q05 / P1 | Measure event age from saving a stock change to seeing it in the other system | Under healthy baseline, ≥99% within 30 s and all valid events within 60 s; check every missed target |

Measure payment-provider response time separately from the store API. Record throughput, p50/p95/p99, errors, rejected orders, CPU, memory, database locks and message waiting time. Check slow requests as well as the average.

### Q2. Reliability, exception handling and compensation

I suggest a 3-second timeout for external calls. For temporary errors, retry with the **same operation ID**. Increase the wait between retries and add a small random delay so requests do not all retry together. Stop after 5 attempts in the first retry cycle. Keep failed tasks saved for a later recovery job or manual review. Do not keep retrying a clear business rejection, such as an invalid order.

If the result of a payment or stock request is unknown, check it first. A new request could charge the customer or deduct stock twice.

A **circuit breaker** pauses calls to a failing service. One starting setting is to pause for 30 seconds when more than half of at least 20 calls fail within 30 seconds. Then send a few trial requests before returning to normal. I would confirm these settings with the team. During failure, show shipping as pending or stock as possibly out of date. Do not show a payment or shipment as successful without confirmation.

| ID / Priority | Fault / action | Expected recovery and evidence |
|---|---|---|
| Q06 / P0 | Crash the message worker after saving the change but before confirming receipt | Redelivery causes no extra effect; saved data and duplicate checks survive restart |
| Q07 / P1 | Inject transient 503/timeouts, then sustained outage | Retries follow the delay and limit; the circuit breaker works; pending tasks are saved; trial requests confirm recovery |
| Q08 / P0 | Fail between payment and inventory commitment | Recovery locates payment by reference and finishes one commitment; if stock cannot be assigned, issue one refund and record the failure |
| Q09 / P0 | Run two compensation workers for the same reservation/stock repair | Release or correct stock once; log both attempts |
| Q10 / P1 | Restore dependency after 10-minute outage at baseline traffic | Process saved tasks that can be recovered within 15 minutes while accepting safe traffic; report other failures separately and assign someone to handle them |
| Q11 / P0 | Disable queue publication/DB connection during order handling | Do not report success before saving the result; do not lose work already reported as successful; no stock left reserved by a failed order |

**Compensation** means fixing a partly completed flow, such as refunding a payment when an order cannot continue. Keep a record of the fix. A failed refund stays pending. A warehouse cancellation also stays pending until confirmed; do not release its stock early. If retries fail, the alert should show the affected order, the reason and who will handle it.

### Q3. Observability and alerts

| ID / Priority | Check / injection | Acceptance evidence |
|---|---|---|
| Q12 / P1 | Trace one normal order and one failed synchronization across services | Correlation ID links order, payment, reservation, shipment and compensation; structured logs include operation/event IDs and state changes |
| Q13 / P0 | Inject a negative balance or excess reservation in a prepared test data | Immediate critical alert with affected SKU/warehouse; sales of the affected SKU can be stopped |
| Q14 / P1 | Create stock differences after both systems process the same updates | Metric: mismatched SKU/warehouse pairs ÷ eligible compared pairs; report count and quantity difference too; if no pairs can be compared, show no data rather than zero differences |
| Q15 / P1 | Age a message past 60 s; increase backlog; leave paid order without fulfillment acknowledgement for 5 min | Alerts identify event age and backlog, and stuck order with business state; alert routing and recovery notification verified |
| Q16 / P1 | Put sensitive fields in request/error paths | Logs and traces mask credentials, payment details and personal data according to the agreed policy; correlation remains usable |

I suggest an alert when the stock difference ratio is above 0.1% in two checks, 5 minutes apart. Also alert on any difference lasting more than 10 minutes, even if the ratio is small. Negative stock or overselling needs an immediate alert. The 5-minute stuck-order check is for a missing warehouse acceptance reply, not an order waiting for its planned shipping date.

### Q4. Security and recoverability

| ID / Priority | Setup and action | Expected result |
|---|---|---|
| Q17 / P0 | U2 reads/cancels/refunds U1's order; customer calls stock adjustment/admin API | Reject the request; do not change or expose private data; check both ownership and role on the server |
| Q18 / P0 | Call protected APIs without authentication or with expired/revoked tokens | Rejected; no protected state disclosed or changed |
| Q19 / P1 | Replay a signed callback; alter body after signing; try beyond replay policy window | Valid duplicate has no repeated effect; tampered/invalid replay rejected according to agreed contract |
| Q20 / P1 | Use injection payloads and excessive payload sizes in identifiers/notes | Safe validation/parameter handling; controlled resource use; no internal errors or unauthorized data access |
| Q21 / P0 | Restore service after process/node failure and replay outstanding events | No loss of work already reported as successful in this test; stock and orders match; suggested service recovery time ≤30 min |
| Q22 / P1 | Restore an isolated backup, replay retained journal/events, then reconcile with providers | Record recovery time and any lost data; find and repair missing updates; confirm separate recovery targets for a whole-region failure |
| Q23 / P0 | Introduce one known stock discrepancy; execute repair twice | Reviewed stock correction changes only intended SKU/warehouse; second run makes no further change; audit contains before/after, source fact, operator and operation ID |

For a data-repair test, I would:

1. Stop new sales for the affected SKU if needed.
2. Save the order and stock records from both systems.
3. Check which updates each system has processed and find the cause.
4. Review the planned correction before applying it.
5. Apply the correction once and compare the records again.
6. Run the repair again to check it does not repeat the stock change.
7. Resume sales when the numbers are correct. Keep the repair record and cause.

Use recorded stock corrections rather than changing balances without a record. Run this drill in a test environment.

### Q5. Entry and exit criteria

**Before formal testing**

- Confirm the main business rules, order states, stock ownership and API fields.
- Have a stable test build, prepared data and test versions of external services. Use mock responses where needed.
- Make logs, stock records, metrics and messages available for checking. Have a way to test timeouts and failures.
- Pass basic buying and cancellation tests. Agree who handles recovery and rollback.

**Before release**

- Run every P0 and planned P1 case. All P0 cases must pass.
- Fix all critical or high-risk bugs affecting payments, stock, permissions or recovery.
- P1 cases should pass. For a low-risk exception, record the impact, workaround, owner and fix date. Get agreement from the product and engineering teams.
- Have no unexplained stock differences, duplicate charges/refunds, overselling or changes made without permission in the test results.
- Meet the agreed performance and sync targets. Pass recovery, alert and data-repair tests.
- Review the results, open bugs and remaining risks with QA, developers and the person responsible for release.

I would hold the release if there is stock corruption, a duplicate payment, a permission failure or lost work that the system already reported as successful. If this happens after release, stop the affected flow and follow the recovery plan. Rolling back code does not undo a payment or shipment; these still need checking and repair.

### Q6. Test order and reporting

I would start with API and order-state tests. Next, use mock payment and warehouse responses to test failures, then check the full flow with real test services. Run concurrency, recovery and performance tests after the basic flow works.

I would automate repeated checks such as stock calculations and duplicate messages. For two actions happening at the same time, test both orders of completion and repeat the test. A single pass is not enough. I would also manually check whether the recovery steps are easy to follow.

For each run, record the case ID, build, environment, data, actual result, evidence and bug link. The final report should show which important business risks were tested and what remains untested.

**Status:** this plan contains 46 functional cases and 23 quality cases. None have been run. Confirm the rules and targets before using the test results to decide whether to release.
