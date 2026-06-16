# Project Plan: TakeMeter (Finance Discourse Classifier)

## 1. Community Focus & Rationale
This project targets the broader Reddit financial ecosystem, primarily pulling data from **r/wallstreetbets**, **r/investing**, and **r/algotrading**. 

This domain is uniquely suited for text classification because it uses an overlapping, highly technical vocabulary ("margin calls", "yield curves", "implied volatility") across wildly different tiers of discourse quality. The community conversation ranges from deep, rigorous mathematical theses to emotional, crowd-driven mania and apocalyptic financial doom. A generic keyword search fails here; a successful model must learn to parse the structural integrity and semantic density of an argument rather than relying on financial buzzwords.

---

## 2. Label Taxonomy

The classifier will categorize text into one of three labels:

*   **`technical_dd`**: The post makes a structured, data-driven argument backed by verifiable metrics (e.g., DCF models, macro indicators, options Greeks, or algorithmic backtests). The argument relies on external financial realities, not internal group dynamics.
*   **`momentum_hype`**: The post focuses on price action, community sentiment, short squeezes, or emotional validation. This also includes **satirical or "fake math" posts** where technical structures are used to justify impossible premises (e.g., meme coin valuations). The underlying logic is circular: an asset is valuable primarily because the crowd is moving toward it.
*   **`existential_doomer`**: Hyper-pessimistic macro-takes, panic vents, or "loss porn" narratives. While these posts may use financial jargon (e.g., "hyperinflation," "liquidity crisis"), their core purpose is cathartic venting, catastrophic signaling, or emotional expression rather than actionable, falsifiable analysis.

### Concrete Label Examples

#### `technical_dd`
*   **Example 1:** "On june 12, the largest ipo in the history of financial markets goes live. $75 billion raise thats like more than triple what saudi aramco pulled in 2019 which was the previous record. Bitpanda is also listing spcx from day one with fractional shares. I spent the last few days actually going through the S1 filing. The spacex handles 82% of all US space launches and 45% of every commercial space contract on the planet while starlink hit 10m subs across 164 countries by end of q1 2026, roughly double what it was a year ago and connectivity revenue came in at $3.26 billion in just q1 alone..."

*   **Example 2:** "friday's gold move is the bond market talking. gold opened 4652 and closed 4538 on the broker daily candle, low of 4511 intraday. 114 dollar single-day drop. silver collapsed from a weekly high near 88 down to a 75.89 close. this looks insane when CPI is at 3.8 percent (highest since may 2023) and PPI is at 6 percent with wholesale gasoline up 15.6 percent in a single month. gold is supposed to be the inflation hedge."

#### `momentum_hype`
*   **Example 1:** "Hype the f\*ck out and buy everything right now. This is the f\*cking moonshot. (satricial quote attributed to Warren Buffett)"

*   **Example 2:** 
"TLDR:
#1 SPCX, like the rocket, goes up and will clear orbit before crashing in October when lockups expire.
#2 Valuation of the stock doesn't matter - it's the float that matters. Almost no one who can sell got in below IPO price
#3 The forced buyers are the boomers and their 401k funds and those tracking QQQ. Hedgies know this, and now we do too. The buy is up to 45% of the float.
#4 Apes strong together - most retial brokers are giving penalties for selling before 15/30/60/90 days, so all we have to do is not be paper handed regards.
#4 Options released next week = more gamma squeeze.
#5 Macro is short term bullish if we continue to work towards and away from a peace deal
#6 Also, everyone hates Elon on Reddit so an inverse is obvious. I hate Elon too, but I like money."

#### `existential_doomer`
*   **Example 1:** "im so tired. What made me think i could do this. 90% of people fail, at trading... yet, I carry on like i'm going to magically make this work. i have nothing. i'm so tired. i'm so exhausted."

*   **Example 2:** "The volatility of the last few months due to the war, the constant “peace talks are looking promising!” then whiplashing to “just kidding everything is getting blown up”, constant need to restructure strategies to factor all of this in(obviously this is always a thing, but not to this extent). It’s all just exhausting, frustrating, and infuriating that it is being allowed to continue. I guess financial laws only apply to us normies."

---

## 3. Hard Edge Cases & Decision Boundaries

### The Hardest Anticipated Boundary: `technical_dd` vs. `existential_doomer`
*   **The Edge Case Scenario:** A user posts a highly pessimistic, apocalyptic market outlook using dense macroeconomic terms, historical references to the 1929 crash, and citations of Federal Reserve liquidity charts. 
*   **The Problem:** It has the formatting and surface-level trappings of structural analysis (`technical_dd`), but the underlying intent is fatalistic narrative spinning and emotional signaling (`existential_doomer`).
*   **Strict Decision Rule:** If the post provides a **falsifiable thesis with specific, actionable data points** (e.g., "If X economic metric hits Y by date Z, then asset A will drop"), it is labeled `technical_dd`. If the post uses technical jargon merely as a decorative word salad to justify an inevitable, unquantifiable system collapse or an emotional vent, it is labeled `existential_doomer`.

---

## 4. Data Collection Plan

*   **Target Sample Size:** 210 total annotated examples (70 posts per label to ensure perfect class balance).
*   **Text Truncation Rule:** To comply with DistilBERT's 512-token limit, for posts exceeding ~350 words, only the first 3-4 paragraphs (usually the main point) will be captured. 
*   **Sampling Strategy:** Rather than scraping chronological top posts, data will be curated via targeted searches to capture edge cases:
    *   *Sorting by "best" or "hot"* on `r/wallstreetbets` to uncover posts where the community is heavily engaged and possibly divided on whether an argument is brilliant analysis or pure delusion.
    *   *Keyword cross-pollination:* Searching identical terms (e.g., "yield curve", "short interest") across `r/investing` (seeking `technical_dd`) and `r/wallstreetbets` (seeking `momentum_hype` or `existential_doomer`).
*   **Underrepresentation Mitigation:** If a specific category (e.g., `technical_dd`) is underrepresented after initial collection, targeted scraping will shift exclusively to `r/algotrading` and `r/SecurityAnalysis` to build up the required count.

---

## 5. Evaluation Metrics

While overall **Accuracy** will serve as the headline metric, it is insufficient for a highly nuanced, subjective text classification task. This project will heavily rely on:
1.  **Per-Class F1-Score:** Because the boundaries between hype, doom, and true analysis are thin, we need to balance Precision (avoiding false alarms) and Recall (not missing true signals) for each label.
2.  **Confusion Matrix Analysis:** This will be the vital diagnostic tool. It will show exactly where the model degrades—specifically testing whether the model struggles to differentiate technical word-salad doomerism from true financial deep dives.

---

## 6. Definition of Success

For this classifier to be considered "good enough" for integration into a real-world community moderation tool (e.g., automatically adding flairs like "High-Quality Analysis" vs. "Speculative Hype"):
*   **Minimum Acceptable Baseline:** Overall Accuracy $\ge$ 75% on the unseen test set, with the fine-tuned DistilBERT model outperforming the zero-shot Llama 3.3 70B baseline by at least 5% in macro F1-score.
*   **Deployable Threshold:** A per-class F1-score of $\ge$ 80% for `technical_dd`. It is acceptable if the model occasionally confuses hype and doomerism, but it must reliably isolate genuine, data-backed analysis from noise.

---

## 7. AI Tool Plan

### Label Stress-Testing
Prior to annotating the 200 examples, the label definitions and decision boundaries will be fed into a frontier LLM. The model will be prompted to generate 10 highly ambiguous financial posts designed to intentionally break the taxonomy boundaries. If these synthetic posts reveal unhandled overlaps, the decision rules will be revised before human annotation begins.

### Failure Analysis
Following evaluation, all misclassified examples will be compiled into a structured JSON payload. An LLM will be utilized to perform clustering on these errors, identifying systematic vulnerabilities (e.g., "Model consistently misclassifies posts under 50 words" or "Model fails to parse structural irony"). These insights will be verified manually against the raw text.