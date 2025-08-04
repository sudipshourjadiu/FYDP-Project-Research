It looks like you shared the official Google Cloud Storage pricing page. Here's a clear and updated summary (as of mid‑2025) of each storage tier, along with use cases and additional costs:

---

## 🧺 Google Cloud Storage Classes & Pricing

### Storage Rates (per GB/month in North America)

* **Standard (hot) storage**: \~\$0.020 / GB–month for regional; \~\$0.022 for dual‑region; \~\$0.026 for multi‑region ([CloudZero][1])
* **Nearline (cold‑but‑accessible \~monthly)**: \~\$0.010/GB‑month (region); \~\$0.011 dual; up to \~\$0.015 multi‑region ([CloudZero][1])
* **Coldline (access \~quarterly)**: \~\$0.004/GB‑month region; \~\$0.0044 dual; \~\$0.007 multi‑region ([CloudZero][1])
* **Archive (access \~yearly)**: \~\$0.0022/GB‑month region; \~\$0.0014 dual; \~\$0.0024 multi‑region ([CloudZero][1])

(Some sources round these to \$0.020, \$0.010, \$0.004, and \$0.001–\$0.004 respectively) ([Economize Cloud][2]).

### Retrieval Fees (for Nearline / Coldline / Archive)

* **Nearline**: \~\$0.01 per GB retrieved
* **Coldline**: \~\$0.02 per GB
* **Archive**: \~\$0.05 per GB ([NetApp][3])

(Standard storage has no retrieval fee.)

### Minimum Storage Durations & Availability Guarantees

* **Standard**: no minimum; \~99.99% availability in regions, >99.99% in dual/multi‑regions
* **Nearline**: minimum 30 days; \~99.9% regional, \~99.95% dual/multi
* **Coldline**: minimum 90 days; similar availability to Nearline
* **Archive**: minimum 365 days; \~99.9% regional, \~99.95% dual/multi ([Google Cloud][4])

### Other Cost Factors

* **Network egress** (data out of Google Cloud)
* **Operation requests**, depending on class and type of API calls
* **Early‑deletion penalties** if object removed before its minimum storage window ([Spot.io][5], [wired.com][6])

Google Cloud also provides a Free Tier for limited usage in select regions: e.g. 5 GB of regional storage, 5,000 Class A and 50,000 Class B operations, and 100 GB outbound transfer per month for North America. ([Promevo][7])

---

## 📊 Tier Comparison Overview

| **Storage Class** | **Cost (USD / GB‑mo)** | **Retrieval Fee** | **Minimum Storage** | **Typical Use Case**                |
| ----------------- | ---------------------- | ----------------- | ------------------- | ----------------------------------- |
| Standard          | \~\$0.020–0.026        | None              | None                | Frequently accessed “hot” data      |
| Nearline          | \~\$0.010–0.015        | \~\$0.01/GB       | 30 days             | Monthly‑access cold data, backups   |
| Coldline          | \~\$0.004–0.007        | \~\$0.02/GB       | 90 days             | Quarterly access, archival backups  |
| Archive           | \~\$0.0022–0.0024      | \~\$0.05/GB       | 365 days            | Rare access less than once per year |

---

## ✅ Choosing the Right Class

* If you frequently access data and need flexibility → **Standard** is best.
* If you mostly write and rarely read (e.g. monthly retrievals) → **Nearline** is cost-effective.
* If access is very infrequent (quarterly or less) → **Coldline** offers lower storage fees.
* For long-term retention (yearly or less access), Archive offers the lowest storage cost—with slower and pricier retrieval.

Always balance **storage cost** vs **retrieval and operation fees** based on your access patterns.

---

## 📌 Additional Notes

* **Regional vs Dual/Multi-region**: Dual or multi-region gives higher availability but slightly higher storage costs.
* **SLA & Durability**: All classes offer the same high durability (11 9's) but differ in uptime SLA. ([Google Cloud][4])
* **Legacy classes** (like Multi‑regional, Regional, DRA) embed into these four standard classes and should be avoided for new buckets. ([Google Cloud][4])
