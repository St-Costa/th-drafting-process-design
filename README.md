<h1 align="center">TH Drafting: process design</h1>

<p align="center">
  <img alt="Repo" src="https://img.shields.io/badge/repo-process%20design%20only-lightgrey">
  <img alt="Code" src="https://img.shields.io/badge/code-private-lightgrey">
  <img alt="Python" src="https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white">
  <img alt="LiteLLM" src="https://img.shields.io/badge/LiteLLM-multi--provider-6E56CF">
  <img alt="Claude" src="https://img.shields.io/badge/Claude-Sonnet%20%2B%20Haiku-D97757">
  <img alt="Tests" src="https://img.shields.io/badge/tests-175-2ea44f">
</p>

This repo contains only the process design. The code, prompts and style rules are private.

## Status

Used in production on a full novel. The publishing project has since closed,
so the tool is no longer developed.

## Contents

- [The problem](#the-problem)
- [What I did](#what-i-did)
- [How the pipeline works](#how-the-pipeline-works)
- [Results](#results)
- [Tests and evals](#tests-and-evals)
- [Why there's no code](#why-theres-no-code)

## The problem

A novelist and I tried to start a publishing house. His job was to design the story
structure and the prose style for our target readers.

Once that was done, a single book of about 50 chapters (a length chosen for our target)
took **3–4 months** from zero to a corrected first draft.

I mapped the process, and one step took most of the time: turning the structured scene
outlines into a first draft. It was also the least valuable step: going from first draft
to corrected draft was much faster and fixed most of the problems, and beta readers then
polished the corrections.

We tried the existing tools first:

| | GPT-4o via API, Novelcrafter | What we needed |
|---|---|---|
| **Context** | loses track across 50 chapters | every chapter knows the characters, places and facts already used |
| **Style** | follows the rules loosely | follows a strict, house-specific rulebook |
| **Control** | one prompt, one answer | each rule checked, and the draft sent back when it fails |

## What I did

- **Automated the first draft.** The writer starts from a generated draft and spends
  his time correcting it, instead of writing everything from a blank page.
- **Turned the novelist's method into precise rules.** Together with him, I wrote his
  style and structure down as explicit rules, so that a model can follow them and a
  check can verify them. The rulebook kept growing as we found new problems. About 30%
  of the rules are checked by plain code (word counts, banned patterns); the rest are
  checked by a model.
- **Kept the model out of the control flow.** The order of the steps, the approval logic
  and the retry limits are fixed in code. The model only writes and reviews, plus one
  choice per chapter: which reviewers to run and how many rewrite rounds to allow.

## How the pipeline works

```mermaid
flowchart TD
    START(["▶ START · new chapter"]) --> IN
    IN[/"📄 INPUTS<br/>Story structure and chapter summaries · Character, place and object sheets<br/>Style rulebook · Facts already used in earlier chapters"/]

    IN --> S["⚙️ CODE<br/>Select the context for this chapter"]
    S --> P["🤖 AGENT<br/>Plan the chapter: which reviewers, how many rounds"]
    P --> D["🤖 AGENT<br/>Write the first draft"]
    D --> R

    subgraph R["🤖 AGENTS · review in parallel"]
        direction LR
        R1[Style]
        R2[Continuity]
        R3[Characters and<br/>point of view]
        R4[Logic and<br/>reader's view]
    end

    R -->|changes needed| W["🤖 AGENT<br/>Rewrite only the flagged parts"]
    W -->|limited rounds| R
    R -->|all approve| C["⚙️ CODE<br/>Automatic rule checks"]
    C --> F["🤖 AGENT<br/>Final polish, then re-checked"]
    F --> L["⚙️ CODE<br/>Record the facts used in this chapter"]
    L -.->|next chapter| IN
    L --> H["👤 WRITER<br/>Corrects the chapter"]
    H --> END(["✅ END · chapter done"])

    classDef input fill:#3a3a3a,stroke:#bfbfbf,color:#fff
    classDef terminal fill:#111,stroke:#fff,stroke-width:3px,color:#fff
    class START,END terminal
    classDef code fill:#1f4e79,stroke:#9cc3e6,color:#fff
    classDef llm fill:#a3541c,stroke:#f4b183,color:#fff
    classDef human fill:#2e6b30,stroke:#a9d18e,color:#fff
    class IN input
    class S,C,L code
    class P,D,W,F,R1,R2,R3,R4 llm
    class H human
```

How the loop stays reliable:

- Each reviewer checks one thing and never reads the other reviewers' reports, so one
  opinion doesn't sway the others. A chapter passes only if every reviewer approves and
  the automatic checks pass too.
- A missing or malformed review counts as "needs changes". A review that quotes text
  that isn't in the draft is thrown out. When the rewrite rounds run out, the chapter
  goes to the writer with a flag.
- After each chapter, the facts it used (places, objects, details) are recorded, so the
  next chapters don't repeat or contradict them.

How it stays cheap:

| | |
|---|---|
| **Models** | Haiku for narrow checks, Sonnet for writing and rewriting |
| **Caching** | the fixed part of each prompt is cached: from the second round on, most input tokens cost ~10% |
| **Parallel reviews** | reviewers run at the same time, since they don't depend on each other |
| **Tracking** | tokens and cost logged for every chapter |
| **Executors** | same pipeline through the API, through an agent harness with no API key, or by hand to debug a step |

## Results

| | Before | After |
|---|---|---|
| Time per book (~50 chapters) | 3–4 months | **~2 weeks**: 1 week of structuring, 1 week of corrections |
| Writer's correction time, whole book | – | **~20 hours** |
| What the writer spends time on | writing drafts | structuring and correcting |

Measured on one novel. The drafts still need the writer, but fewer corrections as models improve,
because how well the rules are followed depends on the model.

## Tests and evals

| | |
|---|---|
| **Tests** | 175 automated tests on the pipeline |
| **Evals** | 15 cases built on real material, covering 11 reviewer roles. Each case lists what a good reviewer must find; every prompt change is scored against them |
| **Decisions** | 12 written architecture decisions, including rejected options. Example: merging all reviewers into one call was cheaper, but the reviewers influenced each other, so I dropped it |

## Why there's no code

The prompts, the rulebook and the orchestration are the core of the work and belong to
the publishing project, so they stay private.