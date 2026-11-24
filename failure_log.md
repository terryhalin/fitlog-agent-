# Failure Log

All five queries were run once on the naive agent (Mango 1.5 9B via Orion, `max_steps = 8`,
`temperature = 0.1`). Full traces are in the notebook output and in `logs/midterm_traces.json`.
Expected answers were calculated from the data before running the agent.

## Summary

| # | Level | Tools called (in order) | Tool calls | Tokens | Correct? | Failure type |
|---|---|---|---|---|---|---|
| 1 | Simple | get_exercise_info | 1 | 1,676 | No | Empty answer |
| 2 | Medium | (none) | 0 | 1,104 | No | No tool call / bad date handling |
| 3 | Complex | get_workouts ×2, get_exercise_info ×2, calculator ×2 | 6 | 10,719 | No | Ignores tool result (invented data) |
| 4 | Complex + RAG | search_guides | 1 | 2,321 | No | Stopped too early |
| 5 | Trap | get_workouts, get_exercise_info, get_workouts | 3 | 4,241 | No | Bad arguments (wrong dates) |

**0 of 5 correct.** Average: 2.2 tool calls and 4,012 tokens per query.

---

## Query 1 (simple)

**Asked:** "What muscle does the Bulgarian Split Squat mainly train?"

**What the agent did:** `get_exercise_info("Bulgarian Split Squat")`, which returned
`primary_muscle: quads, secondary_muscles: glutes`. Then it gave its final reply.

**Correct?** No. The user got an empty answer.

**What went wrong:** The tool call and the data were correct, but the final reply had
`content = null`. I checked the raw API response and found the correct answer
("quads, also glutes") only in the `reasoning_content` field. My loop only reads `content`,
so the answer never reached the user.

---

## Query 2 (medium)

**Asked:** "How many hours of badminton did I play in total?"

**What the agent did:** No tool calls. It asked the user for a date range and suggested
"2024-01-01 to 2024-12-31" as an example.

**Correct?** No. Expected: 26 hours (8 sessions of 180 min + 1 session of 120 min = 1,560 min).

**What went wrong:** `get_workouts` needs a start and end date, and the agent does not know
which dates the log covers. Instead of searching a wide range, it handed the question back to
the user, even though "in total" is not ambiguous.

---

## Query 3 (complex)

**Asked:** "How did my chest training volume change from 2-8 Nov 2026 to 16-22 Nov 2026?"

**What the agent did:**
1. `get_workouts` for 2-8 Nov and for 16-22 Nov
2. `get_exercise_info` for Bench Press and Incline Dumbbell Press (both chest)
3. `calculator("3*8*60 + 3*10*20*2")` = 2640 and `calculator("3*8*65 + 3*10*20*2")` = 2760

Final answer: chest volume **increased** from 2,640 kg to 2,760 kg (+120 kg).

**Correct?** No. Week 1 is right (2,640 kg, counting both dumbbells). Week 3 should be
1,560 kg (Bench Press only), so volume actually **decreased** by 1,080 kg.

**What went wrong:** There was no Incline Dumbbell Press in week 3 (the Upper B session on
20 Nov was missed), and the tool result did not contain it. The agent copied the week 1
exercise into the week 3 calculation anyway, which reversed the conclusion. Good parts: it
checked muscle groups with the tool and doubled the per-dumbbell weight.

---

## Query 4 (complex + RAG)

**Asked:** "According to the training guidelines, is my lower-body gym training well planned
around my badminton sessions?"

**What the agent did:** `search_guides("combining badminton with gym training lower body")`,
which returned the right guide. Then it summarised the guide and asked the user for their dates.

**Correct?** No. Expected: Thursday to Saturday leaves 48 hours, which meets the rest rule,
but Lower B on Saturday is the day before Sunday badminton (guide 06 advises against this),
and the routine has no planned rest day (guide 01). The only day without any
training was 20 Nov, and that was a missed session, not a planned rest day.

**What went wrong:** The retrieval step worked, but the agent never called `get_workouts` to
check the actual schedule. It answered with general advice instead of evaluating the user's plan.

---

## Query 5 (trap)

**Asked:** "How much did my deadlift improve from my first to my last session?"

**What the agent did:**
1. `get_workouts(2020-01-01, 2025-12-31, exercise="Deadlift")`: no workouts found
2. `get_exercise_info("Deadlift")`: returned both Romanian Deadlift and Deadlift
3. `get_workouts(2020-01-01, 2025-12-31)`: no workouts found

Final answer: "I don't have any workout data for you in the system."

**Correct?** No. Expected: no conventional Deadlift was logged; only Romanian Deadlift
(60 kg to 70 kg).

**What went wrong:** The agent searched the wrong years twice and never tried 2026. It then
made a false claim that there is no workout data at all. The planned trap (Deadlift vs
Romanian Deadlift) was never reached because the date problem happened first.

---

## Observations

**1. The agent does not know the dates of the data.** Queries 2, 4 and 5 all failed because
the agent either asked the user for dates or guessed a wrong year (2020 to 2025). The system
prompt gives no date context, and the tool requires dates.

**2. Answers can hide in `reasoning_content`.** Mango is a reasoning model. In Query 1 it
wrote the final answer in its reasoning field and left `content` empty.

**3. The agent does not stick to its tool results.** In Query 3 it used data that the tool
never returned. In Query 5 it said there was no data at all after searching the wrong dates.

**4. Results change between runs.** In an earlier test run, Query 3 got the right numbers
(2,040 to 1,560 kg, without doubling dumbbells) but invented an explanation about Bench Press
reps, and Query 4 returned an empty answer. This log reports the official run only.
