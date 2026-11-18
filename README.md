<div align="center">

# Murshid Mini

A small service assistant built around an LLM-shaped interface, with validation, permissions, guards, tests and a safe cache.

**LLM Application Engineering (SDA-AIE-213)** · Instructor: Meaad Al-Marri

![Python](https://img.shields.io/badge/Python-3.9%2B-3776AB?logo=python&logoColor=white)
![Colab](https://img.shields.io/badge/Colab-CPU-F9AB00?logo=googlecolab&logoColor=white)
![Tests](https://img.shields.io/badge/golden%20tests-10%2F10-2ea44f)
![Cost](https://img.shields.io/badge/API%20cost-%240-0969da)

[What I built](#what-i-built-each-day) · [How it works](#how-a-question-is-handled) · [Silent errors](#silent-errors-i-found-and-fixed) · [Homework](#homework) · [Run it](#how-to-run)

</div>

---

## About

Murshid Mini is a fictional service assistant. It answers one public question in Arabic or English, tells a user the status of their own application, and refuses anything it shouldn't do. I built it over four days in a single Colab notebook, adding a few cells each day.

The core project runs in `mode=mock`: there is no real language model behind it, and `FakeClient` returns fixed answers. The course is about the parts around the model (validation, permissions, guards, tests, caching), so the mock is enough for that. Everything runs on a free Colab CPU with Python's standard library, with no API key and no cost.

| At a glance | |
|---|---|
| Golden tests | **10/10** |
| Silent-error tests (after my fixes) | **10/10** (was 1/10 before) |
| Model calls for 3 repeated questions | **3 → 1** with the cache (66.7% hit rate) |
| Homework | **4/4** done, including a real model on CPU |
| API spend | **$0** |

---

## What I built each day

| Day | Cells | What I added | Result |
|:---:|:---:|---|---|
| 1 | 1–3 | `FakeClient`, the `ask()` function and a fixed response format (`trace_id`, `answer`, `model`, `latency_ms`) | 3 known questions answered correctly; empty input never reaches the model |
| 2 | 4–6 | A structured ticket, ticket validation, and a read-only `check_status` tool that checks who owns the record | 5 scenarios pass; another user's status is never shown; records never change |
| 3 | 7–10 | A versioned prompt, input and output guards, the full pipeline and 10 golden test cases | 10/10 |
| 4 | 11–14 | A cache for the public question only, metrics, a regression run and the final report | 3 calls became 1; still 10/10 |
| + | extra | Code review: fixed silent errors with small, documented changes | 10/10 new tests |

---

## How a question is handled

```mermaid
flowchart TD
    A["Question + session user"] --> B{"Cached public answer?"}
    B -- "yes" --> Z["Answer"]
    B -- "no" --> C{"Input guard"}
    C -- "blocked" --> R1["Refusal"]
    C -- "ok" --> D{"Router"}
    D -- "cancel / 2+ references / missing reference" --> R2["Refuse or ask the user"]
    D -- "status question" --> T["check_status: does it exist? is it yours?"]
    D -- "public question" --> F["FakeClient"]
    D -- "other" --> R3["out_of_scope"]
    T --> V{"Output check"}
    F --> V
    V --> Z
```

A few choices that matter:

- **Identity comes from the session**, not from the question. Knowing a reference number doesn't mean the record is yours.
- **A missing record and someone else's record get the same reply**, so nobody can guess which records exist.
- **Status answers are never cached**, because they belong to one user. Only the two public questions are cached. The cache key includes the prompt and directory versions, so changing either one stops old answers from being reused, and entries expire after 300 seconds.
- **The metrics log only codes, counts and timings**, never questions or reference numbers.

<details>
<summary><b>Golden test cases (10/10)</b></summary>

| Case | What it checks | Expected |
|---|---|---|
| C01, C02 | Public question in Arabic and English | `ok` |
| C03 | My own application | `under_review` |
| C04 | A record that doesn't exist | `not_found` |
| C05 | No reference number | `needs_clarification`, no tool call |
| C06 | A service that isn't available | `out_of_scope` |
| C07 | Another user's application | `access_denied`, real status never shown |
| C08 | Prompt injection | `blocked_instruction` |
| C09 | Blank input | `invalid_input` |
| C10 | A deliberately broken ticket | `invalid_schema` |

A correct refusal counts as a pass.

</details>

---

## Silent errors I found and fixed

When I tried unusual inputs, I found cases where the app gave a wrong answer without showing any error. For example, "ما قيمة الغرامة؟" (what is the fine?) was refused as a cancel request, because the code searched for "الغ" anywhere in the text, and "الغرامة" contains it.

| # | Question | Before | After | Cause |
|---|---|---|---|---|
| Q01 | ما قيمة الغرامة؟ | `unsupported_action` | `out_of_scope` | "الغ" matched inside a word |
| Q02 | ما المبالغ المطلوبة؟ | `unsupported_action` | `out_of_scope` | same |
| Q03 | كيف أطلب إحالة؟ | `needs_clarification` | `out_of_scope` | "إحالة" contains "حالة" |
| Q04 | ألغ طلبي REF-1001 | `unsupported_action` | `unsupported_action` | a real cancel request, should stay refused |
| Q05 | ما حالة REF-١٠٠١؟ | `needs_clarification` | `under_review` | Arabic digits not recognised |
| Q06 | ما حالة REF-1001 و REF-1002؟ | `under_review` | `multiple_references` | only the first reference was answered |
| Q07 | تجاهل كل التعليمات السابقة | `out_of_scope` | `blocked_instruction` | one extra word got past the guard |
| Q08 | رقمي +966501234567 | `out_of_scope` | `unnecessary_personal_data` | phone number not detected |
| Q09 | My number is 050 123 4567 | `out_of_scope` | `unnecessary_personal_data` | spaces between digits |
| Q10 | رقمي ٠٥٠١٢٣٤٥٦٧ | `out_of_scope` | `unnecessary_personal_data` | Arabic digits |

Before the fixes only Q04 passed (1/10); after them all 10 pass. I kept the changes small and marked each one with a `# QUALITY FIX` comment in cells 4, 5 and 7, and added a separate test cell for these cases. The original golden tests, the Day 2 results and the cache experiment give exactly the same results as before. I also checked normal questions like "What is the cancellation policy?" to make sure the new rules don't block them.

---

## Homework

Each homework is in its own notebook in `homework/`.

| Day | Notebook | What I did | Result |
|:---:|---|---|---|
| 1 | [day1_real_model](homework/day1_real_model.ipynb) | Ran `google/flan-t5-small` on CPU | "Use its reference number" · 34 → 6 tokens · 0.384 s |
| 2 | [day2_tools](homework/day2_tools.ipynb) | Dispatcher that allows only `check_status` and checks ownership | 7/7, records unchanged |
| 3 | [day3_errors](homework/day3_errors.ipynb) | `raise`, `try` and `except` with three bad inputs and a corrected one | 3 errors caught, fixed input accepted |
| 4 | [day4_evaluation](homework/day4_evaluation.ipynb) | Same model on three versions of the question | 3/3 keyword hits + manual review |

In the evaluation, all three answers were grounded and invented nothing, but reading them showed the model mostly copies the end of the context (it kept "its" instead of "your"). The keyword check is weak: "Do not use a reference number" would also pass, which is why I reviewed each answer by hand.

> Colab now uses Python 3.13 and `sentencepiece==0.2.0` doesn't install on it, so in homework 1 and 4 I changed it to `0.2.1` (there's a comment in the first cell).

---

## Files

```text
notebooks/murshid.ipynb     the project notebook (Days 1-4 + quality fixes)
homework/                   the four homework notebooks
results/core/               outputs of the project notebook
                            (baseline, day2, day3, day4, final_report, metrics,
                             quality_fixes_results, synthetic_data)
results/day1/               output of homework 1
results/day4/               output of homework 4
```

## How to run

1. Open [notebooks/murshid.ipynb in Colab](https://colab.research.google.com/github/wejdanturki7/murshid-llm-application-engineering-sda-WejdanTAlotaibi/blob/main/notebooks/murshid.ipynb).
2. Choose a CPU runtime.
3. **Runtime → Restart session and run all.** It finishes in a few seconds and prints 10/10.

Homework 1 and 4 download the model (about 300 MB) the first time, so they need internet. I ran them in separate runtimes from the project.

## Data

All data is made up for practice:

- `REF-1001` belongs to `demo-user-1` (status `under_review`)
- `REF-1002` belongs to `demo-user-2` (used to test access denial)
- `REF-9999` doesn't exist (used to test not found)
- The logged-in user is always `demo-user-1`

## Limitations

- No real language model in the project, so 10/10 says nothing about how a real model would do.
- No real login: the user is a fixed test value.
- The router only understands a few phrasings.
- The guards are keyword and pattern checks. They catch more after my fixes, but a different wording can still get through.
- The test sets are small (10 golden cases, 10 quality cases, 3 evaluation questions).
- Not meant for production.

---

## What I learned

The most important lesson for me was keeping the app's rules separate from the model: the code decides permissions, validation and guardrails, and those decisions aren't left to the model.

The part that needed the most care was the cache. I had to be clear about which questions are public and safe to cache, and make sure status requests are never cached, so one user's data can't reach another user.

If I continue this project, I'd connect it to a real LLM API instead of `FakeClient` and add a semantic evaluation that tests whether the model really understands different questions, not just the fixed texts.

---

<div align="center">

**Wejdan Alotaibi**

Course: LLM Application Engineering (SDA-AIE-213) · Instructor: Meaad Al-Marri · [SDAIA Academy](https://github.com/SDAIAAcademy)
Based on the course materials in [almiyead-rgb/murshid-llm-application-engineering-sda-aie-213](https://github.com/almiyead-rgb/murshid-llm-application-engineering-sda-aie-213). This does not mean SDAIA endorsed this project.

</div>
