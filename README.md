# Foodpanda APAC Customer Feedback Analysis

End-to-end analysis of Foodpanda customer reviews across APAC, built in two iterations: **v1** used traditional star-rating sentiment and keyword frequency analysis across 4.6 million reviews in 11 markets; **v2** added an LLM extraction layer to turn raw review text into structured, actionable service issues across 279,723 recent reviews in 9 markets.

**Interactive dashboards (Tableau Public):**

- **[v1 — Market Health & Keyword Analysis](https://public.tableau.com/views/FoodpandaAPACCustomerFeedbackAnalysis2024-2026/Story1?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)**
- **[v2 — Aspect-Based Issue Extraction](https://public.tableau.com/views/FoodpandaAPACCustomerFeedbackAnalysisV2/Story1?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)**

---

## Why two versions

**v1** answered: *which APAC markets show signs of customer experience strain, and what service dimensions (food, rider, restaurant) are driving sentiment in each?* Sentiment came from explicit star ratings, and unigram keyword frequency surfaced what customers mention most. This worked as a first pass — it exposed market-level patterns and the dominant complaint themes — but keywords proved too vague to act on. A unigram cannot distinguish "late delivery" from "not late at all", negation is lost entirely, and a keyword like "small" says nothing about whether the platform can fix it.

**v2** closed that gap with an AI extraction layer: each review is read by Google's Gemini 2.5 Flash Lite and labeled with structured aspects, sentiment, and quoted evidence. The framing is deliberately narrow — rather than scoring overall satisfaction (which the star rating already captures), the extraction targets **foodpanda-controllable issues**: things the platform's operations team could act on. Vendor-side complaints like bland food or small portions are excluded by design; they're the restaurant's responsibility, not the platform's.

The v2 architecture is hybrid: the LLM handles qualitative interpretation (reading a review and deciding which aspects are mentioned); deterministic Python handles all aggregation, filtering, and output.

---

## v1 — Sentiment & Keyword Analysis (4.6M reviews, 11 markets)

### Key findings

**The customer pain point is restaurant-side, not delivery-side.** Riders are rated 4.78 on average; food is rated 3.33 — a consistent 1.45-point gap across all 11 markets. Delivery execution is not the problem.

**Customer sentiment is polarised, not average.** 26% of reviews are 1-star and 45% are 5-star, with only 28% in the 2–4 star middle. A J-shaped distribution that makes the average rating misleading — the 1-star rate should be tracked as its own KPI.

**Small portions are the #1 complaint APAC-wide.** Appears as "small" in Singapore (2,614), Philippines (2,775), Malaysia (4,202) and as "kecik" in Malaysia (4,117). The most consistent complaint across every market analysed.

**Malaysia holds the strongest customer loyalty signal in APAC** — "terbaik" (26,638), "repeat" (23,830), "terima kasih" (17,440+). But also the strongest disappointment signal — "kecewa" (4,521), "disappointed" (3,980). A base worth protecting through a recovery flow before silent churn.

**Taiwan is the APAC top performer (3.85 average) and is being divested to Grab in H2 2026.** There is a window to document what works there before the transition.

Full set of findings and recommendations on the [Recommendations dashboard](https://public.tableau.com/shared/D4W6GY8CS?:display_count=n&:origin=viz_share_link).

### Methodology

- **Sentiment** is derived from explicit star ratings (1–2 = Negative, 3 = Neutral, 4–5 = Positive) rather than NLP scoring. Star ratings are language-agnostic and provided directly by customers — more reliable than NLP on a multilingual dataset.
- **Keyword analysis** uses frequency ranking within sentiment groups, with `share_of_voice` to normalise for slice-size differences. English and Malay keywords were validated by the analyst (native Malaysian speaker).
- **Language detection** via `langdetect`, with a corrected misclassification of Traditional Chinese as Korean (~538k reviews) identified through sampling and remapped to `zh-tw`.
- **Singapore region assignment** combined postal-code regex extraction with OneMap API geocoding, reaching ~60% coverage. Uncovered addresses are labelled "Unknown" rather than imputed.

### v1 limitations (and what motivated v2)

- **Keyword analysis is unigram-only** — negation is not captured (e.g. "tidak sedap" splits into separate tokens). This is the core limitation v2 was built to solve.
- **Singapore region coverage** sits at ~60%; the remaining 40% labelled "Unknown".
- **Chinese keywords** (Taiwan, Hong Kong) are shown for exploratory purposes — native speaker validation recommended before operational use.
- **Six markets excluded from keyword analysis** (Pakistan, Bangladesh, Thailand, Myanmar, Cambodia, Laos) due to language tool limitations on this Python environment.
- **Malay NLP** (Malaya library) was explored but not implemented due to Python 3.14 compatibility constraints.

---

## v2 — LLM Aspect-Based Extraction (279,723 reviews, 9 markets)

### Scope

v1 used the full historical dataset (4.6M reviews, 11 markets). v2 filters to a tighter scope:

- **Time window:** November 2025 – January 2026 (latest 3 months at time of analysis)
- **Markets:** 9 active APAC markets — Pakistan, Malaysia, Bangladesh, Taiwan, Singapore, Myanmar, Cambodia, Laos, and Hong Kong. Thailand was excluded (foodpanda exited in May 2025). Philippines was excluded (no longer scraped in the dataset's latest snapshot).
- **Text filter:** Reviews shorter than 20 characters were dropped (typically emoji-only or single-word responses with no extractable content).

### The 6-aspect taxonomy

The extraction prompt classifies each review against 6 aspect labels, chosen to cover issues that foodpanda's platform operations can directly influence — rider management, logistics, order fulfilment, and food safety escalation.

| Aspect | What it captures | What it excludes |
|---|---|---|
| `rider_service` | Rider attitude, behaviour, communication, address navigation | Generic thanks without specific behaviour |
| `delivery_time` | Explicit speed mentions — late, slow, fast, on time, prompt | Indirect cues like "food was still warm" |
| `delivery_handling` | Food damaged in transit — spills, squashed items | Food taste or quality complaints |
| `order_accuracy` | Missing items, wrong items delivered | Bad quality, small portions, general complaints |
| `packaging` | Container failures, broken packaging, missing utensils | Food taste or quality issues |
| `vendor_quality_flag` | **Severe** food safety issues only — food poisoning, foreign objects, hygiene failures, spoilage | Normal quality complaints (bland, burnt, dry, undercooked) |

Reviews that mention none of these aspects return an empty array. In the flattened output, these appear as rows with null aspect/sentiment/evidence — they represent reviews about food taste, portion size, general satisfaction, or other topics outside the platform's direct control.

### Model selection

Prompt development was done locally using **Llama 3.1 8B via Ollama** — free, fast iteration cycles, no API costs during the experimental phase. Production extraction used **Google Gemini 2.5 Flash Lite** via the `google-genai` async API, chosen for its balance of cost (~$0.10/M input tokens, $0.40/M output tokens), speed, and sufficient output quality for structured aspect extraction.

### Prompt iteration: V1 through V6

The final production prompt was version 6, reached through 6 deliberate iterations. (Prompt versions V1–V6 are internal to the v2 project — not to be confused with the project's v1/v2.) This wasn't a one-shot design — each version addressed specific failure modes observed in the previous one.

**V1** started with a 16-aspect taxonomy covering everything: food quality, food temperature, portion size, app reliability, payment issues, price fairness, and more. The problem was scope — it mixed foodpanda-controllable issues with vendor-side complaints and topics the platform has no leverage over. The model dutifully labeled food taste complaints as `food_quality`, which would have flooded downstream analysis with noise that operations teams can't act on.

**V2** kept the same 16 aspects but added strict output rules: evidence must be a direct quote from the review (5–20 words), each aspect at most once per review, and neutral sentiment only when explicitly stated. This improved output consistency but didn't address the fundamental taxonomy problem.

**V3** cut the taxonomy from 16 to 6 aspects, introducing the "foodpanda-controllable" framing that would define the rest of the project. Aspects were organised into categories (Delivery, Order Fulfilment, Vendor Signal). A `severity` field (low/medium/high) was added to `vendor_quality_flag` to distinguish minor from serious food safety issues.

**V4** dropped the severity field. In practice, it added complexity without proportional analytical value — the downstream analysis needed to know whether a food safety flag existed, not how bad the analyst thought it was. The taxonomy was flattened to a plain 6-label list with definitions.

**V5** was the precision fix. Each of the 6 aspects received a structured definition block: "Use for / DO NOT use for / Example YES / Example NO." This stopped the model from tagging normal quality complaints ("bland", "burnt", "small portion") as `vendor_quality_flag`, and clarified boundary cases. For example, "still warm" was explicitly categorised as handling (food arrived intact), not delivery time — a distinction that had caused inconsistent labels in earlier versions.

**V6** — the final, locked version — added positive/negative example splits for `delivery_time`, distinguishing complaints ("took 2 hours", "very long delivery") from praise ("very prompt delivery", "arrived in 15 mins"). This was the last change before validation.

The exploration notebook where this iteration happened is intentionally not included in this repository — it's working-sketch material with dead ends and rough experiments. The iteration story above documents the methodology so it's visible without needing that file.

### Validation

The V6 prompt was validated on a held-out 30-review stratified sample from Singapore — 6 reviews per star rating (1–5), selected with seed 99, explicitly excluding the earlier 100-review sample used during prompt tuning. I labeled this sample independently and blind, after the final prompt was locked. The labels were my own judgment as the analyst, not a second model's output.

| Metric | Value |
|---|---|
| Review-level exact match | 70% (21/30) |
| Aspect-level F1 | 56% |
| Precision | 43% |
| Recall | 82% |

The lower precision reflects the LLM's tendency to surface aspects I considered too trivial to action — a borderline "rider was nice" tagged as `rider_service` where I would have left it unlabeled. Recall remained high (82%), meaning the model rarely missed aspects I flagged.

For a production use case focused on surfacing issues rather than suppressing false positives, high recall with lower precision is the preferable error profile. Missing a genuine delivery problem is costlier than surfacing an extra "rider was polite" for an analyst to review. The downstream dashboards expose sentiment as a filter, which lets users narrow to negative-sentiment aspects when triaging.

### Key results

| Metric | Value |
|---|---|
| Reviews analyzed | 279,723 |
| Markets covered | 9 |
| Reviews with at least one extracted aspect | ~42% |
| Reviews returning empty (no foodpanda-controllable issue) | ~58% |
| Total aspect mentions extracted | 154,061 |
| Residual API/parse errors | < 0.1% |
| Total extraction time | ~18 hours (across sessions) |
| Total cost | SGD 30.98 (~USD 24) |

### Cost transparency

The total cost for the v2 extraction was **SGD 30.98**, as reported by Google AI Studio billing.

| Component | Detail |
|---|---|
| Model | Gemini 2.5 Flash Lite |
| Pricing | $0.10/M input tokens, $0.40/M output tokens |
| Reviews processed | 279,723 |
| Billed total (Google AI Studio) | SGD 30.98 (~USD 24.39 at 1.27 SGD/USD) |
| Token estimate (production only) | ~SGD 27.26 via `len/4` heuristic |
| Gap | ~SGD 3.72 — development iterations, retries, and connectivity tests |

The token-estimation script uses a `len(text) // 4` character-to-token approximation. This likely undercounts tokens for non-Latin scripts (Urdu, Bengali, Burmese, Chinese characters), so the estimate is directional. The Google AI Studio billed total is the authoritative figure.

### v2 known limitations

**Extraction is not exhaustive.** F1 = 56% (precision = 43%, recall = 82%). Some negative reviews appear with no extracted aspect — because they mention out-of-taxonomy issues (portion size, taste), are too short or generic ("bad food"), or are genuine LLM misses (~18% recall gap).

**Multiple aspects per review.** A single review mentioning 3 issues produces 3 aspect rows in the flattened output. The analysis distinguishes "aspect mentions" (154,061) from "unique reviews" (279,723) — they are different units and labeled as such throughout.

**Aspects can be mentioned positively.** "Rider was so friendly" tags `rider_service` in a 5-star review. Aspect-rate calculations therefore conflate praise and complaints unless sentiment-filtered. The dashboards expose sentiment as a user-adjustable filter to handle this.

**Hong Kong has sparse per-restaurant data.** With ~2,924 reviews total and most restaurants contributing only 1–3 reviews, market-level findings hold but restaurant-level analysis (the vendor watchlist) cannot reliably surface specific Hong Kong restaurants.

**Survivorship bias.** Refunded, cancelled, or removed orders never appear in scraped reviews. The analysis describes a floor of dissatisfaction, not a ceiling.

**Language coverage.** The V6 prompt and validation sample are English-dominant. Markets like Bangladesh (Bengali), Pakistan (Urdu), Cambodia (Khmer), Myanmar (Burmese), and Laos (Lao) contain significant non-English review text. Gemini Flash Lite supports these languages, but extraction accuracy has not been formally validated beyond English, Malay, and Mandarin.

---

## Dashboards

**[v1 dashboard](https://public.tableau.com/views/FoodpandaAPACCustomerFeedbackAnalysis2024-2026/Story1?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)** — market-level experience patterns across 11 APAC markets: rating distributions, food vs rider gaps, keyword analysis by sentiment, and recommendations.

**[v2 dashboard](https://public.tableau.com/views/FoodpandaAPACCustomerFeedbackAnalysisV2/Story1?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)** — a 5-dashboard Tableau Story:

1. **Introduction** — project context and methodology overview
2. **Aspect Distribution by Market** — which markets surface which issues, cross-market comparison
3. **Aspect Distribution by Star Rating** — how aspect mentions correlate with review scores
4. **Restaurant Vendor-Side Audit** — flagged restaurants by `vendor_quality_flag` volume
5. **Review Explorer** — drill into individual reviews with their extracted aspects

---

## Data source

The underlying data comes from the ["Restaurants" and "Reviews" datasets by bwandowando on Kaggle](https://www.kaggle.com/bwandowando) — a public scrape of foodpanda customer reviews across APAC markets, with 2025 and 2026 versioned snapshots. Individual datasets used:

- [Malaysia](https://www.kaggle.com/datasets/bwandowando/malaysian-cities-food-panda-resto-reviews)
- [Singapore](https://www.kaggle.com/datasets/bwandowando/food-panda-resto-reviews)
- [Philippines](https://www.kaggle.com/datasets/bwandowando/food-panda-restaurant-reviews)
- [Taiwan](https://www.kaggle.com/datasets/bwandowando/taiwan-food-panda-restaurant-reviews)
- [Hong Kong](https://www.kaggle.com/datasets/bwandowando/hongkong-food-panda-restaurant-reviews)
- [Bangladesh](https://www.kaggle.com/datasets/bwandowando/bangladesh-cities-food-panda-resto-reviews)
- [Pakistan](https://www.kaggle.com/datasets/bwandowando/pakistan-cities-food-panda-resto-reviews)
- [Thailand](https://www.kaggle.com/datasets/bwandowando/thailand-cities-food-panda-resto-reviews)
- [Cambodia](https://www.kaggle.com/datasets/bwandowando/cambodian-food-panda-resto-reviews)
- [Laos](https://www.kaggle.com/datasets/bwandowando/laos-food-panda-restaurant-reviews)
- [Myanmar](https://www.kaggle.com/datasets/bwandowando/myanmar-food-panda-restaurant-reviews)

---

## Repository contents

| File | Description |
|---|---|
| `v1_foodpanda_APAC_analysis.ipynb` | v1 full pipeline — data loading, deduplication, geocoding, language detection, feature engineering, keyword extraction |
| `v2_production_extraction.ipynb` | v2 production extraction pipeline — loads data, sends reviews to Gemini, saves structured JSON per market, flattens to CSV. Contains all code with execution outputs preserved. |
| `README.md` | This file |

Data files are not included in this repository. Source data is available on Kaggle (linked above). Analytical outputs (`foodpanda_APAC_final.csv`, `foodpanda_keywords.csv`, and the v2 flattened aspect CSV) are generated by the notebooks.

---

## About

Built by **Junior bin Khalit** — data analyst transitioning from operations into analytics.

[LinkedIn](https://linkedin.com/in/junior-k-473700155) · [GitHub](https://github.com/JuniorK96) · [Tableau Public](https://public.tableau.com/app/profile/junior.khalit)
