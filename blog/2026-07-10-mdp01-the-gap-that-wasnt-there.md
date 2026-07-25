---
layout: post
title: "The Gap That Wasn't There"
date: 2026-07-10
type: phase-update
entry_type: note
subtype: diary
projects: [hortora-engine]
tags: [retrieval, benchmark, bge-m3, precision]
series: issue-46-retrieval-quality-improvements
---

# Hortora Engine — The Gap That Wasn't There

**Date:** 2026-07-10
**Type:** phase-update

---

## What I was trying to achieve: close a 3pp precision gap between BGE-M3 and three-leg retrieval

The BGE-M3 benchmark (#36) had been showing 87% precision against a three-leg baseline of 90%. Three percentage points doesn't sound like much, but it meant the new single-model stack was objectively worse than the old three-model setup it replaced. I'd filed an epic (#46) to chase the gap — score the unscored entries, root-cause the regressions, fix what could be fixed.

## What I believed going in: that 90% was a real number

The three-leg benchmark reported 90% precision with a caveat: "87 entries from the three-leg run are unscored." The methodology was consistent — both configurations used "scored entries only" — so the comparison looked fair. I assumed the unscored entries were a minor footnote that might shift things a percentage point either way once scored.

## 138 entries and a reversed conclusion

The first surprise was the count. The to-score files only captured 30 entries — those flagged by the analysis scripts as "new" in hybrid or BGE-M3 runs. But scanning all five result files (dense-only, dense+splade, full-hybrid, three-leg, BGE-M3) against the complete baseline revealed 138 unique (entry, scenario) pairs with no score.

Scoring them was straightforward — each entry has a title and body, each scenario has a context description. The question for each: does this garden entry help someone working on this specific issue? 0 for no, 1 for yes, 2 for directly addresses it.

The distribution told the story: 100 scored 0 (irrelevant), 25 scored 1, 13 scored 2. 72% noise.

The three-leg configuration was returning more unscored entries than BGE-M3 — entries that were noise but invisible to the precision calculation because they had no score. Excluding them shrank the denominator and inflated the ratio. Once every entry had a score, three-leg dropped from 90% to 86%. BGE-M3 stayed at 87%.

The gap reversed. BGE-M3 is +1pp ahead.

## The regressions are real but unfixable

With the corrected baseline I could look at the 8 remaining regression scenarios clearly. Every one follows the same pattern: BGE-M3 returns 8 entries (same count as three-leg), but displaces relevant entries with topically adjacent noise. A CDI event entry gets swapped for a different CDI event entry. An extension deactivation entry gets replaced by a generic Hibernate configuration entry. Close enough to fool the embedding space, wrong enough to score 0.

22 relevant entries lost across the 8 scenarios, 11 noise entries gained. None of the regressions are fixable through engine-side changes — they're intrinsic to the embedding space differences between the model stacks. The cross-encoder reranking can't help either; these entries are topically close enough that the reranker also ranks them reasonably.

But the regressions are offset by improvements elsewhere. The overall number is what matters, and BGE-M3 wins by a point.

## What it means for the backlog

The epic's premise was wrong — there's no precision gap to close. I closed #46, closed the DOMAIN_ABSENCE investigation (#49) as not worth pursuing (it's actually a POLYSEMY problem at 50% on a single scenario, and corpus enrichment won't fix polysemy). The two blocked items (#50, #51) stay open as standalone issues — they'll complete themselves when the neocortex per-leg separation work ships, but the urgency is gone.

The measurement gotcha is worth remembering: excluding unscored entries from precision is a methodology that looks consistent but silently biases toward whichever configuration produces more unscored noise. The bias is a second-order effect of uneven score coverage — a reviewer checking the methodology would approve it.
