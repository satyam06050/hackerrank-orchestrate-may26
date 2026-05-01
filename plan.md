# Ticket Triage Agent — Execution Plan (Revised)

## I/O

* **Input:** `support_tickets.csv`
  Columns: `Issue, Subject, Company`

* **Output:** `support_tickets/output.csv`
  Columns: `Status, Product_Area, Response, Justification, Request_Type`

---

## Step 1: Load Corpus

* Walk directories:

  ```
  data/hackerrank/
  data/claude/
  data/visa/
  ```

* For each `.md` file:

  * `domain` = top folder
  * `product_area` = subfolder
  * Split content using `##` headers

* Store as:

  ```python
  {
    "domain": str,
    "product_area": str,
    "text": str,
    "path": str
  }
  ```

---

## Step 2: Preprocessing

* Combine fields:

  ```python
  text = (Issue + " " + Subject).lower().strip()
  ```

* Normalize:

  * Lowercase
  * Remove extra spaces
  * Optional: remove stopwords

---

## Step 3: Domain Classification

### Priority Order

1. **Company Field**

   ```python
   if Company not empty:
       domain = Company.lower()
   ```

2. **Keyword-Based Fallback**

```python
hackerrank_keywords = ["test", "assessment", "coding", "candidate", "interview"]
claude_keywords     = ["model", "prompt", "api", "response", "anthropic"]
visa_keywords       = ["payment", "card", "transaction", "charge", "refund"]
```

* Count matches per domain
* Select domain with highest score

3. **No Match**

```python
domain = "unknown"
```

---

## Step 4: Request Type Classification

### Keyword Sets

```python
error_keywords   = ["error", "not working", "broken", "fail", "crash"]
feature_keywords = ["add", "feature", "request", "improve", "would be nice"]
```

### Logic

```python
if text.strip() == "":
    request_type = "invalid"

elif any(f in text for f in feature_keywords) and not any(e in text for e in error_keywords):
    request_type = "feature_request"

elif any(e in text for e in error_keywords):
    request_type = "bug"

else:
    request_type = "product_issue"
```

---

## Step 5: Risk Detection

### Risk Levels

#### HIGH (Always Escalate)

```python
["fraud", "unauthorized", "hack", "breach", "stolen"]
```

#### MEDIUM (Conditional)

```python
["payment", "charged", "refund", "invoice", "billing",
 "password", "locked out", "reset", "suspended",
 "gdpr", "data deletion", "legal"]
```

#### LOW

* Everything else

---

## Step 6: Retrieval

### Filtering

* Only consider chunks from detected `domain`

### Scoring

```python
score = keyword_overlap + 0.5 * fuzzy_partial_ratio
```

Where:

* `keyword_overlap` = shared word count
* `fuzzy_partial_ratio` = approximate string similarity

### Selection

* Select **top 3 chunks**
* Minimum threshold:

```python
score >= 0.15
```

* Ensure:

  * Unique `path` (no duplicate sources)

* If no chunks:

```python
confidence = 0
```

* Else:

```python
confidence = top_score
```

---

## Step 7: Decision Logic

```python
if risk == "HIGH":
    Status = "Escalated"

elif confidence < 0.15:
    Status = "Escalated"

elif risk == "MEDIUM" and confidence < 0.25:
    Status = "Escalated"

else:
    Status = "Replied"
```

---

## Step 8: Product Area Assignment

* From **top retrieved chunk**

```python
product_area = top_chunk["product_area"]
```

* If no retrieval:

```python
product_area = "unknown"
```

---

## Step 9: Response Templates

### Reply

```
Summary:
{1–2 sentence paraphrase of top chunk}

Steps:
1. {action 1}
2. {action 2}
```

---

### Escalation

```
Your request has been escalated for manual review.

Reason:
{high-risk keyword detected | no documentation found | low confidence}
```

---

## Step 10: Justification

### Reply

```
Replied. {N} chunks found (area: {product_area}). Risk={risk}. Score={top_score:.2f}. Source={path}
```

### Escalate

```
Escalated. Reason={reason}. Risk={risk}. Score={top_score or 0}
```

---

## Step 11: Edge Cases

| Case              | Action                         |
| ----------------- | ------------------------------ |
| Empty Issue       | invalid + escalate             |
| Company missing   | keyword classification         |
| Unknown domain    | product_area=unknown, escalate |
| Domain tie        | pick highest retrieval score   |
| Issue > 500 chars | truncate before processing     |
| Duplicate chunks  | remove by unique path          |

---

## Step 12: Output

* Format: UTF-8 CSV
* Proper escaping of commas and quotes

### Fields

* `Status` ∈ {Replied, Escalated}
* `Request_Type` ∈ {bug, feature_request, invalid, product_issue}
* `Product_Area` ∈ string
* `Response` ∈ string
* `Justification` ∈ string

---

## Step 13: Validation

* Run on: `sample_support_tickets.csv`

### Target Metrics

* ≥ 80% match on:

  * Status
  * Product_Area

### Debug Logging

```python
{
  "domain": domain,
  "request_type": request_type,
  "risk": risk,
  "confidence": confidence,
  "top_score": top_score,
  "selected_paths": paths
}
```

* Analyze mismatches:

  * Wrong domain
  * Low retrieval score
  * Incorrect risk trigger

---

## Notes

* Prioritize determinism over complexity (important for evaluation)
* Avoid heavy dependencies (no embeddings required)
* Ensure consistent scoring and thresholds
