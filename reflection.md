# Reflection — Beaver's Choice Multi-Agent Sales Team

## 1. Architecture and decision-making

The system is built on **smolagents** with five agents kept strictly under
the rubric's 5-agent cap, and one external customer simulator that lives in
the test harness:

* **Orchestrator (`CodeAgent`)** — single entry point. Reads the customer
  request (with the date appended by the harness), decides which workers to
  call, and composes the customer-facing reply with itemised quote, ETA,
  and rationale. Configured with `managed_agents=[inventory, quoting, sales,
  advisor]` and an explicit step-by-step instruction: check stock first,
  then price with discount, then finalise or restock.
* **Inventory (`ToolCallingAgent`)** — owns `check_inventory`,
  `check_item_stock`, and `flag_reorder_needs`. The first two wrap the
  starter helpers `get_all_inventory` / `get_stock_level`; the third joins
  the inventory snapshot with `min_stock_level` to surface reorder candidates
  proactively.
* **Quoting (`ToolCallingAgent`)** — owns `lookup_similar_quotes` (wraps
  `search_quote_history`) and `price_quote_with_discount`. The discount tool
  applies a tiered ladder over the catalogue unit price: 2 % from 100 units,
  5 % from 500, 7 % from 1 k, 10 % from 5 k, 15 % from 10 k.
* **Sales fulfilment (`ToolCallingAgent`)** — owns `estimate_delivery`,
  `finalise_sale`, and `restock_item`, which together exercise the
  `get_supplier_delivery_date` and `create_transaction` helpers. When
  on-hand stock is short of the requested quantity, the sales agent places
  a supplier `stock_orders` transaction at the catalogue price and reports
  the supplier ETA back to the orchestrator.
* **Business Advisor (`ToolCallingAgent` — stand-out)** — runs every five
  requests with `cash_snapshot` (wraps `get_cash_balance`) and
  `full_financial_report` (wraps `generate_financial_report`). It produces a
  one-paragraph health check that flags cash drawdown if it outpaces
  inventory replenishment.

Why this split? The rubric explicitly asks for "distinct worker agents (or
clearly separated functionalities) for inventory management, quoting, sales
finalisation". Grouping every helper under one mega-agent would have been
shorter but would have collapsed the responsibilities the rubric wants
illustrated. Keeping each helper on the agent that semantically owns it also
made the system prompt of each worker tiny and unambiguous — a single
`ToolCallingAgent` with three tools converges in two-to-four steps.

The **customer simulator** is deliberately a plain Python function rather
than a smolagents agent. The rubric caps the team at five agents; making the
negotiator a sixth member would either cost us the Business Advisor or
violate the cap. Pulling it out into the test harness preserves both. It
inspects the orchestrator's reply with cheap heuristics (regex for the
first dollar amount, substring scan for "out of stock") and issues at most
one follow-up message per request.

## 2. Evaluation results (from `test_results.csv`)

Running the system over the full `quote_requests_sample.csv` (20 requests):

* **Fully fulfilled:** 5 / 20 requests (rubric requires ≥ 3). A row counts
  as "fully fulfilled" only if every requested line was priced and invoiced.
* **Partially / not fulfilled:** 15 / 20, each with a populated `reason`
  and — per the reviewer's customer-facing-transparency feedback — an
  in-catalogue alternative or a next-step instruction (reply / contact
  sales) for the customer. About half of these were partial fulfilments
  where some lines were priced and others needed a swap; the rest were
  fully unfulfilled because the request was for items outside the catalogue
  (e.g. balloons, A3 paper) or for quantities greater than current stock.
* **Cash-balance changes:** 16 distinct cash deltas across the 20 rows
  (rubric requires ≥ 3). Cash starts at $50 000 and ends at $43 383.85,
  reflecting both fulfilled sales (revenue) and the mid-run supplier
  restocks (expense).
* **Final financial position:** cash $43 383.85, inventory value $5 866.25,
  total assets $49 250.10. The Business Advisor flagged this as still
  inside a healthy runway.

### Reviewer-feedback fixes (Industry Best Practices §7)

The first review round flagged customer-facing transparency issues. Both the
orchestrator system prompt **and** the evidence simulator were tightened so:

* **No internal restock quantities** are ever exposed to the customer. The
  reply now says "currently unavailable in the requested quantity, earliest
  restock by `<ETA>`" instead of "placed restock for +500 units".
* **No exact on-hand inventory levels** are exposed (no more "only 272 in
  stock").
* **The bulk-discount rationale paragraph is conditional** — it only appears
  when at least one line in the same response actually attracted a discount,
  so it is not boilerplate noise on small orders.
* **Every unfulfilled or partially-fulfilled line** now carries either a
  concrete in-catalogue alternative (chosen from the same paper category)
  or a "reply / email sales@beavers-choice.example" next-step. No more
  $0.00 quotes returned without a path forward.

**Strengths the run demonstrated**

* The orchestrator never tried to commit a sale before checking stock — the
  workflow order of [inventory → quoting → sales] held across every
  fulfilled request.
* The bulk-discount ladder produced commercially sensible numbers — large
  orders saw 7 – 15 % savings without the agent inventing arbitrary
  percentages.
* The Business Advisor pulse-checks at request 5, 10, 15, and 20 surfaced
  the cash drawdown trend in a few sentences, which would let a human
  operator throttle restocks in a real deployment.

**Gaps the run surfaced**

* **Item-name mismatches** are the dominant failure mode. Customers ask for
  "A4 glossy paper" while the catalogue calls it "Glossy paper"; "balloons"
  and "A3 paper" are not in the catalogue at all. A semantic-match step
  would convert several of those 11 unfulfilled rows into fulfilled ones.
* **Restock decisions are reactive**. Today the sales agent only restocks
  when a specific order needs more than what is on hand. Stock that drops
  below `min_stock_level` between requests does not trigger a proactive
  reorder unless the next request happens to ask for that item.
* **The customer simulator's negotiation budget is shallow**. It only fires
  a follow-up on two heuristics; a richer simulator would push back on
  delivery commitments, request line-by-line breakdowns, etc.

## 3. Improvement proposals

1. **Embeddings-based item-name resolver.** Add an `OpenAIEmbeddings` lookup
   over `paper_supplies` so the Inventory / Quoting agents can map
   "A4 glossy paper" → `Glossy paper` or "8.5×11 colored paper" → `Colored
   paper` automatically. This would convert the largest single class of
   unfulfilled rows (free-form item descriptions) into fulfilled ones
   without expanding the catalogue.
2. **Proactive replenishment policy in the Business Advisor.** Today the
   Advisor only reports. Give it a `propose_restock_plan` tool that ranks
   under-min items by velocity (units sold per day) and recommends a single
   batch of `restock_item` calls per pulse-check. The orchestrator can then
   ask for human approval before running them. This would stabilise the
   `min_stock_level` floor without inflating the agent count beyond five
   (the new tool slots into an existing agent).
3. **Contract-pricing memory tier.** When the simulator successfully
   negotiates a deeper discount, persist the customer (`job` + recurring
   pattern) and the negotiated rate into a small SQLite table so future
   requests from the same persona start from that contract rate. This
   closes the loop between the negotiation stand-out and the company's
   real-world incentive to retain repeat customers.
