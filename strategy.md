# Test Strategy

**What I would focus on**

I would first check that a customer can buy an item, pay, receive it and return it. My main concerns are overselling, wrong payments and stock differences between the store and warehouse. I use M for business models, F for functional tests and Q for quality tests. This is a test plan; I have not run the cases.

**Before testing**

I would confirm when stock is reserved, deducted and restored. For this plan, I assume an order reserves stock, payment marks it as sold, and shipping reduces physical stock. A return adds available stock only after the warehouse checks the item. I would also confirm cancellation rules and what SC2P does.

**Test order**

I would run P0 cases first: normal purchase, last-item purchases, payment, cancellation, duplicate messages and permission checks. Next, I would test timeouts, messages arriving in the wrong order, split shipments and partial returns. These cases matter because errors can affect both customer money and stock.

I would start with API tests and use mock payment and warehouse responses to create failures. Then I would check the full flow with real test services. For each case, I would compare the order, payment and stock records, and save the actual result. I would automate repeated checks, such as stock calculations and duplicate requests. I would manually check whether the recovery steps are clear.

**Failure and recovery**

A timeout does not prove that a payment failed. I would check the result before sending another request and reuse the same operation ID. I would also test service restarts, retries and stock comparison jobs. A repair must work once without repeating the stock change if it runs again.

**Targets and trade-offs**

Suggested targets are 1,000 active users, 500 API requests per second and at least 50 orders per second. At least 95% of order requests should finish within 800 ms. Unexpected errors should stay below 0.1%, and normal stock sync should finish within 60 seconds. I would confirm these targets with the team.

If time is short, I would keep all P0 tests and the main recovery tests. I would reduce browser coverage and long-running load tests first, and record what was not tested. Mock services help create failures, but real integration tests are still needed.

**Release decision**

All P0 cases must pass. There must be no serious payment, stock or permission bugs. Planned P1 cases must run, and any low-risk exception needs team agreement. Performance, stock sync and recovery must meet the agreed targets. I would report the results and remaining risks before recommending release.
