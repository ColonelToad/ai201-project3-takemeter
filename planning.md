# Project Plan: TakeMeter (Finance Discourse Classifier)

## 1. Community Focus & Rationale
This project targets the broader Reddit financial ecosystem, primarily pulling data from **r/wallstreetbets**, **r/investing**, and **r/algotrading**. 

This domain is uniquely suited for text classification because it uses an overlapping, highly technical vocabulary ("margin calls", "yield curves", "implied volatility") across wildly different tiers of discourse quality. The community conversation ranges from deep, rigorous mathematical theses to emotional, crowd-driven mania and apocalyptic financial doom. A generic keyword search fails here; a successful model must learn to parse the structural integrity and semantic density of an argument rather than relying on financial buzzwords.

---

## 2. Label Taxonomy

The classifier will categorize text into one of three mutually exclusive, exhaustive labels:

*   **`technical_dd`**: The post makes a structured, data-driven investment thesis backed by verifiable metrics (e.g., DCF models, macro indicators, options Greeks, or algorithmic backtests). The argument relies on external financial realities, not internal group dynamics.
*   **`momentum_hype`**: The post focuses on price action, community sentiment, short squeezes, or emotional validation. This also includes **satirical or "fake math" posts** where technical structures are used to justify impossible premises (e.g., meme coin valuations). The underlying logic is circular: an asset is valuable primarily because the crowd is moving toward it.
*   **`existential_doomer`**: Hyper-pessimistic macro-takes, panic vents, or "loss porn" narratives. While these posts may use financial jargon (e.g., "hyperinflation," "liquidity crisis"), their core purpose is cathartic venting, catastrophic signaling, or emotional expression rather than actionable, falsifiable analysis.

### Concrete Label Examples

#### `technical_dd`
*   **Example 1:** "Analyzing the spread between 2-year and 10-year Treasury yields alongside recent core PCE prints suggests the market is mispricing a structural shift. If inflation remains sticky at 3.1%, historical backtests of this spread point to a compression in mid-cap valuation multiples over the next two quarters."
*   **Example 2:** "Company X's Q3 balance sheet reveals an unsustainable debt-to-equity ratio of 4.2. With $500M in floating-rate notes maturing in 12 months, their interest coverage ratio will drop below 1.5 if the Fed holds rates steady, making a dilutive equity offering highly probable."

#### `momentum_hype`
*   **Example 1:** "TSLA to the moon! 🚀 The shorts are completely trapped at the $180 line and the volume is exploding. If we hold the line past Friday's expiration, the market makers will be forced to delta-hedge, triggering the mother of all gamma squeezes. Do not sell!"
*   **Example 2 (The Math Troll):** "If we assume Dogecoin captures 100% of global M2 money supply by 2026, and we divide by the algorithmic hash rate of my basement rig, the fundamental floor price is $420.69. The math literally cannot be wrong, load the boat."

#### `existential_doomer`
*   **Example 1:** "The entire financial system is a house of cards waiting to collapse. The fiat printing press has completely destroyed the purchasing power of the middle class, and this upcoming margin compression is going to wipe out 90% of retail portfolios. Stack cash and prepare for the worst."
*   **Example 2:** "I just lost my entire life savings ($42k) betting on weekly options. My wife is leaving me and I'm currently sitting in a Wendy's parking lot wondering where it all went wrong. The market is completely rigged against the little guy."

---

## 3. Hard Edge Cases & Decision Boundaries

### The Hardest Anticipated Boundary: `technical_dd` vs. `existential_doomer`
*   **The Edge Case Scenario:** A user posts a highly pessimistic, apocalyptic market outlook using dense macroeconomic terms, historical references to the 1929 crash, and citations of Federal Reserve liquidity charts. 
*   **The Problem:** It has the formatting and surface-level trappings of structural analysis (`technical_dd`), but the underlying intent is fatalistic narrative spinning and emotional signaling (`existential_doomer`).
*   **Strict Decision Rule:** If the post provides a **falsifiable thesis with specific, actionable data points** (e.g., "If X economic metric hits Y by date Z, then asset A will drop"), it is labeled `technical_dd`. If the post uses technical jargon merely as a decorative word salad to justify an inevitable, unquantifiable system collapse or an emotional vent, it is labeled `existential_doomer`.

---

## 4. Data Collection Plan

*   **Target Sample Size:** 210 total annotated examples (70 posts per label to ensure perfect class balance).
*   **Text Truncation Rule:** To comply with DistilBERT's 512-token limit, for posts exceeding ~350 words, only the first 3-4 paragraphs (the core thesis) will be captured. 
*   **Sampling Strategy:** Rather than scraping chronological top posts, data will be curated via targeted searches to capture edge cases:
    *   *Sorting by "Controversial"* on `r/wallstreetbets` to uncover posts where the community is heavily divided on whether an argument is brilliant analysis or pure delusion.
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

### Annotation Assistance
The dataset will be annotated entirely by hand to maintain strict gold-standard data integrity. However, an LLM will be used to generate a secondary "shadow" set of labels for the test set to act as a pseudo-inter-annotator reliability check.

### Failure Analysis
Following evaluation, all misclassified examples will be compiled into a structured JSON payload. An LLM will be utilized to perform clustering on these errors, identifying systematic vulnerabilities (e.g., "Model consistently misclassifies posts under 50 words" or "Model fails to parse structural irony"). These insights will be verified manually against the raw text.