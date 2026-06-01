# Design Rationale: Stages of Survival

## Overview

This visualization explores whether a tech company's funding stage and layoffs have some pattern. The dataset covers US tech layoff events from 2020–2025, filtered to companies with known funding stages and valid layoff figures.

---

## Visual Encodings

### X Axis: Funding Stage (Band Scale)
The X axis encodes funding stage as an ordered categorical variable, from earliest (Seed) to latest (Subsidiary/Acquired). I chose this over the dataset's `Money_Raised_in_$mil` column because the funding amounts turned out to be unreliable as a continuous axis i.e. Amazon's pre-IPO raise was listed as $108M across all years and not cumulative, Google at $26M, making the scale misleading. Funding stage is a more honest encoding of "where is this company in its lifecycle?" and directly answers the question the visualization poses.

Divided plot width evenly across 15 stages with padding between bands. Dots are jittered horizontally within each band to reduce overplotting.

### Y Axis: % of Workforce Laid Off (Linear Scale)
Percentage of workforce laid off is a normalized severity measure that puts a 100 person startup and a 10,000 person public company on the same scale. Raw headcount alone would bias the chart toward large companies. 

### Dot Size: Number of People Laid Off (Square Root Scale)
Dot radius is encoded using square root so that area is proportional to headcount, not radius. 

### Color: Quadrant Membership
Color encodes which of four quadrants a dot falls in, defined by two binary thresholds:
- **Stage type**: startup (Seed through Series J) vs. established (Private Equity, Post-IPO, Acquired, Subsidiary)
- **Severity**: high layoffs (≥50% of workforce) vs. resilient (<50%)


---

## Quadrant Structure

Faint background shading in each quadrant reinforces the color encoding and helps readers orient before engaging with the legend. Dashed divider lines at the 50% Y threshold and at the Private Equity X boundary make the quadrant boundaries explicit without being visually heavy.

The 50% threshold was chosen over a data-driven value like the median because this visualization is about absolute company health, not relative ranking. A company that laid off half its workforce is in distress by any reasonable standard, that judgment should not shift depending on how other companies in the dataset performed. Using the median would mean the threshold moves as filters change, making the quadrant boundaries inconsistent and harder to interpret across views.

**Quadrant labels** were initially drawn inside the chart as text overlaid on the plot area. They were removed from inside the chart after finding they obstructed data points, and moved entirely into the right-margin legend instead.

---

## Interaction Design

### Timeline Brush (Date Range Filter)
The timeline brush allows continuous selection of any date sub-range from 2020–2025. A context bar chart below the brush shows monthly layoff totals so users can see where the data is dense before selecting.

**Alternative considered:** A year dropdown. The brush was chosen because layoffs spiked sharply in late 2022–early 2023 and again in 2025, a brush lets users isolate those waves precisely. A dropdown forces discrete annual cuts that would hide within-year spikes.


### Industry Dropdown
A secondary filter for industry lets users isolate sectors (e.g., Finance, Healthcare, Consumer). The dropdown was chosen over a multi-select or a toggle group because industries are mutually exclusive in this dataset and there are ~20 of them which is too many for toggle buttons.

### Company Click-to-Highlight
Clicking any dot highlights all layoff events for that company across all visible dates and fades everything else to lower opacity. This lets users track multi-event companies (e.g., Amazon, Meta) that appear multiple times across stages or years.

**Implementation detail:** Dots are sorted largest-first before rendering so smaller dots land on top in SVG paint order, making them clickable even when overlapping a large circle. Selected company dots are additionally `.raise()`d to intercept subsequent clicks correctly.

### Tooltip on Hover 
Hovering reveals company name, industry, stage, headcount laid off, layoff percentage, funding raised, and date.

### White Stroke on Dots
The Post IPO column became heavily overcrowded with many layoff events stacked densely in a single band. To help individual events remain distinguishable within that cluster, a thin white stroke  was added around every dot. 


## Deterministic Jitter

Dots within a band are jittered horizontally using a djb2 hash of `company + date` rather than `Math.random()`. This ensures dots don't jump to new positions on every re-render (which happens when filters change). The offset is bounded to ±37.5% of the band width so dots stay within their stage column.


## What Was Cut

- **"Unknown" stage** was filtered out of both the data and the X axis. It added noise without analytical value.
- **Non-US companies** were filtered out to make meaningful comparsons in same socioecnomic context
- **Companies with $0 funding raised** were filtered out since they couldn't be placed meaningfully on the stage axis.
- **Dot size legend** was omitted because it cluttered the right margin alongside the quadrant legend, and the tooltip already has the exact headcount on hover.

---

## References

**Data**
- Ulrike Herold, *Tech Layoffs 2020–2025*, Kaggle. https://www.kaggle.com/datasets/ulrikeherold/tech-layoffs-2020-2024
- Original data sourced from Layoffs.fyi, a community-maintained tracker of tech layoff events.




