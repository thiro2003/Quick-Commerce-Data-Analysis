# 🛒 Zipto Dark Store DownLane 3: Products & Promotions Analysis

An end-to-end quick commerce data analytics and business intelligence project conducted on **Zipto's Noida Cluster (10 Dark Stores)** analyzing operational performance data from **18 May to 28 Jun 2026**. 

This project explores product sales performance, unit margins, promo code mechanics, and discount leakages, delivering actionable operational recommendations to stop margin burn and scale profitability.

---

## 👥 Contributors & Team

This hackathon project was built collaboratively by:

* **Triveni Chavhan**: [@trivenichavhan731-cmd](https://github.com/trivenichavhan731-cmd)
* **Monika Mengawade**: [@mengawademonika-hash](https://github.com/mengawademonika-hash)
* **Rohit Sul**: Analytics Lead / Team Member ([LinkedIn](https://linkedin.com/in/rohit-sul-780265283))

---

## 📊 Business Context & Key Insights

The project analyzed **2.44 Lakh units sold** across **8 categories**, generating **₹2.67 Cr in Gross Revenue** on delivered orders.

### Key Findings:

1. **Product Performance & Loss-Making SKUs:**
   * **Gross Margin:** 26.5% overall gross margin across 54 SKUs (28 perishable, 26 non-perishable).
   * **Category Weak Spots:** *Meat & Seafood* generated 17.5K units in sales at a low **5.6% margin**, whereas *Snacks & Packaged* earned **39%**.
   * **Best-Seller Trap:** *Lemon Iced Tea* generated ₹9.4L in net revenue (#1 SKU by revenue) but sells below cost.
   * **Negative Margin SKUs:** 6 SKUs lose money at approximately **-9% margin** (e.g., *Butter*, *Chicken Curry Cut*, *Mutton Curry Cut*, *Whole Wheat Bun*, *Orange Juice*, *Lemon Iced Tea*), burning **₹2.86 L** in gross margin (12% of revenue).

2. **Promo Code Performance & Discount Leakages:**
   * **Discount Spend:** ₹14.2 L given in discounts across 11,096 promo orders (22.9% of total delivered orders used a promo code).
   * **Profitability per Discount Rupee:** Overall return was **₹0.08 profit per ₹1 discount spent** (Contribution margin: 2.1% on promo orders vs. 20.8% without promos).
   * **Rule-Breaking / Invalid Usage:** 1,825 delivered orders used promo codes outside their validity windows or below minimum basket size requirements, incurring **₹2.66 L in discount leakages** (3.5% of total orders).
   * **Promo Code Breakdown:**
     * `FRESH50`: Net loss of **₹2.76 L** (-62.6% contribution margin; 61% of its orders broke promo rules).
     * `SAVE20`: **₹4.32 L** in total discount given for a weak 1.7% contribution margin (Volume trap).
     * `FLAT75`: The only code returning positive contribution (**₹1.02 earned per ₹1 discount spent**).

---

## 💡 Strategic Recommendations & Monthly Financial Impact

> *Note: Impact per month calculated as 6-week total ÷ 1.4 months (42 days).*

| # | Action Item | Estimated Impact / Month |
|---|---|---|
| **1** | Stop or redesign `FRESH50` promo code | **+₹1.97 Lakhs** |
| **2** | Reprice or delist the 6 loss-making SKUs (e.g., Lemon Iced Tea, Butter, Curry Cuts) | **+₹2.04 Lakhs** |
| **3** | Reduce `SAVE20` discount from 20% to 15%, scale `FLAT75` | **+₹0.77 Lakhs** |
| **4** | Enforce hard automated checks for promo validity & minimum basket limits at checkout | **+₹0.60 Lakhs** |

---

## 🛠️ Tech Stack & Data Architecture

* **Excel / Power Query:** Data cleaning, transformation, and relational modeling across tables.
* **Power Pivot / DAX:** Data modeling, calculated metrics, explicit measures, and KPI calculations.
* **Power BI / Interactive Dashboards:** Visualizing sales trends, promo ROI, category margins, and SKU performance.
* **Presentation & Documentation:** Executive stakeholder deck (`Insights.pptx`).

---

## 📁 Repository Structure

```text
├── orders.csv            # Raw orders table (order_id, timestamps, status, promo_code, discount)
├── order_items.csv      # Line item details (order_id, sku, quantity, item_price)
├── products.csv         # SKU catalog details (sku, category, mrp, unit_cost, margin_pct)
├── Hackathon.xlsx        # Processed Excel workbook with Power Query & Power Pivot Data Model
├── Insights.pptx         # Executive summary deck for Dark Store DownLane 3
└── README.md             # Project documentation
