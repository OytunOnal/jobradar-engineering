<p align="center"><img src="masthead.svg" alt="JobRadar" width="880"></p>

![The radar filtered to employers on a public sponsor register: the six register countries in the country row, and a London posting judged strong with its sponsor answer](screenshot.png)

**This repository is deliberately thin: the engineering record of a private project, not its code.** The software is being turned into a hosted product; parts of the code are shared on request.

Reads employers' own hiring boards first-hand (not an aggregator's copy), judges every posting against your CV, and answers three questions per posting: **does it fit, will they sponsor a visa, is the language requirement real.**

| | |
|---|---|
| **Who it is for** | People looking for a job in a country where they will need a visa, who have to know before applying whether the employer can sponsor and whether the fit is real. |
| **What it does now** | **You cannot use it yet.** Today it is the author's local, single-user tool. Visa answers cover six countries with a public sponsor register: the Netherlands, the United Kingdom, Denmark, Ireland, Portugal and the Czech Republic. The pool holds about 576,000 postings, of which about 93,000 are live under the tool's current region gate (September 2026; the counts and their sources are in [Measurements § Coverage](MEASUREMENTS.md#coverage-as-counted-on-2026-09-07)). Each posting gets three scores: a keyword score (your own vocabulary), an embedding similarity (how close the posting is to short adverts written from your CV) and a judge's verdict (a language model that marks every requirement line of the posting against your CV before it scores). The hosted, multi-user version is being built; this line is replaced when its first version ships. |
| **How to run it** | Not from here: the code is private, so there is nothing to clone. The measurements and the decision records below are the deliverable; ask for the code. |
| **Feedback** | [Issues on this repository](https://github.com/OytunOnal/jobradar-engineering/issues); to ask for the code, write to the author via [github.com/OytunOnal](https://github.com/OytunOnal). |
| **Licence** | The text, diagrams and measurements may be read, linked and quoted with attribution; other reuse needs permission ([LICENSE.md](LICENSE.md)). The software is not part of this repository and is not licensed by it. |

<p align="center">
  <a href="MEASUREMENTS.md"><b>Measurements</b></a> &nbsp;·&nbsp;
  <a href="ARCHITECTURE.md"><b>Architecture and decision records</b></a>
</p>

## How it works

```mermaid
%%{init:{"theme":"base","themeVariables":{
"primaryColor":"#1f2631","primaryTextColor":"#f4f6f9","primaryBorderColor":"#3d4858",
"lineColor":"#3fd6c6","secondaryColor":"#1f2631","tertiaryColor":"#151a22",
"clusterBkg":"#151a22","clusterBorder":"#2c3541","edgeLabelBackground":"#151a22","titleColor":"#5b6b7d"
},"flowchart":{"wrappingWidth":300,"curve":"basis"}}}%%
flowchart TD
  DISC["<b>Discovery</b><br/>Common Crawl · Wayback<br/>links in postings · name guesses"] -- "probed live" --> BOARDS[("61,804 company boards<br/>30 ATS platforms")]
  BOARDS --> POOL
  AGG["53 boards and aggregators"] --> POOL
  POOL[("<b>The pool</b> · 576k in the database, 93k live<br/>nothing is ever deleted")]
  POOL --> KW["<b>keyword score</b><br/>the user's own vocabulary"]
  POOL --> EMB["<b>embedding similarity</b><br/>to adverts written from the CV"]
  KW --> Q["queue · keyword 0.2 / similarity 0.8"]
  EMB --> Q
  Q --> JUDGE["<b>the judge</b><br/>a ledger per requirement<br/>fit · visa · language"]
  REG[("6 sponsor registers<br/>NL GB DK IE PT CZ")] --> JUDGE
  JUDGE --> RADAR["<b>Radar</b><br/>ranked, risks labelled"]
  classDef key fill:#0e2b27,stroke:#3fd6c6,stroke-width:2px,color:#f4f6f9
  class BOARDS,POOL,RADAR key
```

Three scores per posting, each measured before it was trusted: a **keyword score** (deterministic, free), an **embedding similarity** to short adverts generated from the CV (the query text, not the model, decided ranking quality), and a **judge** that fills a per-requirement ledger from which the score is computed rather than asked for. Visa answers come from six governments' sponsor registers, matched by employer name.

## Five decisions

| | The decision | What forced it |
|---|---|---|
| 1 | **Measure, then decide.** Model, query text, blend weight, reasoning effort, temperature, ceiling: each chosen against a frozen ground truth; the losers are kept. | A "reasonable default" cost a week twice. [Measurements](MEASUREMENTS.md) |
| 2 | **The judge writes a ledger; the code writes the score.** | Asking for a number flipped 13 of 20 verdicts across identical runs. |
| 3 | **Facts, derivations and intent are separate layers.** Every derived value carries the version of what made it; "refresh" is "fill where the version is behind". | Overwriting in place destroyed the calibration data once. [ADR-8](ARCHITECTURE.md#adr-8--staleness-is-cache-invalidation-not-version-arithmetic) |
| 4 | **Risks are labelled, never hidden.** | A hidden posting teaches nothing; a wrong filter is invisible. [ADR-9](ARCHITECTURE.md#adr-9--disclose-the-risk-do-not-hide-the-posting) |
| 5 | **A text-level guard for the bug the type checker cannot see.** | Six writers kept writing moved fields and `tsc` passed every one. |

<p align="center"><img src="blend-sweep.svg" alt="Keyword weight sweep: the curves are flat to 0.3 and fall after; 0.2 was chosen" width="720"></p>

## The data model, in one picture

```mermaid
%%{init:{"theme":"base","themeVariables":{
"primaryColor":"#1f2631","primaryTextColor":"#f4f6f9","primaryBorderColor":"#3d4858",
"lineColor":"#3fd6c6","secondaryColor":"#1f2631","tertiaryColor":"#151a22",
"clusterBkg":"#151a22","clusterBorder":"#2c3541","edgeLabelBackground":"#151a22","titleColor":"#5b6b7d"
},"flowchart":{"wrappingWidth":220,"curve":"basis"}}}%%
flowchart TD
  subgraph FACTS["facts · immutable"]
    JOB["Job · the posting"]
    JC["JobContent · the text"]
  end
  subgraph DERIVED["derivations · versioned, recomputable"]
    JE["JobEmbedding · builtFrom"]
    KSH["KeywordScoreHistory · scorerVersion"]
    LJH["LlmJudgmentHistory · promptVersion"]
  end
  subgraph MINE["one user's view · small and sacred"]
    UJ["UserJob · similarity, verdict, pursuit"]
    UP["UserProfile · CV, adverts, two stamps"]
  end
  JOB --> JE --> UJ
  JOB --> KSH
  JOB --> LJH --> UJ
  UP -- "queryStamp" --> UJ
  UP -- "profileStamp" --> LJH
  classDef key fill:#0e2b27,stroke:#3fd6c6,stroke-width:2px,color:#f4f6f9
  class UJ,UP key
```

Facts never change. Derivations carry the version of what produced them. A user's data is small, keyed to the user, and stamped twice: an advert edit re-ranks the pool in seconds; a CV edit says how many verdicts go stale and what re-judging costs before it moves. The radar's list query over half a million rows measures 4 ms. [Full records →](ARCHITECTURE.md)

## Where it is going

Since September 2026 the tool is being redesigned as a hosted product for cross-border job seekers: a free tier that searches the pool, paid plans that buy a daily budget of judged postings. Same discipline: an evidence-graded inventory, grounded research, a one-pager that survived an adversarial review, and a plan whose first slice is a kill test, not a build.

**Stack.** TypeScript · Next.js · Prisma · SQLite → Postgres · Node's test runner · a seven-provider LLM chain behind one interface · Qwen3-Embedding.

**Author.** [Oytun Önal](https://github.com/OytunOnal). The decisions and the measurements are the portfolio; ask for the code.
