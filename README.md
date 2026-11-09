# Dana's Bakery order processing

A beginner-friendly Python + LangGraph notebook. Six deterministic functions validate an order, check stock, try up to two forward alternatives, calculate the price, decide delivery, and write a friendly customer message. No LLM, OpenAI SDK, API key, or paid API is needed.

## Files

- [prompt.md](prompt.md): complete technical specification, assumptions, graph plan, and prompt-revision notes.
- [dana_bakery.ipynb](dana_bakery.ipynb): runnable notebook with explanations, the actual compiled graph diagram, examples, and assertions.

## Run in Google Colab

[Open the notebook directly in Google Colab](https://colab.research.google.com/github/meira199/dana-bakery-langgraph/blob/main/dana_bakery.ipynb).

To upload a local copy instead:

1. Open [Google Colab](https://colab.research.google.com/).
2. Choose **File → Upload notebook** and upload `dana_bakery.ipynb`.
3. Use a Python CPU runtime; no GPU or credentials are required.
4. Select **Runtime → Run all**. The setup cell installs `langgraph==1.2.20` if needed.
5. Confirm the test cells print `PASS`, then edit `my_order` in the final cell and run it again.

Run cells from top to bottom in a fresh session. Package installation requires internet access. The graph diagram uses `draw_mermaid_png()` with the free Mermaid rendering service; if that service is unavailable, the notebook displays the compiled graph's Mermaid source and continues processing orders and running tests. The renderer receives graph structure, not customer orders. No AI service is called.

## Example

After running the setup and graph cells:

```python
result = graph.invoke({
    "item": "babka",
    "quantity": 2,
    "day": "Thursday",
    "city": "Jerusalem",
})
print(result["message"])
```

This sells **2 rugelach**, the first available substitute, for **60 NIS + 20 NIS delivery = 80 NIS**. Results contain exactly `status`, `item`, `delivery`, `total`, `message`, and `steps`.

## Conventions

- Each invocation uses the original stock snapshot independently; inventory is never decremented.
- Substitutions are automatically accepted because the input has no confirmation field. The full quantity is preserved and the substitute's price is used.
- Alternatives only move forward through babka → rugelach → sourdough → challah, with a maximum of two attempts. Babka × 5 is rejected; challah cannot be a third alternative.
- Avi's authoritative rule is 15% off baked goods only above 150 NIS and only on a non-Friday. Delivery is added afterward and is never discounted.
- Use positive Python integers for quantities. Floats, numeric strings, booleans, and missing values are rejected. Use full English weekday names. All four input fields are required.
- Text matching ignores surrounding spaces and case. An empty city means free pickup; Jerusalem costs 20 NIS for delivery; other cities become free pickup with an explanation.
- Rejected results use `item=None`, `delivery="pickup"`, and `total=0`. Pickup is an output default for a cancelled order, not an actual booking.
- `steps` records functional node visits in chronological order, including repeated checks. Conditional router functions and virtual START/END markers are not processing nodes.

## Local validation

Validated on **November 9, 2026**, using **Python 3.12.14** and **LangGraph 1.2.20**:

- Validated the notebook against the nbformat schema and checked Python syntax in all 14 code cells, including IPython transformation of `%pip`.
- Executed all 14 code cells in a fresh local Jupyter kernel; the delivered notebook includes their saved outputs.
- Passed **45 test orders** covering validation, available stock, first and second substitutes, exhausted substitutions, the two-alternative cap, discounts, Friday exclusion, delivery/pickup, text normalization, and missing fields.
- Checked all six required output keys and full expected histories for every test order, unchanged stock, and independent repeated invocations.
- Passed isolated pricing checks for the exact 150 NIS boundary, 15% above 200 NIS, Friday exclusion, and money rounding. These synthetic pricing checks do not change stock or bypass validation for real orders.
- Compared recorded histories against actual streamed node execution for four representative paths.
- Successfully rendered the actual compiled graph using `draw_mermaid_png()` and embedded the PNG in the notebook.

## Google Colab validation

The user subsequently confirmed successful execution in Google Colab: all **45 test orders** passed, the additional pricing and graph execution tests passed, and the compiled graph diagram displayed successfully. This Colab result was reported by the user; the local validation described above was performed during project creation.

The working notebook is preserved unchanged after that successful test. It uses no local file dependencies.
