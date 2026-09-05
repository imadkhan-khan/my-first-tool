# Fashion trend research tool — full build flow

A practical build plan for a web app that takes a product idea (spotted on a celebrity/influencer), searches it against Amazon via Keepa, and uses an LLM to extract materials/styles, map competitors, and suggest directions.

---

## Stage 0 — Manual trend spotting (outside the app)

You browse influencer/celebrity content yourself. No images are stored, uploaded, or processed by the app — this stays entirely manual. You form a text description of the product ("oversized cropped denim jacket with contrast stitching") and that description is your input to Stage 1.

---

## Stage 1 — Input capture (your web app)

A simple form:
- Product description / name (free text)
- Optional: category (jackets, dresses, sneakers, etc.), price range, target marketplace (amazon.com, .co.uk, etc.)

This gets stored as a "research query" record and passed to Stage 2.

```
POST /api/research
{
  "query": "oversized cropped denim jacket contrast stitching",
  "category": "womens-jackets",
  "domain": "amazon.com"
}
```

---

## Stage 2 — Keepa product search

Use Keepa's **Product Finder** endpoint with title-based keyword search, optionally scoped by category.

```python
import keepa

api = keepa.Keepa("<YOUR_KEEPA_KEY>")

params = keepa.ProductParams(
    title="oversized cropped denim jacket",
    perPage=50,
    sort=[["current_SALES", "asc"]]   # lower sales rank = better seller
)

asins = api.product_finder(params)
products = api.query(asins, history=False, stats=90, typed=True)
```

For each result you'll get: ASIN, title, brand, current price, sales rank, review count/rating, offer count, category — plus bullet-point description text if available.

**Store the raw results** (title, brand, price, rank, rating, ASIN) in your DB, linked to the research query. This is your competitor/market dataset for this query.

---

## Stage 3 — Relevance filtering (LLM pass 1)

Raw title search is noisy — you'll get irrelevant matches. Before doing any style analysis, have Claude filter the result set down to genuinely comparable products.

**Prompt structure:**
```
You are filtering Amazon search results for relevance to a fashion research query.

Query: "oversized cropped denim jacket contrast stitching"

Here are 50 product titles with brand and price:
[JSON list of {asin, title, brand, price}]

Return ONLY the ASINs that are genuinely comparable products (same garment
type and general style intent), excluding accessories, unrelated items, or
completely different product categories. Return as JSON: {"relevant_asins": [...]}
```

This keeps your downstream analysis clean and is a cheap, fast LLM call (title-only, no need for full product data yet).

---

## Stage 4 — Attribute extraction (LLM pass 2)

For the filtered, relevant ASINs, pull full titles + bullet/description text from your stored Keepa data, and ask Claude to extract structured attributes.

**Prompt structure:**
```
Extract structured fashion attributes from each product listing below.

For each product, return:
- material (e.g. "cotton-linen blend", "recycled denim", "unknown" if not stated)
- fit/silhouette (e.g. "oversized", "cropped", "regular")
- notable style details (e.g. "contrast stitching", "raw hem", "distressed")
- price_tier (budget / mid / premium, based on price relative to the set)

Listings:
[JSON list of {asin, title, brand, price, description}]

Return as JSON array.
```

Store this structured output per ASIN — this becomes your queryable "attribute database" over time, not just a one-off report.

---

## Stage 5 — Aggregation & pattern summary

This is pure data work (no LLM needed), done in your backend:
- Count frequency of each material across the result set
- Count frequency of each style detail
- Group by price tier
- Identify brand concentration (who's making the most similar products)

```python
from collections import Counter

materials = Counter(item["material"] for item in extracted)
style_details = Counter(d for item in extracted for d in item["notable_style_details"])
```

This gives you hard numbers: "6 of 40 comparable listings use recycled denim," "raw hem appears in 12 of 40," etc. — the quantitative backbone for the next step.

---

## Stage 6 — Synthesis & suggestions (LLM pass 3)

Feed the aggregated stats (not raw listings — keep the context small and structured) to Claude and ask for a synthesis.

**Prompt structure:**
```
You're analyzing the competitive landscape for a fashion product idea.

Original idea: "oversized cropped denim jacket with contrast stitching"

Market data from {N} comparable Amazon listings:
- Material frequency: {material_counts}
- Style detail frequency: {style_counts}
- Price tier distribution: {price_tier_counts}
- Top brands in this space: {brand_counts}

Based on this data:
1. What's saturated (many competitors already doing this)?
2. What's underrepresented (a potential gap)?
3. What 2-3 concrete material/style directions would differentiate a new
   product while still fitting proven demand signals?

Be specific and reference the data. Flag this as a data-informed suggestion,
not a guaranteed trend — the underlying sample is a snapshot of current
Amazon listings, not a forecast.
```

This is your "prediction" output — really a data-grounded recommendation, and it's honest to present it that way rather than as certainty.

---

## Stage 7 — Display & storage

Show in your web app:
- The original query
- A competitor grid (title, brand, price, rating, rank — pulled straight from Keepa data)
- The attribute breakdown (materials/styles as a simple bar chart)
- The Claude-generated synthesis text

Store everything so repeated queries build a growing dataset over time — this is what turns a one-off lookup tool into something with compounding value, similar to how WGSN's edge is really accumulated historical data.

---

## Suggested database schema (simplified)

```sql
CREATE TABLE research_queries (
  id UUID PRIMARY KEY,
  query_text TEXT,
  category TEXT,
  created_at TIMESTAMP
);

CREATE TABLE keepa_results (
  id UUID PRIMARY KEY,
  query_id UUID REFERENCES research_queries(id),
  asin TEXT,
  title TEXT,
  brand TEXT,
  price NUMERIC,
  sales_rank INT,
  rating NUMERIC,
  review_count INT,
  raw_data JSONB
);

CREATE TABLE extracted_attributes (
  id UUID PRIMARY KEY,
  keepa_result_id UUID REFERENCES keepa_results(id),
  material TEXT,
  fit_silhouette TEXT,
  style_details TEXT[],
  price_tier TEXT
);

CREATE TABLE synthesis_reports (
  id UUID PRIMARY KEY,
  query_id UUID REFERENCES research_queries(id),
  report_text TEXT,
  created_at TIMESTAMP
);
```

---

## Tech stack summary

| Layer | Tool |
|---|---|
| Frontend | Your web app (React/Next.js etc.) |
| Backend | Node.js or Python API layer |
| Market data | Keepa API (title search + product data) |
| Analysis | Claude API (3 lightweight calls: filter → extract → synthesize) |
| Database | Postgres (structured data + JSONB for raw Keepa payloads) |
| Hosting | Wherever you're comfortable — this stack has no unusual infra needs |

---

## Cost & rate-limit notes

- Keepa: token-based billing, tokens vary by how much data you request per ASIN (more tokens if you pull offers/stats/history). For a title search + basic product data, cost stays modest — avoid requesting full price history/offers unless you need it, since those cost extra tokens.
- Claude API: three calls per research query (filter, extract, synthesize). Batch the filter/extract calls (send multiple listings in one prompt, as shown above) rather than one call per ASIN — this keeps cost and latency down.
- Keep the filter step first — it's the cheapest way to avoid wasting extraction/synthesis tokens on irrelevant results.

---

## Build order (phased)

1. **Phase 1 — plumbing:** input form → Keepa search → raw results displayed in a table. No LLM yet. Validates the data pipeline works.
2. **Phase 2 — add LLM extraction:** attribute extraction pass, displayed alongside raw results.
3. **Phase 3 — add aggregation + synthesis:** the material/style counts and the final Claude synthesis report.
4. **Phase 4 — persistence & history:** save every query so your dataset compounds, and let yourself compare "denim jacket" queries across different weeks to see how the competitive set shifts.
5. **Phase 5 (optional, later):** revisit vision/image tooling only if you decide to automate trend-spotting itself — that's a separate, harder problem with its own legal considerations, not required for this version.
