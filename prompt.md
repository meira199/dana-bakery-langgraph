# Dana's Bakery — technical specification

Build a beginner-friendly order processing system for Dana's Bakery using Python and LangGraph in a Google Colab notebook.

First, create a file named `prompt.md` containing this technical specification. Then create a notebook named `dana_bakery.ipynb`.

## Business data

Available products, unit prices in NIS, and available stock:

| Product | Unit price (NIS) | Stock |
| --- | ---: | ---: |
| challah | 18 | 10 |
| babka | 45 | 0 |
| rugelach | 30 | 3 |
| sourdough | 25 | 4 |

Croissants are discontinued and must be rejected. All other unknown products must also be rejected.

## Order input

Each order contains `item`, `quantity`, `day`, and `city`.
Process each order independently using the given stock snapshot.

## Validation

- Reject unknown or discontinued items.
- Reject zero, negative, non-integer, or otherwise invalid quantities.
- Return a polite rejection message.

## Stock and substitutions

- Accept the original product only when enough stock exists for the full quantity.
- If stock is insufficient, try alternatives in this exact sequence: **babka → rugelach → sourdough → challah**.
- Only move forward in this sequence.
- Try at most **TWO** alternative products.
- Each alternative must have sufficient stock for the full original quantity.
- When a valid substitute is found, automatically use it, keep the original quantity, and charge the substitute's price.
- If no acceptable product is found within two alternatives, reject the order.
- Never try a third alternative.
- Document the assumption that proposed substitutions are automatically accepted because the input has no customer-confirmation field.

## Discounts

The final authoritative rule is **Avi's correction**:

- Apply a **15% discount only when the baked-goods subtotal is strictly greater than 150 NIS AND the day is not Friday**.
- Never use the earlier 10% over 200 NIS rule.
- Never discount the delivery fee.
- Apply the discount before adding delivery.

## Delivery

- If `city` is empty, use pickup for free.
- If `city` is Jerusalem, use delivery and add 20 NIS.
- If `city` is any other city, change the order to free pickup and clearly inform the customer.
- Handle surrounding spaces and capitalization sensibly.

## Customer message

- Produce warm, friendly, readable customer messages through deterministic Python templates only.
- Explain substitutions, order cancellations, and pickup changes when relevant.
- Do **NOT** call an LLM or any AI API.
- Ignore the Instagram request; it was withdrawn.

## LangGraph design

- Use `StateGraph` and a clear typed state.
- Build separate functional nodes for order validation, stock checking, alternative selection, price/discount calculation, delivery decisions, and message generation.
- Use conditional edges for validation and availability decisions.
- Use a repeating substitution path with a strict maximum of two alternatives.
- Track all actual visited steps in chronological order using the state.
- Use a reducer for step-history updates if appropriate.
- Ensure the graph can reach `END` for every valid and rejected path.

## Output

The graph must support:

```python
result = graph.invoke({"item": "babka", "quantity": 2, "day": "Thursday", "city": "Jerusalem"})
```

Every result must contain:

- `status`: `"accepted"` or `"rejected"`
- `item`: the product actually sold, or `None` when rejected
- `delivery`: `"pickup"` or `"delivery"`
- `total`: final amount in NIS including delivery, or `0` when rejected
- `message`: a friendly customer-facing message
- `steps`: chronological list of actual processing steps

For rejected orders, use pickup as the default delivery value and document this output convention.

## Notebook requirements

- Install LangGraph if necessary.
- Make the notebook run in a fresh Google Colab session.
- No OpenAI SDK, LLM calls, API keys, or paid APIs.
- Draw the actual compiled graph using `draw_mermaid_png()`.
- Include an explanation of the state, nodes, fixed edges, and conditional edges.
- Include at least 10 test orders covering validation failures, original item available, first substitute, second substitute, failed substitutions, discount, Friday exclusion, Jerusalem delivery, empty-city pickup, and outside-Jerusalem pickup.
- Test that every output includes all six required keys.
- Use assertions for important expected results.

## Files and workflow

- `prompt.md`: this complete technical specification, including design assumptions and a short section listing any later prompt revisions.
- `dana_bakery.ipynb`: the working Colab notebook.
- Optional `README.md` with run instructions.
- Explain the planned graph structure before implementing it, then create the files.
- Validate the Python and notebook structure locally. Do not claim live Colab execution unless it was actually tested.
- Original delivery instruction: do not commit or push to GitHub yet; the user wants to review and test everything first. This hold is superseded by the publication revision below after successful user testing.

## Design assumptions and output conventions

1. **Automatic acceptance:** a sufficient substitute is automatically accepted because the input has no customer-confirmation field. The message explains the replacement and its price.
2. **Independent snapshot:** every invocation uses the same fixed catalog and stock; no invocation decrements stock or shares order history with another invocation.
3. **Strict quantities:** accept positive Python integers only. Reject booleans (even though Python treats them as integers), floats such as `2.0`, numeric strings such as `"2"`, missing values, and other types. Do not silently round or coerce quantities.
4. **Text normalization:** trim surrounding whitespace and compare product names, full English weekday names, and cities case-insensitively. Recognize both `croissant` and `croissants` as discontinued. A whitespace-only city is empty. Require all four fields; reject non-string text fields, missing fields, and unrecognized weekday names politely rather than guessing a day or city.
5. **Forward-only candidates:** alternatives are the next consecutive products after the requested product in the fixed sequence, without wrapping or skipping an insufficient product. For babka, only rugelach and sourdough may be tried; challah must never be the third attempt. Challah has no forward alternatives.
6. **Money:** calculate with `Decimal` and round to two decimal places using `ROUND_HALF_UP`; expose the final `total` as a plain numeric NIS value. Price the full unchanged quantity at the final product's unit price, apply any baked-goods discount, then add delivery.
7. **Rejected output:** always return `status="rejected"`, `item=None`, `delivery="pickup"`, and `total=0`. Here pickup is an output default, not a confirmed pickup booking. Explain that the order was cancelled and no payment is due.
8. **History:** each executed functional node appends its own name to `steps` using an append reducer. Repeated stock checks and alternative selections remain visible in execution order. Conditional router functions and the virtual `START`/`END` markers are edges/control markers, not functional processing nodes, so they do not add history entries.
9. **Schemas:** input fields hold untrusted values until the validation node checks them. A typed internal state stores normalized values, candidate, attempt count, availability, money, and history. A typed output schema exposes the six required result fields.
10. **Termination:** after each insufficient-stock check, route to another alternative only if fewer than two have been tried and a next forward candidate exists. Otherwise generate a rejection. Alternative selection also guards the two-attempt limit. Every branch ends after message generation.
11. **Rendering:** call `graph.get_graph().draw_mermaid_png()` on the compiled graph. Its default free Mermaid image renderer may need internet access; this is diagram rendering, not an AI API. If rendering is unavailable, explain the failure and show the compiled graph's Mermaid text so order processing and tests can still run.

## Planned graph structure

There are six functional nodes:

1. `validate_order`: validate and normalize input; initialize per-order fields.
2. `check_stock`: check the current candidate against the full original quantity.
3. `select_alternative`: move forward by one product and increment the alternative count.
4. `calculate_price`: price the selected product and apply Avi's discount rule.
5. `decide_delivery`: choose pickup/delivery and add the delivery fee after discount.
6. `generate_message`: create a deterministic friendly confirmation or cancellation and finalize the required result fields.

Fixed edges: `START → validate_order`, `select_alternative → check_stock`, `calculate_price → decide_delivery → generate_message → END`.

Conditional edges:

- After validation: valid → `check_stock`; invalid → `generate_message`.
- After a stock check: sufficient → `calculate_price`; insufficient with a permitted next candidate → `select_alternative`; otherwise → `generate_message`.

The only loop is `check_stock → select_alternative → check_stock`. The count increases on each alternative selection and can never exceed two.

## Later prompt revisions

- Avi's 15% / strictly-over-150 / non-Friday correction supersedes the earlier discount rule.
- The Instagram request is withdrawn.
- After the user confirmed that all 45 test orders, additional pricing and graph execution tests, and graph rendering passed in Google Colab, the user authorized publication: review files and Git status, exclude temporary files/secrets/local validation artifacts, initialize Git if needed, create the public `dana-bakery-langgraph` repository under the user's GitHub account, commit only `prompt.md`, `dana_bakery.ipynb`, `README.md`, and `.gitignore`, and push to `main`. Provide repository and direct Colab links. Preserve the working notebook unless a change is necessary. This revision supersedes the earlier hold on committing and pushing.
