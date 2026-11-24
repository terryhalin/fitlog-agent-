# FitLog Agent

A tool-calling agent that answers questions about a personal workout log, built for the
41-601 Agentic AI midterm (CMKL University). It uses the **Mango 1.5 9B** model via the **Orion API**.

The agent is deliberately simple: no frameworks, no retries, no error recovery and no reasoning
pattern in the prompt. The goal of the midterm is to observe where it fails.

---

## How to run

1. Install the packages (once):
   ```
   pip install -r requirements.txt
   ```
2. Copy `.env.example` to `.env` and add your Orion API key:
   ```
   ORION_API_KEY=your-key-here
   ```
3. Open `fitlog_agent.ipynb` and run all cells.
   The first run downloads the embedding model `all-MiniLM-L6-v2` (~90 MB).

Note: Mango's answers vary between runs, so re-running the test queries can give different
results from the ones in the failure log.

---

## Files

```
fitlog-agent/
├── fitlog_agent.ipynb       the agent: setup, data, tools, agent loop, test queries
├── failure_log.md           results of the 5 test queries and what went wrong
├── logs/
│   └── midterm_traces.json  full traces of the official run (tool calls, steps, tokens, answers)
├── data/                    dataset (see below)
├── requirements.txt         Python packages
├── .env.example             template for the API key (.env itself is never uploaded)
└── .gitignore
```

---

## How the agent works

| Part | What it does |
|---|---|
| **Tools** (4) | Python functions, each with a JSON schema that describes it to the model |
| **Tool registry** | Maps the tool name the model outputs to the Python function that runs it (from Week 4 lab) |
| **`llm_call`** | Sends the conversation and tool schemas to Mango and returns either a text answer or tool calls (from Week 4 lab) |
| **`run_agent`** | The agent loop: the model asks for tools, the loop runs them and sends the results back, until the model gives a final answer or `max_steps = 8` is reached (from Week 4 lab) |

Each run returns a trace with the tool calls, number of steps, tokens and the final answer.
The model's `reasoning_content` is also logged (not shown to the user), because I found that
Mango sometimes writes its final answer there and leaves `content` empty.

### Tools

| Tool | What it does |
|---|---|
| `get_workouts(start_date, end_date, exercise=None)` | Returns logged workouts between two dates (optional exercise filter) |
| `get_exercise_info(name)` | Returns an exercise's muscle groups, equipment and load note |
| `calculator(expression)` | Evaluates a maths expression exactly, e.g. volume = sets × reps × weight (from Week 4 lab) |
| `search_guides(query)` | RAG: finds the 2 training guides most similar in meaning to the query (Chroma + `all-MiniLM-L6-v2`) |

---

## Test queries and results

Five queries from simple (1 tool call) to complex (3+ tool calls). The expected answers are
computed from the data with pandas in section 5a of the notebook.

| # | Level | Query | Correct? |
|---|---|---|---|
| 1 | Simple | What muscle does the Bulgarian Split Squat mainly train? | No (empty answer) |
| 2 | Medium | How many hours of badminton did I play in total? | No (no tool call) |
| 3 | Complex | How did my chest training volume change from 2-8 Nov 2026 to 16-22 Nov 2026? | No (invented data) |
| 4 | Complex + RAG | According to the training guidelines, is my lower-body gym training well planned around my badminton sessions? | No (stopped too early) |
| 5 | Trap | How much did my deadlift improve from my first to my last session? | No (wrong dates) |

**0 of 5 correct.** The main causes: the agent does not know which dates the log covers, it
sometimes puts its answer in `reasoning_content`, and it does not always stick to its tool
results. Details are in [`failure_log.md`](failure_log.md).

### About the official run

The five test queries were run once, and that run is the one reported in the failure log.
After it, the Setup and Data cells and section 5a (ground truth) were re-run for display
changes. These cells do not call the model, so the test query results were not affected,
but their execution numbers appear out of order.

---

## Dataset

All data is **mock data** for one example user, created for this project. Because the data is
made up, every correct answer is known, so the agent's answers can be checked exactly.

The example user trains for **3 weeks (2-22 November 2026)** with this weekly routine:

| Mon | Tue | Wed | Thu | Fri | Sat | Sun |
|---|---|---|---|---|---|---|
| Badminton | Upper A | Badminton | Lower A | Upper B | Lower B | Badminton |

- **Upper / Lower** = upper-body / lower-body gym sessions; **A / B** = two different exercise sets.
- Weights increase slightly each week (progressive overload), e.g. Bench Press 60 → 62.5 → 65 kg.

### `data/workout_log.csv`: training diary (55 rows)

One row = one exercise on one day. Gym rows use sets, reps and weight; badminton rows use
duration. Columns that do not apply are left empty.

| Column | Meaning |
|---|---|
| `date`, `day` | When the exercise was done (YYYY-MM-DD) |
| `session` | Upper A, Upper B, Lower A, Lower B, or Badminton |
| `exercise` | Exercise name, e.g. Bench Press |
| `sets`, `reps`, `weight_kg` | Gym exercises: rounds, repetitions per round, weight |
| `duration_min` | Badminton only: session length in minutes |
| `notes` | Free-text comments |

### `data/exercises.csv`: exercise reference (20 rows)

One row = one exercise: `primary_muscle`, `secondary_muscles`, `equipment`, and `load_note`
(how the weight is recorded, e.g. *weight per dumbbell* vs *total bar weight*).
It includes some exercises that never appear in the log (e.g. Deadlift).

### `data/guides/`: training rules (6 documents, used for RAG)

Short guideline documents with clear, checkable rules: recovery and rest, weekly volume
(10-20 sets per muscle group), progressive overload, deloads, warm-ups, and combining
badminton with gym training.

### Realistic edge cases in the data

Real training logs are messy, so the data contains situations a naive agent may get wrong:

- A missed session (no Upper B on Fri 20 Nov).
- A bodyweight exercise recorded with weight 0 (Pull-Up).
- Dumbbell weights recorded per dumbbell, not in total.
- Similar names (Deadlift vs Romanian Deadlift); only Romanian Deadlift is logged.
- One shorter badminton session (120 min on Wed 11 Nov instead of 180).
- No planned rest day in the routine, although guide 01 recommends one (the only day without training, 20 Nov, is a missed session).
