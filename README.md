# TakeMeter: Evaluating Financial Discourse Quality

## 1. Community Focus & Rationale
This project targets the broader Reddit financial ecosystem, primarily pulling data from **r/wallstreetbets**, **r/investing**, and **r/algotrading**. 

This domain is uniquely suited for text classification because it uses an overlapping, highly technical vocabulary ("margin calls", "yield curves", "implied volatility") across wildly varying conversations. The community conversation ranges from deep, rigorous mathematical theses to emotional, crowd-driven mania and apocalyptic financial doom. A generic keyword search fails here; a successful model must learn to parse the structural integrity and semantic density of an argument rather than relying on financial buzzwords.

---

## 2. Label Taxonomy

The classifier categorizes text into one of three labels:

* **`technical_dd`**: The post makes a structured, data-driven argument backed by verifiable metrics (e.g., DCF models, macro indicators, options Greeks, or algorithmic backtests). The argument relies on external financial realities, not internal group dynamics.
    * *Example 1:* "Analyzing the spread between 2-year and 10-year Treasury yields alongside recent core PCE prints suggests the market is mispricing a structural shift. If inflation remains sticky at 3.1%, historical backtests point to a compression in mid-cap multiples."
    * *Example 2:* "Company X's Q3 balance sheet reveals an unsustainable debt-to-equity ratio of 4.2. With $500M in floating-rate notes maturing in 12 months, their interest coverage ratio will drop below 1.5 if the Fed holds rates steady."
* **`momentum_hype`**: The post focuses on price action, community sentiment, short squeezes, or emotional validation. This also includes **satirical or "fake math" posts** where technical structures are used to justify impossible premises. 
    * *Example 1:* "TSLA to the moon! 🚀 The shorts are completely trapped at the $180 line and the volume is exploding. Do not sell!"
    * *Example 2:* "If we assume Dogecoin captures 100% of global M2 money supply by 2026, and we divide by the algorithmic hash rate of my basement rig, the fundamental floor price is $420.69. The math literally cannot be wrong."
* **`existential_doomer`**: Hyper-pessimistic macro-takes, panic vents, or "loss porn" narratives. While these posts may use financial jargon, their core purpose is cathartic venting, catastrophic signaling, or emotional expression.
    * *Example 1:* "The entire financial system is a house of cards waiting to collapse. The fiat printing press has completely destroyed the purchasing power of the middle class. Stack cash."
    * *Example 2:* "I just lost my entire life savings ($42k) betting on weekly options. My wife is leaving me and I'm currently sitting in a Wendy's parking lot wondering where it all went wrong."

---

## 3. Data Collection & Annotation

**Data Sources:** Data was manually curated via targeted searches on Reddit. To capture edge cases, I sorted `r/wallstreetbets` by "Best" to find widely accepted hype, and cross-pollinated search terms (e.g., searching "yield curve" in both `r/investing` and `r/wallstreetbets`) to find the same vocabulary used for different purposes. 

**Label Distribution:** The dataset contains roughly 250 examples, however there was initial label imbalance, as finding enough examples for technical_dd was hard. Most discussion also was hard to fit as just doomer or momentum_hype, while the tone and context is different giving the model just 1 sentence or so it has to make the judgment without the context.

**Difficult-to-Label Examples:**
1.  **The "Historically Accurate Doomer":** A post cited accurate 1929 market crash statistics to argue that the market was going to zero tomorrow. *Decision:* Labeled `existential_doomer`. Despite real numbers, the thesis was unfalsifiable and catastrophic.
2.  **The "Math Meme":** A 500-word post using actual Black-Scholes equations to price options on a literal joke cryptocurrency. *Decision:* Labeled `momentum_hype`. The math was real, but the premise was satirical.
3.  **The "Earnings Panic":** A short post: "Revenue missed by 12%, margins are compressing, I'm selling everything and giving up." *Decision:* Labeled `existential_doomer`. It started as `technical_dd` but devolved into a pure emotional reaction.

---

## 4. Fine-Tuning Approach

* **Base Model:** `distilbert-base-uncased`
* **Training Setup:** Fine-tuned via HuggingFace's `Trainer` API on Google Colab (T4 GPU). 
* **Hyperparameter Decision:** I utilized a learning rate of `2e-5` with a batch size of 16 over 3 epochs. Because the dataset was small and the boundaries were highly subjective, keeping the learning rate low was necessary to prevent the model from immediately overfitting to specific keywords (like "rocket").

---

## 5. Baseline Comparison

To establish a zero-shot baseline, I prompted Groq's `llama-3.3-70b-versatile`. 
* **Prompt Setup:** The prompt provided the model with the exact definitions and examples from the taxonomy, along with strict formatting instructions to output *only* the lowercase label string.
* **Results Collection:** The baseline classified the exact same 37-example test set used to evaluate the fine-tuned DistilBERT model.

---

## 6. Evaluation Report

### Performance Metrics

| Metric | Groq Llama 3.3 (Baseline) | DistilBERT (Fine-Tuned) |
| :--- | :--- | :--- |
| **Overall Accuracy** | **97.30%** | **78.38%** |

*Note: The zero-shot 70B parameter model massively outperformed the fine-tuned 66M parameter model, beating it by nearly 19%.*

**Fine-Tuned Per-Class Metrics:**
* **`technical_dd`**: Precision: 0.00 | Recall: 0.00 | F1: 0.00
* **`momentum_hype`**: Precision: 1.00 | Recall: 0.90 | F1: 0.95
* **`existential_doomer`**: Precision: 0.70 | Recall: 1.00 | F1: 0.82

### Confusion Matrix (Fine-Tuned Model)

| True \ Predicted | `technical_dd` | `momentum_hype` | `existential_doomer` |
| :--- | :--- | :--- | :--- |
| **`technical_dd`** | **0** | 0 | 7 |
| **`momentum_hype`** | 0 | **10** | 1 |
| **`existential_doomer`** | 0 | 0 | **19** |

### Error Analysis (3 Specific Failures)

The model suffered a catastrophic collapse on the `technical_dd` class, predicting 100% of true `technical_dd` examples as `existential_doomer`. 

1.  *Text:* "Analyzing the Fibonacci retracement levels on the weekly Nasdaq chart. We are perfectly bouncing off the 0.618 golden ratio. Bullish continuation."
    * *Predicted:* `existential_doomer` (Confidence: 0.40)
    * *Analysis:* The confidence score is incredibly low (random guessing is 0.33). Because the model learned that `momentum_hype` contains exclamation points and emojis, it looks at this dry text and assumes it must be doomerism. It completely failed to learn that charting terminology indicates analysis.
2.  *Text:* "Calculated the Gamma exposure (GEX) of the SPX. It's deeply negative, meaning market makers will sell into weakness, accelerating any selloff."
    * *Predicted:* `existential_doomer` (Confidence: 0.41)
    * *Analysis:* This is a classic edge case failure. The post uses highly technical math ("Gamma exposure"), but the conclusion is negative ("accelerating any selloff"). The model latched onto the negative sentiment and ignored the mathematical structure, routing it to Doomer. 
3.  *Text:* "CRWD is unstoppable! Cybersecurity is a blank check industry right now. Earnings will be a massive beat! 🚀🔒"
    * *Predicted:* `existential_doomer` (Confidence: 0.38)
    * *Analysis:* This was the one `momentum_hype` failure. It is fascinating because it includes rocket emojis (the classic hype signal), yet the model predicted Doomer. The incredibly low confidence (0.38) suggests the model encountered an unexpected token structure and defaulted to its most frequently guessed "other" class.

To fix this, the model needs a significantly larger training set of `technical_dd` that expresses *positive* or *neutral* sentiment. Right now, it clearly associates "dry/complex" with "doomer".

### Sample Classifications

| Post Text | True Label | Predicted Label | Confidence |
| :--- | :--- | :--- | :--- |
| "I just lost my entire portfolio on SPY puts. $40k gone. I'm so tired of this rigged casino." | `existential_doomer` | `existential_doomer` | 0.94 |
| "Apes hold the line! GME to the moon! 🚀🚀 We squeeze them on Friday!" | `momentum_hype` | `momentum_hype` | 0.98 |
| "Scraping SEC Edgar for insider buying data. Found a strong correlation..." | `technical_dd` | `existential_doomer` | 0.36 |
| "Using a GARCH model to forecast volatility for the upcoming earnings..." | `technical_dd` | `existential_doomer` | 0.41 |

* **Why the correct prediction is reasonable:** The model predicted the first post as `existential_doomer` with 94% confidence. This is highly logical because the post contains hallmark features of the class: specific monetary loss ("$40k gone"), fatalistic emotional language ("so tired"), and conspiratorial coping mechanisms ("rigged casino"). The model accurately identified these as signals of emotional venting rather than analysis.

---

## 7. Reflection: What the Model Actually Learned

I intended for the model to learn the difference between *evidence-based reasoning* and *emotional reacting*. 

Instead, the model learned a much simpler heuristic: **"If it has emojis and capital letters, it is Hype. If it sounds dry, technical, or slightly negative, it is Doomer."** The model completely failed to carve out a decision boundary for `technical_dd`. Because financial jargon is heavily shared between serious analysis and Doomer loss-porn, DistilBERT (being a smaller, 66M parameter model) struggled to contextualize the syntax. It overfit to the emotional valance of the text. By contrast, the 70B parameter Llama 3 baseline possessed the pre-trained logical capability to recognize when an argument was structurally sound vs. fatalistic, achieving near-perfect accuracy.

---

## 8. Spec Reflection

* **How the Spec Helped:** Creating strict decision boundaries in the `planning.md` (specifically the rule regarding "falsifiable thesis vs. decorative word salad") made the manual annotation process incredibly fast. I rarely had to second-guess myself while labeling the 210 examples.
* **Where Implementation Diverged:** I initially planned to manually truncate posts to 3-4 paragraphs to fit DistilBERT's 512-token limit. In practice, I realized that truncating a highly technical DD post sometimes removes the actual "falsifiable thesis" at the end, leaving only the introductory context. This likely contributed to the model confusing `technical_dd` with doomerism, as the "proof" was chopped off.

---

## 9. AI Usage Acknowledgment

1.  **Label Stress-Testing:** Before collecting data, I fed my taxonomy into Claude 3.5 Sonnet and asked it to generate 10 highly ambiguous financial posts designed to break my labels. This exercise revealed that I hadn't accounted for "Satirical Math" posts, prompting me to update the `momentum_hype` definition to include fake math before I started annotating.
2.  **Baseline Prompt Formatting:** I utilized an LLM to help me write the exact formatting instructions for the Llama 3.3 zero-shot prompt, ensuring the output would be cleanly parseable by the Colab notebook's evaluation script (stripping out punctuation, markdown, and conversational filler).