It looks like you've shared the Filecoin Docs page about **Filecoin Programs and Tools for Storage Providers** — here's a structured and cleaner overview of what's covered:

---

## 📦 Filecoin Programs & Tools (for Storage Providers)

This Filecoin documentation outlines multiple programs and platform tools designed to help **storage providers** (SPs) receive **verified deals** and streamline deal workflows:

### 🔧 Core Tools

* **Web3.storage**:
  A user-friendly onboarding protocol that uses Elastic IPFS to ensure scalability and performance. Developers can upload data to Web3.storage, which handles Car file creation and publishes deals to the Filecoin network for long-term storage. ([docs.filecoin.io][1])

* **Filecoin Green**:
  Initiatives focused on environmental sustainability, promoting carbon-neutral storage operations. It includes tools like the Filecoin Energy Dashboard and energy validation processes to help SPs decarbonize. ([docs.filecoin.io][1])

* **Spade**:
  A deposit-renewal automation tool that maintains long-lived storage deals by renewing them before expiry—ideal for datasets needing continual availability. ([docs.filecoin.io][1])

* **Singularity**:
  A full-featured ingestion pipeline to onboard large-scale (PiB-level) datasets, bringing together data prep, CAR packaging, wallet operations, and seamless deal submissions. It integrates with over 40 storage systems and supports both push/pull modes. ([docs.filecoin.io][1])

---

### 🧱 Partner Platforms and Ecosystem Tools

* **Akave**:
  A platform combining Filecoin-based storage with data encryption and modular design—suited for machine learning pipelines, AI datasets, and SaaS apps. ([Reddit][2])

* **CIDGravity**:
  A SaaS solution handling client onboarding, dynamic pricing, provider matching, and deal monitoring. Fully integrates with Boost and other deal-making engines. ([docs.filecoin.io][1])

* **Swan (FilSwan)**:
  Cross-chain cloud computing and storage gateway enabling institutions and enterprises to interact with Filecoin with minimal friction. Supports cloud-like APIs and provider-search pipelines. ([docs.filecoin.io][1])

* **Open Panda** (historical):
  A platform (now deprecated) that enabled access to large open datasets stored on Filecoin, oriented toward researchers and analysts. ([docs.filecoin.io][1])

---

### ⚠️ Former Tools (Deprecated/Retired)

Several programs have been sunset or replaced, including:

* **Evergreen**: decked the Slingshot program to renew open-dataset deals indefinitely; replaced by Spade. ([docs.filecoin.io][1])
* **CO2.Storage**: focused on environmental data storage schema solutions; alpha stage, never fully launched. ([docs.filecoin.io][1])
* **Filecoin Tracker**: deprecated April 20, 2024; formerly provided network dashboards. ([docs.filecoin.io][1])
* **Slingshot**: earlier open‑dataset onboarding community, replaced by Singularity workflows. ([docs.filecoin.io][1], [docs.filecoin.io][3])
* **Dataprograms.org**, **Moonlanding**, **Filecoin Dataset Explorer**, and **Big Data Exchange**: are no longer active or have evolved. ([docs.filecoin.io][1])

---

### 🧠 How These Tools Help Storage Providers

Together, these programs:

* Simplify or automate onboarding of **verified data** (Filecoin Plus)
* Reduce technical barriers with tools like **Singularity** and **Web3.storage**
* Enable dynamic deal management with **Spade** and ecosystem platforms like **CIDGravity**, **Akave**, and **Swan**
* Support sustainable storage with **Filecoin Green**

They aim to support efficient, reliable, and impactful data storage aligned with Filecoin’s mission of preserving humanity’s important data.

---

### ✅ Next Steps — Tools Matched to Storage Provider Needs

| Need / Use Case                                | Recommended Tool(s)               |
| ---------------------------------------------- | --------------------------------- |
| Basic data onboarding and developer-friendly   | **Web3.storage**, **Singularity** |
| Automated deal renewal and incident management | **Spade**                         |
| Client & provider matchmaking, pricing labs    | **CIDGravity**                    |
| Hybrid cloud and cross-chain integrations      | **Swan / FilSwan**, **Akave**     |
| Sustainable operations and carbon accounting   | **Filecoin Green**                |

---

If you have a specific goal—like onboarding enterprise datasets, running your own SP node, or participating in Filecoin Plus—I can help guide you to the most relevant tools or detailed setup steps. Just let me know what you're trying to accomplish!

[1]: https://docs.filecoin.io/storage-providers/filecoin-deals/filecoin-programs?utm_source=chatgpt.com "Filecoin programs and tools | Filecoin Docs"
[2]: https://www.reddit.com/r/filecoin/comments/1bt7bjy?utm_source=chatgpt.com "How to actually use filecoin for intended purpose"
[3]: https://docs.filecoin.io/storage-providers/filecoin-deals/storage-deals?utm_source=chatgpt.com "Storage deals | Filecoin Docs"


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

---

Would you like:

* A full setup guide with Docker for Boost/Spade?
* Help finding SPs offering cheap 30–90 day deals?
* A template config and bash scripts for automation?

Let me know how hands-on you'd like the prototype to be.

---

Great — let’s step back and explain **how Spade + Boost + Filecoin work conceptually**, especially in the context of your **surveillance video storage** use case.

---

## 🧠 1. **What is Filecoin?**

Filecoin is a **decentralized storage network** where you can pay storage providers (SPs) around the world to store your data for a specific **duration** (e.g., 1 month, 3 months, 1 year). It's optimized for **cold storage** — data that's stored reliably but rarely accessed.

* You don’t upload files directly.
* You make **“deals”** — agreements to store a data file for a certain **price and duration**.

---

## 🧠 2. **What is Boost?**

Boost is a tool used to **negotiate and send storage deals** to providers on the Filecoin network. Think of it as a "Filecoin upload engine" that talks to the network, proposes deals, and manages confirmations.

With Boost:

* You tell it *what file* to store, *where*, *for how long*, and *how much you're willing to pay*.
* It packages this into a **storage deal** and sends it to a provider.

Boost replaces an older, more manual process.

---

## 🧠 3. **What is Spade?**

Spade is an **automation layer** built on top of Boost (and other tools). It manages:

* Finding files you want to store
* Scheduling when to make deals
* Tracking which files are already stored
* Optionally **renewing** deals when they’re about to expire

Think of it like a robot assistant:

* You tell it "Store these files for 1 month"
* It makes sure that happens — and nothing more
* You can tell it to **not renew** deals, so after 1 month, the data simply disappears

---

## 🧠 4. **How Do Deals Work?**

Each deal has:

* **CID** (a unique ID for your content)
* **Provider** (who stores your data)
* **Price** (per GiB per day)
* **Duration** (how long the provider must hold your data)

For example:

| CID       | Provider | Price             | Duration | Ends in |
| --------- | -------- | ----------------- | -------- | ------- |
| abc123... | SP-xyz   | 0.0005 FIL/GB/day | 90 days  | Nov 2   |

This is **immutable**: once made, it’s on the Filecoin blockchain.

---

## 🧠 5. **Where Does the Data Actually Go?**

Your file is first:

1. **Split and compressed** into a special format (CAR = Content Addressable Archive)
2. Uploaded to a **storage provider** who runs big hard drives and specialized Filecoin software
3. The storage deal is **verified on-chain**, proving:

   * The provider received the data
   * They’ll store it as agreed
4. If the provider loses your data, they get **penalized**

---

## 🧠 6. **How Does Retention Work?**

Here’s where your use case is special — you want **temporary storage**.

* In Filecoin, every deal has an **expiration date**.
* If you make a deal for **90 days**, after 90 days:

  * The provider is **no longer obligated** to keep it.
  * Your data may be deleted automatically, or overwritten.

So:

* Want 1-month retention? Set deals for 30 days.
* Want 3 months? Set them for 90 days.
* Want auto-renew? Use Spade to **recreate** the deal before it expires.

---

## 🧠 7. **Why is This Useful for Surveillance Video?**

* Surveillance videos take up **lots of space**
* Most are **not watched**, just stored "in case"
* You only need to **retain them for 1–3 months**
* Filecoin lets you store them **cheaply**, with predictable expiration

This avoids:

* Expensive AWS/S3 bills
* Complicated lifecycle rules
* Manual deletion

---

## 🧠 8. **How Does Retrieval Work?**

Filecoin is cold storage:

* Retrieval is possible, but **not instant**
* You’ll need the **CID** and may wait hours
* Some providers require **payment** for retrieval

So it’s ideal when:

* You mostly **store**, and **rarely fetch**

---

## 🧠 9. **The Role of You (the User)**

You act as the **client**:

* Decide what data to store
* Choose duration and cost
* Monitor your storage deals
* Retrieve when needed (rarely)

You **don’t have to trust** any single provider — the blockchain enforces rules.

---

## 🎯 Summary — What’s Actually Happening?

1. 🎥 You create video files (surveillance footage).
2. 📦 You convert and package those files into CAR files.
3. 🤝 You make a deal with a provider using Boost.
4. 📅 Spade tracks that deal — and *lets it expire* after 1–3 months.
5. 🗂️ Your video is stored at low cost, and auto-deleted later.

**No need to manage servers or delete files manually.**
