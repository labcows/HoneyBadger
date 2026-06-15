# HoneyBadger: Building a Financial Transactions Database from Scratch, in C

> A from-the-ground-up build of the core of a OLTP database — the part
> that records money movements and refuses to lose or corrupt them, even when
> the server dies mid-transaction.

<!-- Badges go here once the code repo + CI are public. Example:
![CI](https://github.com/<you>/<repo>/actions/workflows/ci.yml/badge.svg)
![C11](https://img.shields.io/badge/C-C11-blue) -->

---

## Project status

🚧 **Development in progress — Phase 0 of 3.** Foundations and data model are in
place; the transaction engine and durability layer are next.

✅ = built and tested · 🚧 = in progress · 📋 = planned and designed, not yet written

| Phase | What it delivers | Status |
|---|---|---|
| **Phase 0** — Foundations | Build system, testing framework, data modeling(account & transfer) | 🚧 In progress |
| **Phase 1** — Core engine | Create accounts, make a single payment, enforce the balancing rules | 📋 Planned |
| **Phase 2** — Two-phase payments | Pending → Posted / Cancel / Gaurantee the full  data integrity | 📋 Planned |
| **Phase 3** — Durability | Batching, Write-Ahead Log, Crash Recovery | 📋 Planned |

**Done so far:**

- ✅ An automated test framework using MakeFile.
- ✅ Continuous integration (CI) using GitHub Actions that re-runs every test on every change.
- ✅ A development environment that runs identically on any machine via Docker.
- ✅ A dedicated money type with overflow protection (so a balance can never
  silently wrap around to a wrong number).
- ✅ The two core data records — **Account** and **Transfer** — each a precisely
  128-byte fixed-size structure, verified at compile time.

---

## Table of Contents

1. [Motivations](#1-motivations)
2. [What it is, in one minute](#2-what-it-is-in-one-minute)
3. [The two questions driving the design](#3-the-two-questions-driving-the-design)
4. [How the system works](#4-how-the-system-works)
5. [Design decisions and trade-offs](#5-design-decisions-and-trade-offs)
6. [The roadmap](#6-the-roadmap)
7. [How to build and run it](#7-how-to-build-and-run-it)
8. [What I'm learning](#8-what-im-learning)


## 1. Motivations

I worked on a **payment backend service** — the kind of system that moves money
between accounts. While fixing problems in that system, one worry kept coming
back to me: **a server can go down at any moment.**. When that happens to a system that handles money, "oops, my bad" is not a good answer. The record of who paid whom has to survive, and it has to be *clean*. **No half-finished payments, no money that vanished or got duplicated.**

That curiosity turned into a question: **what kind of system can guarantee that
money records stay reliable — close to 99.9% trustworthy — even when the
machine running it dies silently?**

This project is my attempt to understand that by building it myself, from the
ground up, in the C programming language. Rather than use an existing database,
I'm rebuilding the *core* of one — the part that records transactions and
protects them — so I can understand exactly how reliability is achieved instead
of just trusting that it is.

> 💡 **What's a "ledger"?**
> A ledger is a master record-keeping system that tracks and organizes financial transactions, assets, and liabilities.
> Simply speaking, it is a record book of money movements. Like the running
> balance in your bank app, listing every deposit and withdrawal. A *digital*
> ledger is the software version of that book. This project builds one whose
> defining feature is that it **never loses or corrupts its records.**


## 2. What it is, in one minute

This project is a small, fast, **in-memory transaction ledger**. The database
focused on one job: recording money movements between accounts correctly and
durably.

- **It uses double-entry bookkeeping** — the centuries-old accounting rule that
  every transaction touches two accounts (money leaves one, arrives in another)
  so the books always balance. This can provide the reliability and cridibility of the financial system and its record.

- **It writes every change to a permanent log on disk *before* applying it**,
  so that if the power dies, the system can rebuild its exact state when it
  restarts.

- **It keeps the accounts and balances in the computer's memory (RAM)** so that reads and writes are extremely fast.


It is inspired by [TigerBeetle](https://tigerbeetle.com), a real
financial-transactions database, and reuses its core ideas at a smaller scale
for learning and applying to build reliable and robust software for clients.

> 💡 **"In-memory" vs "on disk" — why both?**
> RAM is fast but forgets everything when the power goes off. A disk is slow but
> remembers. This system uses RAM for speed *and* the disk for safety, getting
> the best of both: it serves answers from memory, but every change is safely
> written to disk first.


## 3. The two questions driving the design

Everything in this project comes back to two questions I wanted to answer with
working code, not opinions:

### Question 1 — Speed: how do you process a flood of payments quickly?

The answer is **batching.** Instead of writing each payment to disk one at a
time, the system collects many payment records and writes them down in a batch.

### Question 2 — Integrity: how do you make sure money is never lost or invented?

**Double-entry bookkeeping** — every transfer subtracts from one account and
  adds the same amount to another. If the totals don't match, something is
  wrong, and the system can catch it. Money cannot appear from nowhere.

**Write-ahead logging (WAL)** — before changing any balance, the system first
  writes changes to a permanent log. If the system crashes halfway, it replays the log on restart and finishes (or safely discards) what was in flight.


## 4. How the system works

Here is the path a single payment takes through the system:

```
                 A payment comes in
                        │
                        ▼
        ┌───────────────────────────────┐
        │  1. Validate it                │   Is the amount valid? Do both
        │     (is this request legal?)   │   accounts exist? Same currency?
        └───────────────┬───────────────┘
                        ▼
        ┌───────────────────────────────┐
        │  2. Write to the log (WAL)     │   Record the intent on disk and
        │     BEFORE touching balances   │   wait until it's truly saved.
        └───────────────┬───────────────┘
                        ▼
        ┌───────────────────────────────┐
        │  3. Apply it in memory         │   Move the money: subtract from one
        │     (update both balances)     │   account, add to the other.
        └───────────────┬───────────────┘
                        ▼
                  Payment recorded


   If the power dies anywhere above:
   on restart, the system reads the log and rebuilds the exact balances.
```

The system is **single-threaded** — it processes one batch of payments at a
time, in order, with no two things happening at once. It's actually a deliberate reliability choice (explained in the next section): when only one thing happens at a time, an entire category of "two payments collided" bugs simply cannot occur.

> 💡 **Two kinds of transfer this system handles**
> - **Instant payment:** money moves immediately and is final.
> - **Two-phase payment:** money is first *reserved(pending)* (like a hotel placing a
>   hold on your card), then later either *captured(posted)*
>   or *released(cancelled)*. This is how real authorization →
>   settlement flows work, and the system supports full capture, cancellation,
>   and *partial* capture.


## 5. Design decisions and trade-offs

| Decision | Why | What I gave up |
|---|---|---|
| **Written in C** | Direct control over memory and layout. Forces me to understand what higher-level languages hide. | Convenience and safety nets — I have to manage details by hand. |
| **Single-threaded** | Eliminates an entire class of concurrency bugs by construction; makes behavior predictable and testable. | Can't use multiple CPU cores for the core engine. |
| **Allocate all memory once, at startup** | No surprise slowdowns or failures from memory allocation while running; predictable performance. | Capacity limits are fixed when the program starts. |
| **Fixed 128-byte records** | Predictable memory layout, cache-friendly, simple math to find any record. | Less flexible than variable-size records. |
| **Double-entry bookkeeping** | Makes "money appeared/vanished" bugs structurally impossible, not just unlikely. | Slightly more bookkeeping per transaction. |
| **Log to disk before applying (WAL)** | The crash-safety guarantee depends on it. | Every change pays the cost of a disk write — mitigated by batching. |


## 6. The roadmap

The project is scoped in versions, so there's always a working, demonstrable
milestone:

- **v0 — the correct core** *(Phases 0–1)*: accounts, single payments, and the
  balancing rules that keep them reliable.
- **v1 — the durable system** *(Phases 2–3)*:
  reserve/capture/cancel payments, the write-ahead log, batching, and full crash
  recovery. At this point you can kill the process and the data survives intact.
- **v2 — the proof** *(future)*: a "deterministic simulation tester" that
  injects random crashes and disk failures from a single seed and verifies the
  system always recovers correctly, plus performance benchmarks with real
  numbers.

> 💡 **What's a "deterministic simulation tester"?** *(a future goal)*
> It's a way of testing by simulating thousands of disasters — crashes,
> corrupted disks, bad timing — in a controlled world where every run can be
> *replayed exactly*. If a bug appears, you can reproduce it perfectly from a
> single number (a "seed"). It's how the most reliable databases prove they're
> reliable.

A detailed, commit-by-commit plan lives in the project's `ROADMAP.md`.

---

## Todos

## 7. How to build and run it

## 8. What I'm learning

---

_This project is built for learning, inspired by [TigerBeetle](https://tigerbeetle.com).
It is a personal reimplementation of core ideas, not affiliated with or derived
from TigerBeetle's source code._
