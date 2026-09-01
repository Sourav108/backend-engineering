# Relevance Ranking and the Okapi BM25 Scoring Algorithm

---

## 1. What Is It?
In search engines, **Relevance Ranking** is the mathematical scoring process that calculates a numerical relevance score (`_score`) for each matching document in response to a user query, ordering the results from most relevant to least relevant.

**Okapi BM25 (Best Matching 25)** is the industry-standard probabilistic ranking algorithm utilized by Apache Lucene, Elasticsearch, and OpenSearch.

---

## 2. Why Does It Exist?
Legacy search engines used simple **TF-IDF (Term Frequency - Inverse Document Frequency)**:
- **The Keyword Stuffing Flaw**: In TF-IDF, Term Frequency scales linearly. A spam document repeating the word `"cheap hotel"` 500 times receives a score $500\times$ higher than a legitimate article mentioning it 3 times.
- **Document Length Bias**: A 100-page encyclopedia mentioning a word 5 times purely by coincidence would rank higher than a concise 1-paragraph article exclusively about that subject.

BM25 resolves both flaws using **Term Frequency Saturation** and **Document Length Normalization**.

---

## 3. Mental Model: The BM25 Term Frequency Saturation Curve

```text
Score Boost
  ^
  |                                     -------------------- (BM25 Asymptotic Ceiling)
  |                                 ---
  |                             ---
  |                         ---
  |                      --
  |                    -
  |                  /
  |                /
  |              /  (Legacy TF-IDF: Unbounded linear growth / Spammer paradise)
  |            /
  |          /
  |        /
  |      /
  |    /
  +------------------------------------------------------------> Term Count (TF)
      1   2   3   4   5   10          50                 500
```

---

## 4. How Does It Work?

### The BM25 Mathematical Equation

$$\text{Score}(D, Q) = \sum_{i=1}^N \text{IDF}(q_i) \cdot \frac{f(q_i, D) \cdot (k_1 + 1)}{f(q_i, D) + k_1 \cdot \left(1 - b + b \cdot \frac{|D|}{\text{avgdl}}\right)}$$

---

### The 3 Core Components of BM25:

#### 1. Inverse Document Frequency ($\text{IDF}$)
$$\text{IDF}(q_i) = \ln \left( 1 + \frac{N - n(q_i) + 0.5}{n(q_i) + 0.5} \right)$$
- Measures **how rare or informative a term is across the entire corpus**:
  - A common word like `"the"` appears in $1,000,000$ documents $\longrightarrow \text{IDF} \approx 0$ (**Zero Score Boost**).
  - A rare technical term like `"spiffe"` appears in only $12$ documents $\longrightarrow \text{IDF}$ is **Huge** (**High Score Boost**).

---

#### 2. Term Frequency Saturation ($k_1 \approx 1.2$)
- $f(q_i, D)$ is the frequency of the term in document $D$.
- $k_1$ controls the saturation limit (default: $1.2$).
- The first 2–3 occurrences of a term provide a sharp score increase; further repetitions quickly hit an asymptotic ceiling, **neutralizing keyword stuffing**.

---

#### 3. Document Length Normalization ($b \approx 0.75$)
- $|D|$ is the length of the current document; $\text{avgdl}$ is the average document length across the entire index.
- $b$ controls how aggressively long documents are penalized (default: $0.75$).
- If a short 10-word product title matches the query `"wireless headphones"`, it receives a significantly higher score than a 5,000-word user manual mentioning the phrase once.

---

## 5. Field Boosting & Function Score Queries

In production search, business signals (e.g. matching in the title vs body, product popularity, publish date recency) override pure text scoring.

```json
POST /production_articles/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query": "kubernetes networking",
          "fields": [
            "title^3",      // 3x Score Boost for title matches!
            "summary^1.5",  // 1.5x Score Boost for summary matches
            "content"       // 1.0x Base Score
          ]
        }
      },
      "functions": [
        {
          // Boost recent articles using Exponential Date Decay
          "exp": {
            "published_at": {
              "origin": "now",
              "scale": "30d",
              "decay": 0.5
            }
          }
        },
        {
          // Boost high-popularity articles using log(view_count)
          "field_value_factor": {
            "field": "view_count",
            "modifier": "log1p",
            "factor": 0.1
          }
        }
      ],
      "score_mode": "sum",
      "boost_mode": "multiply"
    }
  }
}
```

---

## 6. Performance

| Search Query Scenario | Pure Boolean Match | Legacy TF-IDF | Modern Okapi BM25 |
|---|---|---|---|
| Spam Document (Word repeated 500x) | Ranked #1 | Ranked #1 (Distorted) | **Ranked Low (Saturated)** |
| Concise Title vs 100-page Doc | Equal Rank | 100-page doc ranks higher | **Concise Title Ranks #1** |
| Rare Term + Common Term | Equal weight | Moderate | **Rare Term Dominates Score** |

---

## 7. Interview Questions

### Q1: What are the two main tuning parameters in the BM25 scoring algorithm, and how do they affect search ranking?
<details>
<summary>Reveal Answer</summary>

**Answer**:
The two primary parameters in BM25 are **$k_1$** and **$b$**:
1. **$k_1$ (Term Frequency Saturation Parameter - default: 1.2)**:
   - Dictates how quickly the score saturates as a term appears multiple times in a document.
   - Setting $k_1 = 0$ ignores term frequency completely (treating occurrences as binary yes/no).
   - Higher values of $k_1$ allow additional occurrences to increase the score for longer before leveling off.
2. **$b$ (Document Length Normalization Parameter - default: 0.75)**:
   - Dictates how strongly document length relative to average length penalizes the score.
   - Setting $b = 0$ completely disables length normalization (a 10-word title and a 10,000-word book with the same term count receive identical scores).
   - Setting $b = 1.0$ fully scales the penalty proportional to document length.
</details>

---

## 8. Quick Revision
- **BM25**: Modern Lucene relevance scoring algorithm replacing TF-IDF.
- **IDF**: Rare words score higher than common words.
- **TF Saturation ($k_1$)**: Prevents keyword spamming from distorting relevance.
- **Length Normalization ($b$)**: Penalizes long bloated documents relative to average length.
- **Field Boosting (`title^3`)**: Multiplies score weights for high-priority fields.
- **Function Score**: Blends business signals (popularity, date decay) with textual relevance.
