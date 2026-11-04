# 🔍 Observability & Monitoring — Homework Assignment

Welcome to your hands-on homework for **Module 3, Lesson 4 — Observability &
Monitoring**.

In Lesson 2 of Module 2 the last takeaway was three words: *Logs. Logs. Logs.*
Nothing was built on it at the time. This homework is where it gets paid off. You
will look for one customer's payment inside half an hour of ordinary traffic, read
a dashboard that is green on a day when customers lost money, and decide which of
twelve overnight alerts were worth waking someone for.

Two tasks run in the same GitHub fork you have used since Lesson 1. The third needs
no computer, only your own product and twenty quiet minutes.

Budget **about 60 minutes**, plus ten for the reflection at the end.

---

## What You Will Do

1. **Task 1 — Find one payment in a haystack.** 186 log lines from three services.
   One customer's transfer never completed. Find out what happened to it.
2. **Task 2 — Read a dashboard that is lying to you.** Five green numbers, all of
   them correct, on a day with five support tickets and seven missing transfers.
   Decide which single metric would have caught it.
3. **Task 3 — Triage a night of alerts.** No tools. Twelve alerts fired between
   22:00 and 06:00. For each one: wake someone, ticket in the morning, or change
   the alert.

Copy-paste is provided for both workflows. You are not expected to write any YAML,
or to understand every line of it.

---

## How This Connects to the Lecture

| Lecture point | Where you will see it in this exercise |
|---|---|
| **Logs answer the question you thought of in advance** | Task 1: searching the customer's name returns one line and no story |
| **A trace is a shared id, agreed before anything went wrong** | Task 1: the same id in two services, in order, with timings |
| **The most useful signal is sometimes an absence** | Task 1: the third service logged nothing at all for that payment |
| **Monitoring watches known questions** | Task 2: five thresholds, all of them respected |
| **Observability is for the questions nobody wrote down** | Task 2: none of the five numbers can express what went wrong |
| **A metric measured outside the application knows nothing about the business** | Task 2: the gap between accepted and posted, and where it must be counted |
| **Thresholds, anomalies and what deserves a person** | Task 3: twelve alerts, three possible answers each |
| **Alert fatigue is a reliability risk, not an annoyance** | Task 3: alert 7, which has fired for forty nights and has never been real |

---

## Before You Start

**What you already have:** a fork of this repository at
`github.com/YOUR-USERNAME/sk03_kbc-it-academy-sm-po`, with Actions enabled.

Nothing else is needed. No environments, no secrets, no variables, no sign-up, no
credit card. Neither workflow touches the application code, so this homework does
not depend on anything from Lesson 3.

**Keep your fork public.** Actions minutes are free on public repositories. There is
nothing sensitive in this one.

**If you have lost the fork, or skipped the earlier homework:** fork
[the repository](https://github.com/turbobobbytraykov/sk03_kbc-it-academy-sm-po)
again, then open the **Actions** tab in your fork and click the green *"I understand
my workflows, go ahead and enable them"* button. That is the whole of the catch-up
and it takes two minutes.

---

## Task 1 — Find One Payment in a Haystack *(~25 minutes)*

The story: it is 15:30. A colleague in the branch calls. A customer, Marie Dupont,
made a transfer at about ten past two. The app confirmed it on screen. The money is
not in the other account and she wants to know where it is.

You have the logs. That is not the same as having the answer.

### 1a. Add the workflow

In your fork, click **Add file → Create new file** and name it exactly:

```
.github/workflows/trace-a-payment.yml
```

Paste this in:

```yaml
name: Trace a payment

on:
  workflow_dispatch:

jobs:
  busy-afternoon:
    runs-on: ubuntu-latest
    steps:
      - name: Half an hour of traffic, three services, one log
        run: |
          NAMES="Petrov Ivanova Georgiev Dimitrova Kolev Stoyanova Nikolov Angelova"
          i=0
          for MIN in $(seq -w 0 29); do
            for SEC in 11 34; do
              i=$((i + 1))
              TRACE=$(printf '%08x' $((0x5f000000 + i * 7919)))
              ACC=$((4400 + i % 37))
              NAME=$(echo $NAMES | cut -d' ' -f$(((i % 8) + 1)))
              printf '14:%s:%s api-gateway     trace=%s POST /transfers customer="%s" status=202\n' "$MIN" "$SEC" "$TRACE" "$NAME"
              printf '14:%s:%s payment-service trace=%s account=%s validate=ok 41ms\n' "$MIN" "$SEC" "$TRACE" "$ACC"
              printf '14:%s:%s ledger          trace=%s account=%s posted amount=%s.00 EUR\n' "$MIN" "$SEC" "$TRACE" "$ACC" "$((i * 13 % 880 + 20))"
            done
            if [ "$MIN" = "07" ]; then
              printf '14:07:41 api-gateway     trace=8ac41d02 POST /transfers customer="Dupont, Marie" status=202\n'
              printf '14:07:41 payment-service trace=8ac41d02 account=4471 validate=ok 39ms\n'
              printf '14:07:42 payment-service trace=8ac41d02 account=4471 calling ledger (attempt 1)\n'
              printf '14:07:52 payment-service trace=8ac41d02 account=4471 ledger timeout after 10000ms\n'
              printf '14:07:53 payment-service trace=8ac41d02 account=4471 calling ledger (attempt 2)\n'
              printf '14:08:03 payment-service trace=8ac41d02 account=4471 ledger timeout after 10000ms, giving up\n'
            fi
          done
```

Commit it **directly to `main`**. If you commit to a new branch instead, the workflow
will not appear in the Actions tab. This is the most common stumble in every one of
these homeworks.

Then go to **Actions → Trace a payment → Run workflow → Run workflow**. Open the run
and click the step name to expand the log. You should have 186 lines.

### 1b. Search for the customer

Use the search box at the top of the log view. If your Actions log does not show one,
click the **gear icon → View raw logs**, then use your browser's own find with
**Ctrl+F** (**Cmd+F** on a Mac). Either works for the whole task.

Search for:

```
Dupont
```

**One hit.** The gateway accepted her transfer at 14:07:41 and answered `202`, which
is the code that means *accepted*. That is why the app confirmed it on screen.

Now try to follow her payment further. Search the log for her name again and read
what the other two services said about it. They said nothing. `payment-service` and
`ledger` do not record customer names. They record account numbers, and you do not
have hers.

This is the wall people hit at 15:30 on a real afternoon. The system logged plenty.
It did not log the thing that connects one part to the next.

### 1c. Search for the trace instead

Look at the gateway line again. It carries `trace=8ac41d02`. Search for that:

```
8ac41d02
```

**Six hits, in order, across two services.** Read them top to bottom and the story
tells itself. Validation passed in 39 ms. The payment service called the ledger,
waited ten seconds, got nothing, tried once more, waited ten seconds again, and gave
up at 14:08:03.

Now count the lines that came from the ledger itself. There are none. For every other
payment that afternoon the ledger wrote a `posted` line. For this one it never wrote
anything.

### 1d. One more count

Search for `api-gateway` and note the number of matches. Then search for `posted`.

You should find **61** and **60**.

The gateway accepted 61 transfers. The ledger posted 60. Nobody had to know Marie
Dupont's name to see that one payment went missing, and nobody had to wait for her
to call. The difference between two counters was there the whole time.

### 1e. What you should have

Three things, for yourself: the six-line trace, the fact that the ledger logged
nothing, and the two numbers 61 and 60. The third one matters most, and Task 2 is
built on it.

> **The point of the task.** Nothing about the system changed between your first
> search and your second one. What changed was whether the question could be
> answered at all, and that was decided months earlier by whoever chose what to put
> in a log line. A shared trace id is not a technical detail. It is a decision that
> every team has to make at the same time, before anything goes wrong.

---

## Task 2 — Read a Dashboard That Is Lying to You *(~15 minutes)*

### 2a. Add the second workflow

**Add file → Create new file**, named exactly:

```
.github/workflows/yesterdays-numbers.yml
```

Paste this in:

```yaml
name: Yesterday's numbers

on:
  workflow_dispatch:

jobs:
  dashboard:
    runs-on: ubuntu-latest
    steps:
      - name: Publish the dashboard
        run: |
          {
            echo "## Payments platform, yesterday"
            echo ""
            echo "| Signal | Value | Threshold | Status |"
            echo "|---|---|---|---|"
            echo "| Availability | 100.00% | above 99.90% | :green_circle: |"
            echo "| HTTP 5xx responses | 0 | under 50 | :green_circle: |"
            echo "| Response time, p95 | 341 ms | under 800 ms | :green_circle: |"
            echo "| Requests served | 14,392 | no threshold | :green_circle: |"
            echo "| CPU, payment-service | 38% | under 80% | :green_circle: |"
            echo ""
            echo "Every number above is correct."
            echo ""
            echo "## The same day, from the service desk"
            echo ""
            echo "| Ticket | What the customer said |"
            echo "|---|---|"
            echo "| 88431 | The screen confirmed my transfer. The money never arrived. |"
            echo "| 88437 | The screen confirmed my transfer. The money never arrived. |"
            echo "| 88452 | The payment shows a different name from the one I typed. |"
            echo "| 88460 | The payment shows a different name from the one I typed. |"
            echo "| 88474 | The account search finds nothing and shows no error. |"
            echo ""
            echo "Transfers accepted by the gateway yesterday: **1,204**"
            echo ""
            echo "Transfers posted by the ledger yesterday: **1,197**"
          } >> $GITHUB_STEP_SUMMARY
          echo "Dashboard published. Open the Summary page of this run."
```

Commit it directly to `main`, then **Actions → Yesterday's numbers → Run workflow**.

### 2b. Read it on the Summary page

When the run finishes, click the run itself rather than the job. The dashboard is on
the **Summary** page, laid out as tables. That is deliberate. It should look like
something you have seen on a wall.

Five signals, five thresholds, five green circles. Every value is measured correctly.
Nobody faked anything.

Underneath it, the same day from the other side. Five tickets. Seven transfers
accepted and never posted.

### 2c. Why the dashboard is green

Two reasons, and you have met both of them already.

**The service answers `200 OK` when it fails.** In Lesson 3 you saw an account lookup
that returned a success code with the error written into the body of the response. A
dashboard that counts status codes reads that as a working request. The failure is
invisible to every measurement taken outside the application.

**Some failures are silent.** The repository still contains a test called
`aCommaInsideAValueBreaksTheParser`. It passes. The name `Dupont, Marie` is cut at
the comma and stored as `Dupont`, and nothing is logged, no error is returned and no
counter moves. Tickets 88452 and 88460 are that defect, seen from the customer's side.

### 2d. Pick one metric

Here are four candidates. Choose the single one that would have caught yesterday, and
write one sentence on where it would have to be measured.

| | Candidate metric |
|---|---|
| **A** | HTTP 5xx rate at the load balancer |
| **B** | Response time, p95 |
| **C** | Transfers accepted, minus transfers posted, per hour |
| **D** | CPU and memory on the payment service |

A, B and D can all be collected without the application's help, which is why they are
the ones most systems have. **C cannot.** Nothing outside the code knows what
*posted* means. Somebody has to define it, somebody has to emit it, and that costs
development time that competes with features.

That is the difference between monitoring and observability, put as a budget
question rather than a definition. And it is the number you already found by hand in
Task 1, when you counted 61 and 60.

### 2e. What you should have

The letter you chose, and your one sentence about where the metric lives. Keep it
short. You will want it for Task 3.

---

## Task 3 — Triage a Night of Alerts *(~20 minutes, no tools)*

This is the task with real work in it, and the one your colleagues will recognise on
Monday.

Below are twelve alerts that fired overnight on a payments platform. You are the
person deciding what happens to each one. There are three possible answers, and you
must give exactly one to each alert:

- **WAKE** — call a person now, at this hour.
- **TICKET** — it can wait until the morning.
- **CHANGE** — this alert should not have fired in this form. Say what you would do
  to it.

| # | Time | Alert |
|---|---|---|
| 1 | 22:14 | Certificate for the public API expires in 30 days |
| 2 | 22:47 | Disk usage on the log server is 81%, threshold is 80% |
| 3 | 23:02 | Nightly interest batch started at 23:02, usual start is 22:30 |
| 4 | 23:41 | Card authorisation success rate 91% over five minutes, usual is 99.4% |
| 5 | 00:15 | CPU on payment-service above 70% for ten minutes |
| 6 | 01:03 | No update from the external FX rate feed for 45 minutes |
| 7 | 01:30 | Backup finished with a warning, one file skipped |
| 8 | 02:12 | Zero transfers posted in the last fifteen minutes, usual for this hour is 3 to 10 |
| 9 | 02:40 | Mobile app crash rate four times the daily average, on one Android version |
| 10 | 03:26 | Database replication lag 12 seconds, threshold is 10 |
| 11 | 04:55 | 400 failed logins from a single address in two minutes |
| 12 | 05:48 | Nightly batch finished six minutes later than yesterday |

Then answer three questions in a line each.

1. **How many WAKEs did you give?** Would you defend that number to the person you
   woke up, in the morning, in front of their manager?
2. **Alert 7 has now fired every night for forty nights and has never once been
   real.** You have three options: fix the cause, delete the alert, or leave it. Say
   which one you choose. Leaving it is a decision too, and it has a price. Say what
   the price is.
3. **Alert 8 is the only one on this list that fires because nothing happened.**
   Could your own product raise an alert like that today? If not, what would have to
   exist first?

Two notes on filling it in:

- **"I don't know" is a valid answer**, exactly as it was in the Lesson 4 environment
  grid. An honest gap is a finding.
- **There is no correct sheet.** Two experienced people will disagree on at least
  four of the twelve, and the disagreement is the useful part.

Keep this page. You will need it in Lesson 5.

---

## Reflection

One question. It is for you, and nothing is collected.

> **An alert about your product fires at 02:40, and it is real. Who is woken up, who
> decides that this is an incident, and who talks to customers? Write the three
> names.** If you cannot write a name, write down who would know it.

This is the opening question of the next lesson.

---

## Bonus Challenge 🌟 — Thirty panels, sixty seconds

Grafana is the dashboard tool you are most likely to meet at work, and there is a
public sandbox that needs no account and no sign-up.

1. Open [play.grafana.org](https://play.grafana.org).
2. Pick any dashboard from the list and open it. Wait for the panels to fill.
3. Count the panels. Then imagine the phone has just rung and something is wrong.
   **How many of those panels would you look at in the first sixty seconds?**

For most people the answer is two or three, on a dashboard with thirty. The rest were
built for a different question, on a different day, by someone who is no longer
asking it. A dashboard is not a measure of how much you can see. It is a measure of
what you decided to look at first.

Nothing you click there can break anything. It is read-only and shared with the world.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| The workflow does not appear in the Actions tab | It was committed to a branch rather than `main` | Commit the file directly to `main` |
| "Workflows aren't being run on this forked repository" | Actions not enabled on the fork | Actions tab → click the green *I understand my workflows* button |
| The log shows only a few lines | The step is collapsed | Click the step name to expand it |
| There is no search box above the log | Actions log views differ | Gear icon → **View raw logs**, then Ctrl+F |
| Searching finds nothing at all | The raw log opened in a new tab and you are searching the old one | Search in the tab that actually holds the log |
| The Task 2 tables do not appear | You are looking at the job log, not the run's **Summary** page | Go back one level and click **Summary** |
| A Grafana Play panel says "no data" | Shared public instance, some panels go stale | Open a different dashboard |

---

## The Bigger Picture

Everything above was free, ran in a browser, and took an hour. In a bank the same
three ideas cost real money and real argument.

- **The trace id in Task 1** has to be agreed by every team that touches the request,
  including the ones that bought their part rather than built it. Then the logs have
  to be kept somewhere searchable, and that is priced per gigabyte per day. How long
  you keep them is a budget decision that gets made before anyone needs them.
- **The metric in Task 2** needs an owner for the definition. Which transfers count as
  posted, what happens to reversals, whether a retry is one payment or two. That
  conversation is product work, not infrastructure work, and it usually lands with
  the person who owns the backlog.
- **The twelve alerts in Task 3** imply a rota, an out-of-hours arrangement and
  somebody whose job includes deleting alerts. Most organisations have the first two.

One thing this homework deliberately does not show you: **a real tracing tool**. Every
hosted one wants an account, and several want a card eventually. You will see one in
the session instead. What is missing here is the picture, not the idea. You already
did the work by hand, which is roughly what these tools automate.

---

## Notes for the lecturer

*Not part of the participant handout. Delete before sending, or keep on a separate
page.*

- **Neither workflow has been run in GitHub Actions.** Both shell scripts were
  executed and verified outside Actions on 6 Sep 2026: 186 lines, one hit for
  `Dupont`, six for `8ac41d02`, zero ledger lines for that trace, 61 gateway lines
  against 60 `posted` lines, timestamps monotonic. The step summary renders as valid
  markdown. **Run both once in a fork before sending**, and check the Actions log
  search box against the raw-log fallback while you are there.
- **Facts to re-check about a week before sending.** Grafana Play needs no account
  (Grafana Labs' own release note, 27 Jul 2026). Actions minutes are free on public
  repositories. Add the Grafana Play row to `reference/FACTS.md` with the date you
  check it.
- **No dependency on the application code.** Unlike the Lesson 3 homework, nothing
  here needs `btraykov/add-automated-tests` merged, and nothing needs environments,
  variables or secrets. The only prerequisite is a fork with Actions enabled.
- **Nothing is collected.** The earlier files used hand-in language for the
  reflection. This one does not, which matches how the course actually runs. Confirm
  that is the direction you want, and the Lesson 2 and Lesson 3 files can be brought
  into line if so.
- **Support burden:** Task 1b is where questions will cluster, because the Actions log
  search box has more than one form and some people will land on the raw log. The
  fallback is in the handout and in the troubleshooting table. Task 2 is where people
  open the job instead of the run Summary.
- **What to demo live** (so the homework is recognition, not discovery): ninety
  seconds of Task 1, the two searches back to back. Do not show the ledger absence,
  and do not show Task 2 at all. The green dashboard next to the ticket list is worth
  more when they find it themselves.
- **Expected reaction per task:** T1 — the ledger logged nothing. T2 — option C
  cannot be measured from outside. T3 — how few of the twelve deserve a person.
- **Ties back to:** M2 L1 takeaway 6, *Logs. Logs. Logs.*, which was left as a teaser
  on purpose and is now paid off. M3 L3, both the `200 OK` on a failed lookup and
  `aCommaInsideAValueBreaksTheParser`. Task 2 is the same defect seen from the
  customer's side rather than the test's.
- **Ties forward to:** M3 L5. The reflection is that lesson's opening question, and
  the Task 3 sheet is an input to the outage simulation. Ask them to bring it.
- **Review lands in Section 0 of M3 L5**, which is in person. Two or three of the
  triage sheets on screen work well in a room, the way the rollback tables did.
  Report the tasks and what the tools do, not what people experienced.
- **The deck's homework slide must print** `M3-L4-OBSERVABILITY-AND-MONITORING.MD`,
  exactly.

---

*Observability & Monitoring Homework · Module 3, Lesson 4 · KBC IT Academy*
