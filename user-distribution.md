# Todo App – User Distribution Plan (Draft)

> **Status:** Rough draft. Percentages, spike choices, think time, and pacing are proposals for the team to discuss and agree on before finalizing.

## Purpose
This document defines how virtual users will be split across the Todo application's user stories in our planned JMeter load, stress, and spike tests, and explains the reasoning behind each choice. These are planning decisions made before building or running any tests. No production analytics were available, so all percentages are based on expected real-world user behavior and the dependencies we observed while exploring the application.

---

## 1. Application Exploration Summary

We explored the application in Chrome with DevTools (Network tab) to record each action, its HTTP requests, and its dependencies. API documentation is not exposed, so the Network tab was our source of truth.

| User Story | Action | Requests Observed | Requires Login | Depends On | Type |
|---|---|---|---|---|---|
| 1. Register | Create account | POST | No | Unique username | Write |
| 2. Login/Logout | Log in | POST, then several GETs to load data | No | Registered account | Auth + Read |
| 2. Login/Logout | Failed login | POST (error response) | No | — | Auth (negative path) |
| 2. Login/Logout | Log out | POST | Yes | Active session | Auth |
| 3. Todos | Create todo | POST, then GET | Yes | Logged-in session | Write + Read |
| 3. Todos | Edit todo (rename) | PUT, then GET | Yes | Existing todo | Write + Read |
| 4. Subtasks | Create subtask | POST, then GET | Yes | Existing todo | Write + Read |

**Authentication:** Login returns a JWT stored in a cookie named `token`, valid for 24 hours. In JMeter, an HTTP Cookie Manager in each Thread Group handles this automatically, so each virtual user keeps its own session.

**Key observation:** Every write (POST/PUT) is followed by a GET as the frontend refreshes the list. Our JMeter scripts will include these GETs so the simulated load matches what real browsers send.

---

## 2. Load & Stress Distribution

Per guidance, load and stress tests use the **same distribution**. Only the number of concurrent users changes: load uses expected traffic, and stress increases users until the application degrades or fails.

| User Story | % of Users | Breakdown Within Story |
|---|---|---|
| 1. Register | **5%** | Register → Login → Logout |
| 2. Login/Logout | **15%** | 12% successful login → view todos → logout; 3% failed login |
| 3. Todos | **50%** | 30% create todo; 20% edit todo |
| 4. Subtasks | **30%** | Create subtask (on an existing todo) |
| **Total** | **100%** | |

### Reasoning

**Register – 5%**
A user registers only once in their lifetime but returns to use the app many times. At any given moment, new signups make up a small share of traffic. It's still included because registration is write-heavy (new record plus a uniqueness check on the username) and could become a bottleneck.

**Login/Logout – 15%**
This slice represents users who log in to check their list and leave without making changes. Login also runs inside every other workflow (Stories 3 and 4 each start with a login), so authentication is exercised by roughly 95% of all users. The 3% failed-login slice tests the error path, since real users mistype passwords and the app must handle that without slowing down.

**Todos – 50%**
Managing todos is the core purpose of the application, so it gets the largest share. Creating outweighs editing (30% vs. 20%) because a todo must exist before it can be edited, and most edits are occasional renames.

**Subtasks – 30%**
Subtasks are a secondary feature that depends on todos. Not every todo gets subtasks, so this slice is smaller than Todos. Subtask requests touch nested parent/child data, which makes them a likely place for slow database queries, so it still gets a significant share.

---

## 3. Spike Test Distributions

Per guidance, each spike test increases the share of **one workflow** to simulate a sudden real-world surge. The remaining workflows are reduced so each distribution still totals 100%.

### Spike 1 – Signup Surge

| User Story | Base % | Spike % |
|---|---|---|
| 1. Register | 5% | **40%** |
| 2. Login/Logout | 15% | 10% |
| 3. Todos | 50% | 30% |
| 4. Subtasks | 30% | 20% |

**Scenario:** A promotion, launch, or class assignment sends a wave of new users to sign up at once.
**Why we chose it:** Registration is the most write-intensive single action. Every request inserts a new user and checks username uniqueness. A sudden burst tests whether the database and password hashing can keep up, and whether duplicate-check queries slow down as the user table grows.

### Spike 2 – Login Rush

| User Story | Base % | Spike % |
|---|---|---|
| 1. Register | 5% | 5% |
| 2. Login/Logout | 15% | **45%** |
| 3. Todos | 50% | 30% |
| 4. Subtasks | 30% | 20% |

**Scenario:** The start of a workday or school day, when many users open the app at the same moment to check their tasks.
**Why we chose it:** This is the most common real-world traffic spike for a productivity app. It stresses authentication (JWT creation) and the burst of GET requests that load each user's data after login.

---

## 4. Think Time & Pacing

**Think time** is the pause before each step, representing the time a real user spends reading, typing, or deciding. **Pacing** is the total time from the start of one iteration to the start of the next, which controls how often each virtual user repeats its journey.

| Flow | Steps (TT = think time before the step) | Max Think | Pacing |
|---|---|---|---|
| Register | Launch → TT 10–20s → Register → TT 5–10s → Login → TT 5–10s → Logout | 40s | **90s** |
| Login/Logout | Launch → TT 5–10s → Login → TT 10–20s → Logout | 30s | **60s** |
| Failed Login | Launch → TT 5–10s → Failed Login | 10s | **30s** |
| Create Todo | Launch → TT 5–10s → Login → TT 5–10s → Add Todo → TT 3–5s → Logout | 25s | **60s** |
| Edit Todo | Launch → TT 5–10s → Login → TT 5–10s → Add Todo → TT 5–10s → Save Edit → TT 3–5s → Logout | 35s | **60s** |
| Create Subtask | Launch → TT 5–10s → Login → TT 5–10s → Add Todo → TT 5–10s → Save Subtask → TT 3–5s → Logout | 35s | **60s** |

### Reasoning

**Think time** is based on what a real user does before each step:
- **5–10s before login:** typing a username and password. This is the same in every flow, so login behaves consistently.
- **10–20s before submitting registration:** filling out a new account form takes longer than logging in.
- **10–20s before logout (Login/Logout flow):** these users are there to read their list, so they linger.
- **5–10s before add/edit/subtask:** typing a short task name.
- **3–5s before logout after working:** the user is done and leaving.

Ranges are used instead of fixed values because real users don't all pause the same amount.

**Pacing** is calculated as:

    journey time = (maximum think times) + (steps × 2s response-time budget)
    pacing = journey time + buffer, rounded up

Maximum think times are used so pacing still fits the slowest user. The 2s per step accounts for slower responses under load. If a journey exceeds its pacing during stress testing, the next iteration starts immediately and throughput drops, which we will report as a sign of saturation.

- **Most flows use 60s pacing,** which keeps each flow's share of traffic in line with its share of users.
- **Registration uses 90s pacing** on purpose. Each registration consumes a unique username from `register_users.csv`, and longer pacing reduces how many names a test run uses (names needed = threads × test duration ÷ pacing).
- **Edit Todo and Create Subtask have the tightest buffer** (about 15s). If responses slow significantly under stress, these flows will overrun first.

---

## 5. Assumptions
- No production traffic data was available. Percentages reflect expected user behavior.
- Returning users (Stories 2–4) log in with pre-created accounts from `login_users.csv` (perfuser001–perfuser200), seeded through the register request before testing.
- Registration uses unique usernames from `register_users.csv` (newuser001–newuser200). Names are consumed once per run, so the database is restored from a backup before each test.
- Think time and pacing values (Section 4) are estimates and will be validated by timing each flow manually in the browser.
- User counts for each test type (load, stress, spike) will be defined separately in the test plan.

## 6. Deviations from User Stories
- **Edit subtask (Story 4):** The story specifies creating *and editing* subtasks, but the application does not currently support editing subtasks. Subtask testing covers creation only.
- **Delete todo / delete subtask:** Available in the app but not part of any user story, so excluded from the distribution.
