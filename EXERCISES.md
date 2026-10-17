# FreshKart Build Week: Take-home Exercises (after Session 3)

You have a working FreshKart support bot. `bot.py` builds one pipeline:

```
{input, chat_history}
  -> RunnablePassthrough.assign(order_id = RunnableLambda(order_id_from))
  -> RunnablePassthrough.assign(signals = RunnableParallel(intent, order, customer, inventory, rider, safety, claimed_amount))
  -> RunnableLambda(decide_route)                         = router
  -> RunnableLambda(add_note) -> RunnableBranch(6 lanes)  = reply_chain
```

FreshKart's ops team is happy, but they have asked for **7 new features** (plus a bonus). Each one solves a real problem from `case_study.md`. Each one also needs a LangChain idea you have not used yet, or a new way of using one you have.

Difficulty: ★ easy → ★★★★★ hard. Do them in order: later exercises build on earlier ones.

---

## Ground rules (read these first)

These rules are the point of the exercise. A feature that works but breaks a rule does not count.

1. **Every feature is a Runnable in `bot.py`.** You must be able to test it from a terminal with `.invoke()`, `.batch()` or `.stream()`, without opening Streamlit.
2. **`app.py` only displays things.** No business logic, no `if route == ...` decisions, and no extra LLM calls in `app.py`. If you are writing an `if` in `app.py` that changes *what the bot decides*, it belongs in the pipeline.
3. **Do not break the pipeline shape.** Add stations to the belt. Do not replace the belt with one big Python function that calls the LLM by hand (that is Architecture A3, and we know its flaws).
4. **Policy rules stay in plain Python (`decide_route`).** The LLM never decides money or safety.
5. **Keep it simple:** functions and plain dicts only. No pydantic, no `BaseModel`, no classes, no `with_structured_output`, no agents, no tools, no RAG or vector stores.
6. **The 18 sample tickets must still route correctly.** Run the evaluation after every exercise: 18/18, every time.
7. **Never print or commit your API key.**

### How to test one Runnable on its own

Every station on the belt has the same three buttons. Use them to test your new piece before you plug it in:

```python
# python -i bot.py   (or a notebook cell after `from bot import *`)
my_new_step.invoke({"input": "FK1001 mein bread missing tha"})
my_new_step.batch([{"input": "..."}, {"input": "..."}])
for chunk in my_new_step.stream({"input": "..."}): print(chunk)

# The whole router on the 18 tickets (your regression test):
res = router.batch([{"input": t["message"], "chat_history": []} for t in TICKETS])
print(sum(r["route"] == t["expected_route"] for r, t in zip(res, TICKETS)), "/ 18")
```

---

## Exercise 1: PII shield before the LLM ★★

**The story.** Section 7 of the case study says customer PII must not go into prompts unless it is needed (DPDP Act). Customers often paste their phone number, email or UPI id into the chat, and today all of it goes straight to OpenRouter.

**What to build.**
- A station that masks the data before any prompt sees it:
  - phone numbers → `[PHONE]`
  - emails → `[EMAIL]`
  - UPI ids → `[UPI]`
  - card numbers (12–16 digits) → `[CARD]`
- The **original** text must still be used by the regex signals (`order_id`, `claimed_amount`, `safety`), so that nothing about routing changes.
- The intent prompt and the 6 reply lanes must read the masked text.
- Order ids like `FK1001` and rupee amounts like `₹520` must **not** be masked.

**Required concept.**
- A `RunnableLambda` that masks text.
- `RunnablePassthrough.assign(masked_input=...)` at the **start** of `router`, so the box carries both `input` and `masked_input`.
- The intent branch inside `signals` and the lane prompts must now read `masked_input`.

**Where it plugs in.** At the front of `router`, before the existing `assign(order_id=...)` step. Inside `signals`, the intent branch gets a small `RunnableLambda` adapter so the prompt receives the masked text.

**Acceptance tests.**

| Message | Expected |
|---|---|
| `Please call me on 98450 12345 or mail ravi.k@gmail.com, UPI ravi@okaxis. FK1001 mein Britannia bread missing tha, refund do.` | The text the LLM receives has `[PHONE]`, `[EMAIL]` and `[UPI]`. The order id is still `FK1001` and the route is still `auto_refund`. |
| T13 (`₹520 twice debited`) | `₹520` is not masked. `claimed_amount` is still 520. |
| All 18 tickets | Still 18/18. |

Prove the first test by showing the masked text the intent prompt received. Hint: `intent_prompt.invoke({...}).to_string()`, or print it in a tiny `RunnableLambda` placed before the LLM.

**Hints.** `re.sub`, `RunnablePassthrough.assign`, `RunnableLambda(lambda x: {"input": x["masked_input"]})`. Watch the order of your regexes: an email contains an `@`, and so does a UPI id.

**Stretch.** Show a "PII masked: 3 items" badge in the "Why this answer?" panel. The *count* comes from the pipeline; `app.py` only displays it.

---

## Exercise 2: Language signal + reply in the customer's language ★★

**The story.** FreshKart gets tickets in English, Hinglish, Hindi, Kannada and Telugu (case study, section 3). Ops wants to see the language in the queue, so they can assign Kannada tickets to Kannada-speaking agents. The bot must reply in the customer's language.

**What to build.**
- A new signal, `language`, with one of these labels: `english`, `hinglish`, `hindi`, `kannada`, `telugu`, `other`.
- It runs **at the same time** as the other 7 signals.
- The lane prompts receive `{language}` and must reply in it. If the language is `other`, reply in English.
- `language` appears in the "Why this answer?" panel and as a column in the Ops queue.

**Required concept.**
- Add a new key to the existing `RunnableParallel`, built as `prompt | llm | StrOutputParser() | RunnableLambda(clean_up)`.
- The clean-up function must fall back to `"english"` if the LLM answers with anything outside the list. This is the same pattern as `parse_intent`.
- Pass it on to the lanes through `add_note` or a new `assign`.

**Acceptance tests.**

| Message | Expected |
|---|---|
| `FK1007 ಆರ್ಡರ್ ತುಂಬಾ ತಡವಾಗಿ ಬಂತು, ಯಾಕೆ?` (Kannada: "order came very late, why?") | `language = kannada`, reply in Kannada, route `wismo`, and the ₹50 coupon is mentioned. |
| `FK1003 లో వచ్చిన పెరుగు ఎక్స్‌పైర్ అయిపోయింది, రీఫండ్ కావాలి` (Telugu: "the curd was expired, I want a refund") | `language = telugu`, reply in Telugu. |
| T01 | `language = hinglish` |
| All 18 tickets | Still 18/18. |

**Think about it.** Your new signal is a second LLM call. Did the router get slower? Time `router.invoke` before and after, and explain the result in one line. (Hint: what does `RunnableParallel` do?)

**Stretch.** Add the intent and the language in **one** LLM call that returns `intent,sentiment,language`, then compare the cost and speed against two calls. Which would you ship under the ₹2-per-ticket budget?

---

## Exercise 3: Never-down intent with retry + fallback model ★★★

**The story.** On Diwali night OpenRouter returned errors for 4 minutes, and the bot crashed for 3,000 customers. Ops wants a bot that **never crashes**. If the AI is down, the safe answer is "send it to a human".

**What to build.**
- The intent chain retries a failed call (2–3 attempts).
- If the main model still fails, it tries a **backup model**, for example `meta-llama/llama-3.1-8b-instruct` on OpenRouter.
- If the backup also fails, the intent becomes `"unclear"`. Your existing `decide_route` already sends `unclear` to `human_review`.
- The reply lanes also need a fallback: a fixed, polite message such as *"Sorry, we're facing a technical issue. A FreshKart agent will contact you shortly."*

**Required concept.**
- `.with_retry(stop_after_attempt=...)`
- `.with_fallbacks([...])`
- A final fallback that is just a `RunnableLambda` returning `"unclear"`, or the fixed text for the lanes.

No `try/except` anywhere: the chain itself must be fault-tolerant.

**Where it plugs in.** On the `intent` branch inside `signals`, and on each lane inside `branch`. Hint: wrap what `lane()` returns.

**Acceptance tests.**

| Setup | Expected |
|---|---|
| `.env` has `MODEL=this/model-does-not-exist` | All 18 tickets still get a reply with no exception. Tickets that needed the LLM show the backup model answered. |
| Both models broken (e.g. a wrong API key for this test only) | `intent = unclear`, route `human_review`, and the fixed apology text. No crash. |
| Normal key | Still 18/18. |

**Hints.** `ChatOpenAI` already retries internally (`max_retries`). Set `max_retries=0` while testing, so you can see *your* retry working. Ask yourself which errors are worth retrying: a wrong model name (404) will never succeed, but a timeout might.

**Stretch.** Record which model actually answered (main or backup) and show it in "Why this answer?".

---

## Exercise 4: Ops hand-off card for escalations ★★★

**The story.** When a ticket goes to `human_review`, the senior agent reads the whole chat to understand it. That is 2–3 minutes per ticket. Ops wants a **hand-off card** made at the same moment as the customer reply:

```json
{"team": "food_safety", "priority": "P1", "summary_en": "Child vomiting after eating biryani from FK1009.", "next_step": "Call customer within 30 min; file FSSAI report within 24h."}
```

- `team` is one of `food_safety`, `finance`, `rider_ops`, `support`.
- `priority` is one of `P1`, `P2`, `P3`.
- `summary_en` is always in English, even if the customer wrote in Hinglish or Kannada.

**What to build.**
- For `human_review` tickets only, produce **both** the reply to the customer **and** the card, in parallel.
- Other routes produce no card (`None`).
- The Ops queue shows team, priority and summary as columns.

**Required concept.**
- A `RunnableParallel(reply=..., card=...)`.
- A card chain built as `prompt | llm | JsonOutputParser()`.
- A `RunnableBranch` so the card is only made for `human_review`, and every other route gives `None`.

Design choice: you can put the parallel step *inside* the human_review lane, or after the whole branch. Pick one and explain why in a comment.

**Acceptance tests.**

| Ticket | Expected card |
|---|---|
| T09 (food poisoning) | `team = food_safety`, `priority = P1` |
| T13 (double debit) | `team = finance` |
| T10 (rude rider) | `team = rider_ops` |
| T01 (auto refund) | `None`, and no extra LLM call is made (prove it). |

**Watch out.** Your `reply_chain` now returns a dict, not a string, so `st.write_stream` will break. Stream it and print the chunks: what do you see? Fix it in the *display* code only. The pipeline is right; the screen just needs to show `reply` chunks as text and the `card` once it arrives.

**Hints.** `JsonOutputParser` lives in `langchain_core.output_parsers`. Tell the model "Answer ONLY with JSON with keys team, priority, summary_en, next_step". Also check `team` against the allowed list in a `RunnableLambda` after the parser, so a bad answer falls back to `support`. The LLM is not trusted with business values.

---

## Exercise 5: Bulk triage for the morning backlog ★★★

**The story.** Every morning there are about 1,200 tickets that came in overnight. Ops wants to upload them as a CSV and get every ticket **routed** (no replies yet). Then they can staff the human-review queue before 9 am.

**What to build.**
- A new "Bulk triage" area in the Ops tab: upload a CSV with columns `id,message`, then click "Triage".
- In `bot.py`, a `bulk_triage` Runnable that takes the list of messages and returns one row per ticket: `id, order_id, intent, route, reasons`.
- One broken row (an empty message, or a message that makes the LLM call fail) must **not** stop the others. That row shows `route = error`.
- Show the total time, and a count per route.

**Required concept.**
- `router.batch(inputs, config={"max_concurrency": 5}, return_exceptions=True)`
- A `RunnableLambda` that turns the results (some of which may be exceptions) into plain row dicts.

**Acceptance tests.**

| Test | Expected |
|---|---|
| Upload `data/tickets.csv` | 18 rows, and the routes match `expected_route` (18/18). |
| Time it against a `for` loop calling `router.invoke` | The batch is at least 2x faster (we measured about 8s against 23s with `max_concurrency=5`). Show both numbers. |
| Add a row with an empty message | That row shows `error` or `info`, and the other 18 are fine. |

**Think about it.** What happens with `max_concurrency=50` on a free OpenRouter key? Why does the setting exist?

---

## Exercise 6: Model switcher at runtime ★★★★

**The story.** The CTO asks: "Is `gpt-4o-mini` really the best value? Try a cheaper Llama and a stronger model." Rebuilding the chain for every model is messy. FreshKart wants **one** chain object whose model can be changed per request.

**What to build.**
- A sidebar select box to choose between at least 2 OpenRouter models, for example `openai/gpt-4o-mini` and `meta-llama/llama-3.1-8b-instruct`.
- The choice is passed in with each request. The chain is **not** rebuilt.
- A "Compare models" button that runs the 18-ticket evaluation on each model. It shows the score out of 18, the total time, and (stretch) the total tokens.

**Required concept.**
- `llm.configurable_alternatives(ConfigurableField(id="model"), default_key=..., other_key=ChatOpenAI(...))`, or `configurable_fields(model_name=ConfigurableField(id="model_name"))`.
- The choice is passed via `config={"configurable": {"model": "llama"}}` on `.invoke`, `.batch` and `.stream`.

**Where it plugs in.** Where `llm` is created in `bot.py`. Every chain that uses `llm` gets the switch for free. Think about why that works.

**Acceptance tests.**

| Test | Expected |
|---|---|
| The same `router` object with `{"model": "mini"}` and `{"model": "llama"}` | Both run. The scores are printed. |
| Switch the model in the UI mid-chat | The next reply uses the new model, and memory is kept. |
| Which model sent T18 (the "VIP ₹2000") ticket to `human_review`? | Both. Explain why the **model choice cannot change** this route. (Hint: where does the ₹1,000 rule live?) |

**Stretch.** Read `usage_metadata` from the model's message to count tokens, and estimate ₹ per ticket for each model.

---

## Exercise 7: Live pipeline tracker ★★★★

**The story.** Ops does not trust a black box. While a ticket is being processed, they want to *watch the belt*:

```
✓ order lookup         0.01s
✓ customer lookup      0.01s
✓ safety keywords      0.00s
✓ intent (LLM)         0.84s
✓ decide_route         0.00s  -> human_review (food safety)
… writing reply
```

**What to build.**
- Every station gets a readable name.
- As each station finishes, the UI shows a line with its name and how long it took. Use `st.status` or a placeholder that updates.
- The parallel stations should **visibly overlap**: the intent takes ~0.8s, but the lookups finish at the same time, not after it.

**Required concept.** Name each step with `.with_config(run_name="intent")`. Then capture start and end, either:
- with `.with_listeners(on_start=..., on_end=...)`, where the listener gets a run object with the name and timings, or
- with `router.astream_events(..., version="v2")`, filtered on `on_chain_end` events and the step names.

Pick one and explain why in a comment.

**Watch out.** Listeners attached to the stations inside a `RunnableParallel` run on **worker threads**, not on Streamlit's thread. Don't call `st.*` inside a listener. Instead, append `(run.name, run.start_time, run.end_time)` to a list, and let `app.py` display that list. In `on_end`, `run.outputs` is a dict like `{"output": ...}`, not the bare value.

**Acceptance tests.**

| Test | Expected |
|---|---|
| T09 | All 7 signals, `decide_route` and the lane appear with timings. |
| Sum of signal times vs total router time | The sum is **greater** than the total time, and you can explain why (parallel!). |
| Turn it off | The pipeline still works with no tracker attached. The tracker is optional, not part of the logic. |

---

## Bonus: Reply guard ★★★★★

**The story.** In testing, the `human_review` lane once wrote "Don't worry, your ₹780 will be refunded today". That promises money before a human has looked at the case. Legal says this must never happen.

**What to build.**
- After a `human_review` or `policy_decline` reply is drafted, a **second chain** checks it. The check is: "Does this reply promise a refund amount or date, or reveal internal words like route, reasons or internal note? Answer yes/no."
- If yes, a `RunnableBranch` replaces the draft with a safe template.
- The other lanes skip the check, so they cost nothing extra.

**Required concept.**
- A draft chain, then `RunnablePassthrough.assign(verdict=checker_chain)`, then a `RunnableBranch` that picks the draft or the template.
- Bonus points if part of the check is a plain-Python regex (₹ amounts, "refund today") and not only the LLM. Why is that safer?

**Explain the trade-off (required).** The guarded lanes can no longer stream token by token. Why? Is that acceptable for escalations? Write 3–4 lines.

---

## Extra test tickets

| id | message | used in |
|---|---|---|
| X1 | `Please call me on 98450 12345 or mail ravi.k@gmail.com, UPI ravi@okaxis. FK1001 mein Britannia bread missing tha, refund do.` | Ex 1 |
| X2 | `FK1007 ಆರ್ಡರ್ ತುಂಬಾ ತಡವಾಗಿ ಬಂತು, ಯಾಕೆ?` | Ex 2 |
| X3 | `FK1003 లో వచ్చిన పెరుగు ఎక్స్‌పైర్ అయిపోయింది, రీఫండ్ కావాలి` | Ex 2 |
| X4 | `FK1009 ka khana kha ke bacche ko ulti ho rahi hai` | Ex 4 (P1 food_safety) |
| X5 | *(empty message)* | Ex 5 |

---

## Submission checklist

- [ ] All features are in `bot.py` as Runnables. `app.py` only displays results.
- [ ] `grep -nE "pydantic|BaseModel|^class |with_structured_output" bot.py app.py` returns nothing.
- [ ] 18/18 on the sample tickets after **every** exercise.
- [ ] `pytest -q` passes. Add at least one test per exercise to `tests/test_app.py`. The fake LLM pattern there means most tests need no API key.
- [ ] For each exercise, a short comment in `bot.py` that starts with `# Ex N:` and names the LangChain concept used.
- [ ] Your `.env` is not committed.

## Grading rubric (per exercise, 10 points)

| Points | What we check |
|---|---|
| 4 | It works: every acceptance test passes. |
| 3 | It uses the **required** LangChain concept, inside the pipeline (not glued on in `app.py`). |
| 2 | The 18/18 regression still passes and `pytest` is green. |
| 1 | It is simple: short, readable, with a clear `# Ex N:` comment. |

The bonus is worth +10. The "Think about it" answers are discussed in class.
