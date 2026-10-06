# 🛒 Zipto Dark Store DownLane 3: Products & Promotions Analysis

An end-to-end data analytics and business intelligence project conducted on **Zipto's Noida Cluster (10 Dark Stores)** analyzing performance data across **18 May to 28 Jun 2026**. 

This project explores product sales performance, product margins, promo code mechanics, and offer leakages, delivering actionable operational recommendations to stop margin burn and scale revenue.

---

## 👥 Contributors & Team

This hackathon project was built collaboratively by:

* **Triveni Chavhan**: [@trivenichavhan731-cmd](https://github.com/trivenichavhan731-cmd)
* **Monika Mengawade**: [@mengawademonika-hash](https://github.com/mengawademonika-hash)
* **Rohit Sul**: Analytics Lead / Team Member

---

## 📊 Business Context & Key Insights

The objective of this project was to analyze **2.44 Lakh units sold** across **8 categories** yielding **₹2.67 Cr in Gross Revenue** on delivered orders.

### Key Findings:
1. **Product Performance & Loss-Making SKUs:**
   * **Gross Margin:** 26.5% overall gross margin across 54 SKUs (28 perishable, 26 non-perishable).
   * **Category Weak Spots:** *Meat & Seafood* generated 17.5K units in sales at a low **5.6% margin**, whereas *Snacks & Packaged* earned **39%**.
   * **Best-Seller Trap:** *Lemon Iced Tea* generated ₹9.4L in net revenue (#1 by revenue) but sells below cost.
   * **Negative Margin SKUs:** 6 SKUs lose money at approximately **-9% margin**, burning **₹2.86 L** in gross margin (12% of revenue).

2. **Promo Code Performance & Discount Cost:**
   * **Discount Spend:** ₹14.2 L given in discounts across 11,096 promo orders (22.9% of delivered orders).
   * **Profitability per Discount Rupee:** Overall return was **₹0.08 profit per ₹1 discount spent** (Contribution margin: 2.1% with promo vs. 20.8% without).
   * **Rule-Breaking / Invalid Usage:** 1,825 delivered orders used promo codes outside their validity windows or below minimum basket sizes, incurring **₹2.66 L in leakages**.
   * **Promo Breakdown:**
     * `FRESH50`: Net loss of **₹2.76 L** (-62.6% contribution margin; 61% of its orders broke promo rules).
     * `SAVE20`: **₹4.32 L** discount given for a weak 1.7% contribution margin.
     * `FLAT75`: Only code returning positive contribution (**₹1.02 earned per ₹1 spent**).

---

## 💡 Strategic Recommendations & Monthly Financial Impact

| # | Action Item | Estimated Impact / Month |
|---|---|---|
| **1** | Stop or redesign `FRESH50` promo code | **+₹1.97 Lakhs** |
| **2** | Reprice or delist the 6 loss-making SKUs (e.g., Lemon Iced Tea, Butter, Chicken/Mutton Curry Cut) | **+₹2.04 Lakhs** |
| **3** | Reduce `SAVE20` discount from 20% to 15%, scale `FLAT75` | **+₹0.77 Lakhs** |
| **4** | Enforce hard checks for promo validity & minimum basket limits at checkout | **+₹0.60 Lakhs** |

---

## 🛠️ Tech Stack & Tools Used

* **Excel / Power Query:** Data cleaning, data transformation, and modeling.
* **Power Pivot / DAX:** Data modeling, calculated measures, explicit filters, and KPI calculations.
* **Power BI / Dashboards:** Interactive visualization of sales, promo ROI, and category performance.
* **Presentation & Documentation:** Key stakeholder presentation slides (`Insights.pptx`).

---

## 📁 Repository Structure

```text
├── Hackathon.xlsx        # Cleaned datasets, Power Query models, and Data Tables
├── Insights.pptx         # Executive summary deck for Dark Store DownLane 3
└── README.md             # Project documentation
