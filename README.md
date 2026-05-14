# Foodpanda APAC Customer Feedback Analysis v2 — Aspect-Based Sentiment Extraction

LLM-powered extraction of foodpanda-controllable service issues from 279,723 customer reviews across 9 APAC markets.

**[→ View the interactive dashboard on Tableau Public](https://public.tableau.com/views/FoodpandaAPACCustomerFeedbackAnalysisV2/Story1?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)**


## Project Overview

This is version 2 of a customer feedback analysis built on publicly scraped foodpanda reviews. **v1** used unigram keyword frequency to surface what customers mention most — useful for a first pass, but unable to distinguish "late delivery" from "not late at all." **v2** adds an AI extraction layer: each review is read by Google's Gemini 2.5 Flash Lite and labeled with structured aspects, sentiment, and quoted evidence.

The framing is deliberately narrow. Rather than scoring overall satisfaction (which the star rating already captures), the extraction targets **foodpanda-controllable issues** — things the platform's operations team could act on. Vendor-side complaints like bland food or small portions are excluded by design; they're the restaurant's responsibility, not the platform's.

The architecture is hybrid: the LLM handles qualitative interpretation (reading a review and deciding which aspects are mentioned); deterministic Python handles all aggregation, filtering, and output.

**v1 links:** [GitHub](https://github.com/JuniorK96/foodpanda-apac-customer-feedback-analysis) | [Tableau Public](https://public.tableau.com/app/profile/junior.khalit/viz/FoodpandaAPACCustomerFeedbackAnalysis2024-2026/Story1)

## Data Source

The underlying data comes from the ["Restaurants" and "Reviews" datasets by bwandowando on Kaggle](https://www.kaggle.com/bwandowando) — a public scrape of foodpanda customer reviews across APAC markets.

v1 used the full historical dataset (4.6M reviews, 11 markets). v2 filters to a tighter scope:

- **Time window:** November 2025 – January 2026 (latest 3 months at time of analysis)
- **Markets:** 9 active APAC markets. Thailand was excluded (foodpanda exited in May 2025). Philippines was excluded (no longer scraped in the dataset's latest snapshot).
- **Text filter:** Reviews shorter than 20 characters were dropped (typically emoji-only or single-word responses with no extractable content).
- **Final scope:** 279,723 reviews across Pakistan, Malaysia, Bangladesh, Taiwan, Singapore, Myanmar, Cambodia, Laos, and Hong Kong.

## Methodology

### The 6-Aspect Taxonomy

The extraction prompt classifies each review against 6 aspect labels. These were chosen to cover issues that foodpanda's platform operations can directly influence — rider management, logistics, order fulfilment, and food safety escalation.

| Aspect | What it captures | What it excludes |
|---|---|---|
| `rider_service` | Rider attitude, behaviour, communication, address navigation | Generic thanks without specific behaviour |
| `delivery_time` | Explicit speed mentions — late, slow, fast, on time, prompt | Indirect cues like "food was still warm" |
| `delivery_handling` | Food damaged in transit — spills, squashed items | Food taste or quality complaints |
| `order_accuracy` | Missing items, wrong items delivered | Bad quality, small portions, general complaints |
| `packaging` | Container failures, broken packaging, missing utensils | Food taste or quality issues |
| `vendor_quality_flag` | **Severe** food safety issues only — food poisoning, foreign objects, hygiene failures, spoilage | Normal quality complaints (bland, burnt, dry, undercooked) |

Reviews that mention none of these aspects return an empty array. In the flattened output, these appear as rows with null aspect/sentiment/evidence — they represent reviews about food taste, portion size, general satisfaction, or other topics outside the platform's direct control.

### Model Selection

Prompt development (V1–V5) was done locally using **Llama 3.1 8B via Ollama** — free, fast iteration cycles, no API costs during the experimental phase. Production extraction used **Google Gemini 2.5 Flash Lite** via the `google-genai` async API, chosen for its balance of cost (~$0.10/M input tokens, $0.40/M output tokens), speed, and sufficient output quality for structured aspect extraction.

### Prompt Iteration: V1 through V6

The final production prompt was version 6, reached through 6 deliberate iterations. This wasn't a one-shot design — each version addressed specific failure modes observed in the previous one.

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

## Key Results

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

## Dashboards

The analysis feeds a 5-dashboard Tableau Story:

1. **Introduction** — project context and methodology overview
2. **Aspect Distribution by Market** — which markets surface which issues, cross-market comparison
3. **Aspect Distribution by Star Rating** — how aspect mentions correlate with review scores
4. **Restaurant Vendor-Side Audit** — flagged restaurants by `vendor_quality_flag` volume
5. **Review Explorer** — drill into individual reviews with their extracted aspects

[Tableau Public — https://public.tableau.com/views/FoodpandaAPACCustomerFeedbackAnalysisV2/Story1?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link]

## Cost Transparency

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

## Known Limitations

**Extraction is not exhaustive.** F1 = 56% (precision = 43%, recall = 82%). Some negative reviews appear with no extracted aspect — because they mention out-of-taxonomy issues (portion size, taste), are too short or generic ("bad food"), or are genuine LLM misses (~18% recall gap).

**Multiple aspects per review.** A single review mentioning 3 issues produces 3 aspect rows in the flattened output. The analysis distinguishes "aspect mentions" (154,061) from "unique reviews" (279,723) — they are different units and labeled as such throughout.

**Aspects can be mentioned positively.** "Rider was so friendly" tags `rider_service` in a 5-star review. Aspect-rate calculations therefore conflate praise and complaints unless sentiment-filtered. The dashboards expose sentiment as a user-adjustable filter to handle this.

**Hong Kong has sparse per-restaurant data.** With ~2,924 reviews total and most restaurants contributing only 1–3 reviews, market-level findings hold but restaurant-level analysis (the vendor watchlist) cannot reliably surface specific Hong Kong restaurants.

**Survivorship bias.** Refunded, cancelled, or removed orders never appear in scraped reviews. The analysis describes a floor of dissatisfaction, not a ceiling.

**Language coverage.** The V6 prompt and validation sample are English-dominant. Markets like Bangladesh (Bengali), Pakistan (Urdu), Cambodia (Khmer), Myanmar (Burmese), and Laos (Lao) contain significant non-English review text. Gemini Flash Lite supports these languages, but extraction accuracy has not been formally validated beyond English, Malay, and Mandarin.

**Cost figure context.** SGD 30.98 is the actual Google AI Studio billed total. The separate token-estimation script approximated production-only cost at SGD 27.26; the gap is development, iteration, and retry overhead. The token estimate uses a `len/4` heuristic that likely undercounts non-Latin scripts, so it is directional only.

## Repository Contents

| File | Description |
|---|---|
| `v2_production_extraction.ipynb` | Production extraction pipeline — loads data, sends reviews to Gemini, saves structured JSON per market, flattens to CSV. Contains all code with execution outputs preserved. |
| `README.md` | This file. |


- **Author:**
Built by **Junior bin Khalit** 
[LinkedIn](https://linkedin.com/in/junior-k-473700155) · [GitHub](https://github.com/JuniorK96) · [Tableau Public](https://public.tableau.com/app/profile/junior.khalit)
