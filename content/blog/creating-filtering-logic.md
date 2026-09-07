---
title: Creating Filtering Logic
description: A step-by-step refactoring journey from a naive if-chain to a full filter engine, using a property search example.
date: 2026-09-07
tags: java design-patterns
---

*Placeholder post — working through this roadmap step by step. Will fill in code and writeup as each stage is done.*

## The setup

Built around `Property`, `GeoPoint`, some sample test data, and a `SearchRequest` with a `search()` method left as a `TODO` — that's Step 1, the naive if-chain.

Expected result for the sample query in `main()`: **P1** and **P5** match (price 20L–40L, 2 or 3 BHK, has gym).

Here's the roadmap — build each step, test against the sample data, then evolve to the next. Don't jump ahead; the point is feeling each pain before fixing it.

## Step 1 — Naive if-chain (starter given)

Fill in `search()` with one big loop and `if` checks. Get it working and correct first.

Once it works, try adding a 5th filter (say, furnishing) — notice how much you have to touch.

## Step 2 — Extract `Filter` interface + concrete strategy classes

Create `Filter` with `boolean isSatisfiedBy(Property)`. Write `PriceFilter` and `BhkFilter` as separate classes. Rewrite `search()` to loop and check `filter1.isSatisfiedBy(p) && filter2.isSatisfiedBy(p) && ...`.

Then ask yourself: can I express "2BHK OR 3BHK in Bandra, OR 4BHK anywhere"? You can't yet — that's the next problem.

## Step 3 — `CompositeFilter` (AND/OR nesting)

Make `CompositeFilter implements Filter`, holding `List<Filter>` + an operator. Rebuild the sample query as a tree instead of a flat chain.

Then notice: you probably wrote `PriceFilter` and would need a near-identical `AreaFilter`. That duplication is the next thing to kill.

## Step 4 — `FieldExtractor<T>` + generic `RangeFilter`/`MultiSelectFilter`

Replace `PriceFilter`/`BhkFilter` with one `RangeFilter` and one `MultiSelectFilter`, each taking a `FieldExtractor` (method reference like `Property::getPrice`).

Then notice: constructing these by hand, wiring the right extractor each time, is fiddly and easy to get wrong.

## Step 5 — `FilterCriterion` + `FilterFactory`

Build a criterion DTO (type + field + values) and a factory that turns a criterion into the right `Filter` + extractor combo.

## Step 6 — `FilterRequestBuilder`

Wrap "loop over criteria, call factory, add to composite" into a fluent builder.

## Step 7 — `FilterEngine`

Extract the "apply filter to list of properties" loop into its own tiny class, decoupled from construction.

## Step 8 — `BookingPlatform`

Own the property list, wire builder + factory + engine behind one `search()` method.

---

*Next up: Step 1 code and what broke first.*
