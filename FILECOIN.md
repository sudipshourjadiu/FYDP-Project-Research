If **Option A (Web3.Storage)** is too expensive for your use case (which is typical for large surveillance archives), **Option B (Spade + Boost)** is a better fit, especially when you need:

* **Low cost**
* **Flexible retention** (e.g., 1–3 months)
* **Cold archive** (rarely accessed)
* **Full control over providers and deal terms**

---

## ✅ Summary: Use Filecoin via Spade + Boost for 1–3 Month Retention

Here’s how you can use **Spade** (for automation) and **Boost** (for making deals) to store surveillance video at low cost with **explicit retention control**.

---

## 🔧 Tools You’ll Use

| Tool                               | Purpose                                                   |
| ---------------------------------- | --------------------------------------------------------- |
| **Singularity**                    | Convert video files to CAR format                         |
| **Boost**                          | Filecoin deal-making engine (replaces older Lotus Market) |
| **Spade**                          | Automates uploading & **expiration-based renewal**        |
| **Filecoin node (Lotus/Boost SP)** | Interact with network (can be local or remote)            |

---

## 🗃️ Step-by-Step Setup for Retention-Controlled Cold Storage

### **1. Convert Files to CAR Format**

Use **Singularity CLI**:

```bash
singularity prep --input-dir ./videos --output-dir ./cars
```

* This will package your video files into **CAR (Content Addressable Archives)**.

---

### **2. Configure Spade for Temporary Retention**

Spade uses config files to determine:

* How long a file should be stored
* When to **renew** or let the deal **expire**

🔧 Example config for 1–3 month retention:

```toml
[dataset.surveillance]
source_path = "./cars"
retention_days = 90  # or 30 for one month
renewal_enabled = false  # Do NOT auto-renew
```

* With `renewal_enabled = false`, Spade won't renew the deal after expiration.
* Filecoin deals will naturally **expire** after the term.

---

### **3. Submit Deals via Boost**

You can run Boost locally or interact with a remote Boost storage provider.

Spade will handle the deal creation with:

* Provider negotiation
* Deal size optimization
* Failover retries

Optionally specify **trusted providers** (SPs) offering:

* Cheap short-term deals
* Verified deal acceptance (if you have Filecoin Plus)

---

### **4. Monitor Your Deals**

Use:

```bash
spade status
```

Or query Boost’s UI/API for deal lifecycle status.

---

## 💰 Storage Cost Tips

* **Use verified deals (Filecoin Plus):**
  These have *zero or near-zero cost*, but require:

  * Applying for a datacap via KYC
  * Storing public-interest data (possibly not ideal for private surveillance)

* **Target low-cost providers**:
  Some SPs offer **\$0.0005–0.002/GB/month**. You can pre-filter by:

  * Country/region (e.g., lower-cost SPs in Asia or Eastern Europe)
  * Deal duration

---

## 📤 Retrieval (If Needed)

You’ll need:

* The **Piece CID** (from deal log)
* Use Boost or lotus client to retrieve via SP
* Retrieval may take **hours** (cold storage)

---

## 🧪 Prototype Setup (Optional)

I can provide a **ready-to-use Spade config + Singularity CLI script**, like this:

```bash
#!/bin/bash
# Video backup pipeline

INPUT_DIR="./videos"
CAR_DIR="./cars"

# Step 1: Prepare CAR files
singularity prep --input-dir "$INPUT_DIR" --output-dir "$CAR_DIR"

# Step 2: Run Spade to submit short-term deals
spade run --dataset surveillance
```

---

## 🧩 Optional: Hybrid Setup

For active recent data (0–7 days): store on **local disk** or **Wasabi/Backblaze**

For older data (8–90 days): store on **Filecoin via Spade+Boost**
