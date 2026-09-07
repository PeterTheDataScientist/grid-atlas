# What did not work

An honest log, added to as things break rather than written retrospectively once everything works.

---

## Line-based parsing assumed the area and its time were on the same line

**What was tried.** The first parser read each line of a page's visible text and expected an area name and a time range together, as they appear in a written sentence.

**What happened.** Every notice in a table produced zero records. HTML tables flatten to one cell per line, so the parser saw a bare area name (no time, skipped) then a bare time range (no area, discarded). On the test fixture it found 5 candidate notices and parsed 0.

**Why it matters.** It would have failed silently in production, reporting "0 new records" and looking like a quiet day rather than a broken parser. It was caught only because the fixture test asserted specific expected areas rather than "some notices were found".

**The lesson.** Assert on content, never on counts. A test checking "more than zero" would have passed against a parser inventing them.

---

## Lint passed locally and failed on CI, and the thing it caught was real

**What happened.** `ruff` passed locally and failed on the first CI run with 8 errors. The two environments resolved different versions and the newer one's defaults included rules the older one did not.

**Why it matters, and it is not the lint.** The rule was DTZ001, naive `datetime` construction. Notices are published in Harare local time; satellite granules are in UTC. A naive datetime makes that join silently wrong by two hours: an outage announced for 22:00 local compared against the 22:00 UTC pass, which is midnight local. It would still produce a number, and the number would be nonsense.

**The fix.** Timezone-aware end to end, ruff pinned, and the rule set named explicitly rather than inherited from whatever version resolves.

---

## The first live run produced exactly one record, and it was a sentence about geysers

**What happened.** An early text-parsing prototype, run against published utility web pages to see whether announced supply windows could be harvested as a second signal, produced one "record": area `"Why are bills not going down despite all the shedding?"`, window 12:00 to 22:00. The source line was a paragraph explaining that a geyser "contributes about 60-70% of the total bill". The regex matched `60-70` and `% 24` wrapped it into a plausible outage window.

**The fix.** A validation pass between matching and accepting: a bare number range is only a time if it carries a colon, an am/pm marker, or the 4-digit HHMM form. Percentages rejected. Written hours above 23 rejected before wrapping. The real paragraph is now a regression test.

**The second finding, which mattered more.** With the false positive gone the run returned nothing at all, from any source. There is no machine-readable supply schedule to harvest, so the announced-window idea cannot be a primary signal and the satellite measurement became the record rather than a check on someone else's record.

**The lesson.** Test against the real source before building on an assumption about it. The synthetic fixture proved the parser worked. Only the live run proved there was nothing to parse.

---

## The measurement box was measuring coastline

**What was tried.** Per city, take the median radiance of a fixed 0.3 degree box around the centre. Simple, uniform, defensible-sounding.

**What happened.** Cape Town read **3.6** nW/cm2/sr against Johannesburg's **19.1**, which says Cape Town is five times dimmer than Johannesburg. That is obviously false, and it was the number the first published dashboard was built on.

The cause: 27 percent of Cape Town's box is ocean and mountain, sitting near zero. With more than a quarter of pixels dark, the **median lands in the dark half**. Cape Town's 90th percentile is 28.6 against Johannesburg's 33.4, so the two cities are comparable and the statistic was the entire problem. Harare had the same defect more mildly, 18 percent dark pixels dragging its median to 3.2 against a 90th percentile of 10.3.

**Why it is worth writing down.** Nothing about the output looked broken. Every city produced a plausible number, the map rendered, the ranking sorted. A statistic can be wrong in a way that is invisible unless you check a case where you already know the answer. Cape Town was that case only because "Cape Town is dimmer than Harare" is obviously absurd to anyone who has seen either city.

**The fix.** Stop sampling a geographic square and start sampling each city's own lit footprint, derived from a per-pixel 90th-percentile envelope of its own observations. Coastal cities, cities against escarpments and cities with rural fringes are all corrected by the same mechanism, and the mask comes from the data rather than a hand-drawn boundary.

**The second fix, structural.** The pipeline now stores pixel arrays rather than summary statistics. The first version stored medians, so changing the statistic meant re-downloading everything. A method change should be a re-aggregation, not a re-download.

**The lesson.** Sanity-check a new metric against a case where you already know the answer, before building anything on top of it. The correct check is not "do the numbers look reasonable" but "is there a specific value here I can independently say is wrong".

---

## The weekly job quietly deleted two thirds of the dataset for three weeks

**What happened.** On 16 August the series reached 989 observations across 39
new-moon nights spanning 2022 to 2025. The next scheduled run, on 17 August,
published 363. The one after that 374, then 337, and 337 again on 7 September.
Nobody noticed, because each run looked exactly like a successful run: it
completed, it committed, and the dashboard rendered.

**Why.** Two decisions that are each defensible alone and destructive together.

The pixel arrays are too large for git, so they live in the GitHub Actions cache.
That was a reasonable call and it is documented. But the Actions cache is
ephemeral: GitHub evicts entries that have not been read for a week, and evicts
under repository size pressure. So the pixel store, which is the only input the
published series is derived from, is not durable.

And `aggregate.py` wrote `data/places.json` fresh on every run. Not merged, not
appended, replaced. So the moment the cache was evicted, the next run rebuilt
from only the dates in its rolling 18 month window and overwrote three and a half
years of record with a few months of it.

**Why merging the output was the wrong fix.** The obvious repair is to union the
old series with the new one. That produces an internally inconsistent file. The
per-city statistics (`normal_radiance`, `mean_dark_share`, and the 90th-percentile
lit footprint itself) are all derived from the pixel set the run actually held.
Union the nightly series from two different pixel sets and the summary numbers no
longer describe the series sitting beside them. The file would look richer and be
less true, which is worse than the bug.

**The fix, in two parts, neither of which is a merge.**

First, the aggregation refuses to shrink. If a run produces materially fewer
observations than the file it is about to overwrite, it raises rather than
publishes. This is the same rule this codebase applies to every other source: a
thin pixel cache is not evidence that fewer nights exist, exactly as a network
timeout is not evidence that no outage happened. The threshold is one constant
and an explicit `--allow-shrink` exists for the case where a reduction is real.

Second, a missing cache now triggers a full rebuild rather than a partial one.
The rolling 18 month window is correct when the store survived and `extract.py`
only has to top it up. When the store is gone, that same window silently rebuilds
a fraction of the archive, so the workflow now checks for `pixels.npz` and starts
from the beginning of the archive when it is absent.

**How it was found.** Not by monitoring. By comparing this repository against a
duplicate created by accident, before deleting the duplicate, and noticing the
duplicate held 989 observations where the live one held 337. The guard exists so
that the next occurrence announces itself instead of waiting for someone to
compare two copies by hand.

**What this says about the design.** DESIGN.md argues for append-only JSONL in git
because "a dataset that can silently vanish is not a dataset", and then puts the
only copy of the pixel store in a cache that silently vanishes. Stating a
principle in a design document does not implement it. The principle was right and
one component was quietly exempt from it.
