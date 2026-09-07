<p align="center">
  <img src="masthead.svg" alt="JobRadar" width="880">
</p>

<p align="center">
  A job discovery engine that reads employers' own hiring boards first-hand,<br>
  judges every posting against a CV with a language model, and answers three questions per posting:<br>
  <b>does it fit, will they sponsor a visa, is the language requirement real.</b>
</p>

<p align="center">
  <a href="ARCHITECTURE.md"><b>Architecture and decision records</b></a> &nbsp;·&nbsp;
  <a href="MEASUREMENTS.md"><b>What was measured, and what it decided</b></a>
</p>

---

> This is the public face of a private project. It holds the engineering story: the architecture, the decision records, and the measurements behind them. The code is private while the tool is being turned into a hosted product; parts of it are shared on request.

## What it is

JobRadar started as a personal tool for one job search and grew into a system with 575,859 postings from 30 applicant-tracking platforms, 53 board and aggregator connectors and 61,804 company boards discovered automatically. Every posting is scored three ways, each layer measured before it was trusted:

1. **A keyword scorer** that gates and ranks on the user's own vocabulary. Deterministic, free, runs on every posting.
2. **An embedding similarity** to a set of short "pseudo-adverts" generated from the CV. The query side, not the model, turned out to decide ranking quality (precision at 100 went from 0.30 with the raw CV to 0.46 with adverts).
3. **A language-model judge** that reads each posting against the CV and fills a ledger — each requirement marked direct, adjacent or missing, with the ramp to close it — from which the score is computed rather than asked for. Thirteen prompt versions; the final one measured on 1,183 postings.

Visa answers come from governments, not from vibes: the public sponsor registers of six countries (Netherlands, United Kingdom, Denmark, Ireland, Portugal, Czechia) are matched by employer name, so a licensed sponsor ranks first and says so. Postings are parsed into sections so each consumer reads the part it needs. Nothing is ever deleted: a posting the gates reject is stored and flagged, so a scorer fix is a re-score, not a re-crawl.

## The engineering, in five decisions

- **Measure, then decide.** The embedding model, the query text, the keyword/embedding blend weight, the judge's reasoning effort, its temperature and its output ceiling were each chosen by a measurement with a frozen ground truth, and the losing options are kept in the record. [MEASUREMENTS.md](MEASUREMENTS.md)
- **Facts, derivations and intent are separate layers.** Facts are immutable, derivations are versioned and recomputable, user data is small and sacred. Every derived value carries the version of the thing that made it, so "refresh" means "fill where the version is behind" everywhere. [ADR-8](ARCHITECTURE.md#adr-8--staleness-is-cache-invalidation-not-version-arithmetic)
- **The judge writes a ledger, the code writes the score.** Asking a model for a number produced verdict flips on 13 of 20 postings across three identical runs; asking it to fill a per-requirement ledger and computing the number from that brought the flips under control and made every verdict explainable line by line.
- **Risks are labelled, never hidden.** A stale date, a ghost-sounding repost, a missing sponsorship statement: each is a badge under the score. A hidden posting teaches nothing, and a wrong filter is invisible. [ADR-9](ARCHITECTURE.md#adr-9--disclose-the-risk-do-not-hide-the-posting)
- **A guard for the bug the type checker cannot see.** When per-user fields moved off the shared posting row, six writers kept writing them and `tsc` passed every one: a nested relation write in the same object literal widens Prisma's argument type until sibling keys go unchecked. A text-level test now scans every posting query for a moved field, a split helper spread whole, or a local assigned from one; it was pointed at the six shapes that shipped broken and must fire on each.

## Where it is going

As of September 2026 the tool is being redesigned as a hosted, multi-user product for cross-border job seekers, with a free tier that searches the pool and paid plans that buy a daily budget of judged postings. The redesign runs on the same discipline: an evidence-graded inventory of every module, two rounds of grounded research, a one-pager that survived an adversarial review, and a plan whose first slice is a kill test, not a build.

## Stack

TypeScript throughout. Next.js App Router, Prisma, SQLite (moving to Postgres), Node's built-in test runner (618 tests), a seven-provider LLM chain behind one interface, Qwen3-Embedding for vectors. No framework for scraping: thirty ATS adapters with a pure mapper beside each fetcher, tested against fixtures.

## Author

Oytun Önal. The design decisions and the measurements are the portfolio; ask for the code.
