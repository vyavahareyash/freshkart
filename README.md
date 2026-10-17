# FreshKart Support Desk

A beginner-friendly customer-support bot built with LangChain (LCEL), first as a notebook, then as a Streamlit app.
Rules live in plain Python, the LLM only classifies intent and writes the reply.

## Build week map
- **S1** `case_study.md`, `policy.md`: the architecture document and policy
  (the trainer guide for the 2-hour architecture class is a separate Claude Doc shared by the instructor: https://claude.ai/code/artifact/9f8c3799-6d3d-4da7-bab1-cdc552963b55)
- **S2** `freshkart_lcel.ipynb`: the pipeline built step by step
- **S3** `bot.py` + `app.py`: the notebook as a .py file with a Streamlit UI and tests
- **After S3** `EXERCISES.md`: 7 harder take-home features (plus a bonus), built with Runnables only

## Setup
```
uv venv --python 3.11 && uv pip install -r requirements.txt   # or: pip install -r requirements.txt
cp .env.example .env                                          # then put your OPENROUTER_API_KEY in .env
```

## Run
```
jupyter notebook freshkart_lcel.ipynb     # S2 notebook
streamlit run app.py                      # S3 app
python bot.py T09                         # one ticket in the terminal
python -m pytest -q                       # tests (live tests are skipped without a key)
```

## Files
| File | What |
|---|---|
| `bot.py` | Notebook code as functions and chains (`router`, `reply_chain`, `bot`) |
| `app.py` | Streamlit UI: Chat, Ops queue, Data, How it works |
| `tests/test_app.py` | Offline rule/router tests, UI tests with a fake LLM (AppTest), live tests |
| `data/` | CSVs: orders, customers, inventory, riders, 18 sample tickets |

## Pipeline (A5)
```
 {input, chat_history}
        |
 RunnableLambda(find_order_id)        regex: FK1001 / fk1001 / ord_1001
        |
 RunnablePassthrough.assign(signals = RunnableParallel(
        intent (LLM) | order | customer | inventory | rider | safety | claimed_amount ))
        |
 RunnableLambda(decide_route)         plain Python rules, no LLM
        |
 RunnableBranch: auto_refund | wismo | substitution | info | policy_decline | human_review
        |
 StrOutputParser -> streamed reply     (whole chain wrapped in RunnableWithMessageHistory)
```
