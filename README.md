#!/usr/bin/env python3
"""
Zipto Dark Store Down - Lane 3 (Product & Promotions)
=====================================================
Cleans order_items_messy / products_messy / promo_codes_messy, enriches the shared
orders_clean spine with the OFFICIAL contribution definition, reconciles everything,
and produces the Lane 3 findings tables.

Usage
-----
    python lane3_pipeline.py --data ./data --out ./output

Inputs  (in --data):  order_items_messy.csv  products_messy.csv
                      promo_codes_messy.csv  orders_clean.csv
Outputs (in --out):   products_clean.csv  promo_codes_clean.csv  order_items_clean.csv
                      orders_enriched.csv  cleaning_log_lane3.csv
                      findings_*.csv  (one file per finding table)
Requires: pandas, numpy
"""
import argparse
import warnings
from pathlib import Path

import numpy as np
import pandas as pd

warnings.filterwarnings("ignore")
LOG = []  # cleaning log rows, written as we go


def log(file, issue, rows, action, why):
    LOG.append({"file": file, "issue": issue, "rows_affected": int(rows),
                "action": action, "justification": why})


# ----------------------------------------------------------------------------
# 1. PRODUCTS
# ----------------------------------------------------------------------------
def clean_products(path):
    raw = pd.read_csv(path, dtype=str)
    p = raw.copy()
    p["sku"] = p.sku.str.strip().str.upper()
    p["product_name"] = p.product_name.str.strip()

    cat = p.category.str.strip().str.title().str.replace(" And ", " & ", regex=False)
    log("products", "category written in mixed case (FRUITS & VEGETABLES / dairy & eggs ...)",
        (cat != raw.category).sum(), "Normalised to Title Case (8 categories)",
        "Otherwise one category splits into up to 3 groups in every pivot")
    p["category"] = cat

    flag_map = {"yes": 1, "y": 1, "true": 1, "1": 1, "no": 0, "n": 0, "false": 0, "0": 0}
    flag = p.is_perishable.str.strip().str.lower().map(flag_map)
    assert flag.notna().all(), "unmapped is_perishable value"
    log("products", "is_perishable stored as Yes/Y/TRUE/1/No/N/FALSE/0",
        (~raw.is_perishable.isin(["0", "1"])).sum(), "Mapped to 1/0",
        "Verified against shelf_life_days: perishables = 2-7 days, non-perishables = 240-540 days")
    p["is_perishable"] = flag.astype(int)

    for c in ["shelf_life_days", "mrp", "unit_cost"]:
        p[c] = pd.to_numeric(p[c])
    p["margin_pct"] = ((p.mrp - p.unit_cost) / p.mrp).round(4)
    p["negative_margin_flag"] = (p.unit_cost > p.mrp).astype(int)
    log("products", "unit_cost higher than MRP (sold below cost at list price)",
        p.negative_margin_flag.sum(), "KEPT (real business issue, not a data error); flagged in negative_margin_flag",
        "Deleting would hide a finding. Butter, Chicken Curry Cut, Mutton Curry Cut, Whole Wheat Bun, Orange Juice, Lemon Iced Tea")
    assert p.sku.is_unique
    return p


# ----------------------------------------------------------------------------
# 2. PROMO CODES
# ----------------------------------------------------------------------------
def _parse_date(s):
    for fmt in ("%Y-%m-%d", "%d/%m/%Y", "%d-%b-%Y"):
        try:
            return pd.to_datetime(s, format=fmt)
        except (ValueError, TypeError):
            pass
    return pd.NaT


def clean_promos(path):
    raw = pd.read_csv(path, dtype=str)
    pr = raw.copy()
    pr["promo_code"] = pr.promo_code.str.strip().str.upper()
    pr["discount_type"] = pr.discount_type.str.strip().str.upper()
    for c in ["valid_from", "valid_to"]:
        pr[c] = pr[c].map(_parse_date)
    n_iso = raw.valid_from.str.match(r"\d{4}-\d{2}-\d{2}$").sum() + raw.valid_to.str.match(r"\d{4}-\d{2}-\d{2}$").sum()
    log("promo_codes", "valid_from / valid_to in 3 formats (yyyy-mm-dd, dd/mm/yyyy, dd-Mon-yyyy)",
        len(raw) * 2 - n_iso, "Parsed each format explicitly -> ISO dates",
        "dd/mm vs mm/dd ambiguity avoided by explicit format; 18/05/2026 and 28/06/2026 confirm day-first")
    assert pr.valid_from.notna().all() and pr.valid_to.notna().all()
    assert (pr.valid_to >= pr.valid_from).all()
    for c in ["discount_value", "min_order_value"]:
        pr[c] = pd.to_numeric(pr[c])
    pr["applicable_stores"] = pr.applicable_stores.str.replace(" ", "").str.upper()
    return pr


# ----------------------------------------------------------------------------
# 3. ORDER ITEMS
# ----------------------------------------------------------------------------
def clean_items(path, products):
    oi = pd.read_csv(path)
    n0 = len(oi)
    oi["sku"] = oi.sku.str.strip().str.upper()

    exact = oi.duplicated().sum()
    oi = oi.drop_duplicates()
    log("order_items", "exact duplicate rows", exact, "Dropped duplicates (kept first)",
        "Same item counted twice inflates quantity and revenue")

    # same order_item_id twice, copies differ only because one has blank unit_cost
    before = len(oi)
    oi = (oi.assign(_nc=oi.unit_cost.isna()).sort_values(["order_item_id", "_nc"])
            .drop_duplicates("order_item_id", keep="first").drop(columns="_nc"))
    log("order_items", "same order_item_id twice, one copy missing unit_cost", before - len(oi),
        "Kept the copy that has unit_cost", "order_item_id is the primary key")

    neg = (oi.quantity < 0).sum()
    oi["quantity"] = oi.quantity.abs()
    log("order_items", "negative quantity (sign error)", neg, "Set to absolute value",
        "Order gross_amount only reconciles with |qty|: an item of -1 explains a gap of exactly 2x its price")

    oi = oi.merge(products[["sku", "unit_cost"]].rename(columns={"unit_cost": "_pc"}), on="sku", how="left")
    nblank = oi.unit_cost.isna().sum()
    oi["unit_cost"] = oi.unit_cost.fillna(oi._pc)
    oi = oi.drop(columns="_pc")
    log("order_items", "blank unit_cost", nblank, "Filled from products.unit_cost for the same SKU",
        "unit_cost never differs from the product master where both exist")
    assert oi.unit_cost.notna().all() and oi.sku.isin(products.sku).all()

    oi["line_revenue"] = (oi.quantity * oi.unit_price).round(2)
    oi["line_cogs"] = (oi.quantity * oi.unit_cost).round(2)
    print(f"order_items: {n0:,} -> {len(oi):,} rows")
    return oi.sort_values("order_item_id").reset_index(drop=True)


# ----------------------------------------------------------------------------
# 4. ORDERS (shared spine) + OFFICIAL economics
# ----------------------------------------------------------------------------
def enrich_orders(path, promos):
    o = pd.read_csv(path, parse_dates=["order_ts", "delivered_ts"])
    o["promo_code"] = o.promo_code.fillna("NONE")
    # official contribution definition from the problem statement
    o["contribution_official"] = np.select(
        [o.order_status == "delivered", o.order_status == "returned"],
        [o.net_amount - o.cogs_amount - o.delivery_cost, -(o.cogs_amount + o.delivery_cost)], 0.0)
    o["order_date"] = o.order_ts.dt.normalize()

    x = o.merge(promos, on="promo_code", how="left")
    has = x.promo_code != "NONE"
    o["promo_outside_validity"] = (has & ((x.order_date < x.valid_from) | (x.order_date > x.valid_to))).astype(int).values
    o["promo_below_min_order"] = (has & (x.gross_amount < x.min_order_value)).astype(int).values
    allowed = x.applicable_stores.fillna("ALL")
    wrong_store = [bool(h) and a != "ALL" and s not in a.split(",") for h, a, s in zip(has, allowed, x.store_id)]
    o["promo_wrong_store"] = np.array(wrong_store, dtype=int)
    o["promo_invalid_any"] = ((o.promo_outside_validity + o.promo_below_min_order + o.promo_wrong_store) > 0).astype(int)
    log("orders (shared)", "promo_code blank for orders without a promo", (o.promo_code == "NONE").sum(),
        "Filled with 'NONE'", "Lets promo vs no-promo be filtered without null handling")
    return o


def reconcile(o, oi):
    g = oi.groupby("order_id").agg(items_gross=("line_revenue", "sum"), items_cogs=("line_cogs", "sum"))
    chk = o.merge(g, left_on="order_id", right_index=True)
    ok_g = ((chk.items_gross - chk.gross_amount).abs() < 0.05).mean()
    ok_c = ((chk.items_cogs - chk.cogs_amount).abs() < 0.05).mean()
    print(f"RECONCILIATION  gross match: {ok_g:.2%} | cogs match: {ok_c:.2%} | orders without items: {len(o) - len(chk)}")
    assert ok_g == 1.0 and ok_c == 1.0, "items do not reconcile with orders"
    log("order_items vs orders", "reconciliation: SUM(qty*price) and SUM(qty*cost) per order vs gross_amount / cogs_amount",
        len(chk), "PASS - 100% of orders match after cleaning", "Proof that the cleaning rules are right, not just plausible")


# ----------------------------------------------------------------------------
# 5. FINDINGS
# ----------------------------------------------------------------------------
def findings(o, oi, p, months, out):
    D = o[o.order_status == "delivered"]
    promo = o.promo_code != "NONE"
    res = {}

    # A. what we sell
    m = oi.merge(o[["order_id", "order_status", "gross_amount", "net_amount"]], on="order_id").merge(
        p[["sku", "product_name", "category", "is_perishable", "negative_margin_flag"]], on="sku")
    d = m[m.order_status == "delivered"].copy()
    d["net_alloc"] = d.line_revenue / d.gross_amount * d.net_amount
    cat = d.groupby("category").agg(units=("quantity", "sum"), revenue=("line_revenue", "sum"), cogs=("line_cogs", "sum"))
    cat["gross_margin_pct"] = (1 - cat.cogs / cat.revenue).round(4)
    res["findings_category"] = cat.sort_values("gross_margin_pct")
    sku = d.groupby("product_name").agg(units=("quantity", "sum"), revenue=("line_revenue", "sum"),
                                         net_revenue=("net_alloc", "sum"), orders=("order_id", "nunique"))
    res["findings_top_products"] = sku.sort_values("net_revenue", ascending=False)

    neg = d[d.negative_margin_flag == 1]
    negsku = neg.groupby("product_name").apply(lambda x: pd.Series(
        {"units": x.quantity.sum(), "loss_before_discount": (x.line_revenue - x.line_cogs).sum()}))
    negsku.loc["TOTAL"] = negsku.sum()
    res["findings_below_cost_skus"] = negsku

    # B. what discounts cost (OFFICIAL contribution)
    g = o.groupby("promo_code").agg(orders=("order_id", "count"),
                                    returned=("order_status", lambda s: (s == "returned").sum()),
                                    contribution_official=("contribution_official", "sum"))
    g["discount_on_delivered"] = D.groupby("promo_code").discount_amount.sum()
    g["net_delivered"] = D.groupby("promo_code").net_amount.sum()
    g["margin_pct_of_net"] = (g.contribution_official / g.net_delivered).round(4)
    g["return_rate"] = (g.returned / g.orders).round(4)
    res["findings_by_promo"] = g

    # C. where it concentrates
    pv = o[promo]
    res["findings_promo_orders_by_store"] = pv.pivot_table(index="store_id", columns="promo_code", values="order_id",
                                                           aggfunc="count", fill_value=0)
    res["findings_promo_contribution_by_store"] = pv.pivot_table(index="store_id", columns="promo_code",
                                                                 values="contribution_official", aggfunc="sum", fill_value=0).round(0)
    st = o.assign(promo=promo).groupby("store_id").agg(orders=("order_id", "count"), promo_share=("promo", "mean"),
                                                       return_rate=("order_status", lambda s: (s == "returned").mean()),
                                                       first_order=("order_ts", "min"))
    res["findings_store_promo_share"] = st.round(4)

    # D. ₹ impact per month (arithmetic is printed so it can be rebuilt live)
    f50 = o[o.promo_code == "FRESH50"]
    f50_ab = f50[f50.store_id.isin(["S03", "S07"])]
    inv = D[D.promo_invalid_any == 1]
    f50_inv = inv[inv.promo_code == "FRESH50"]
    rows = [
        ("Promo orders lose money overall (official contribution)", pv.contribution_official.sum()),
        ("FRESH50 total contribution", f50.contribution_official.sum()),
        ("FRESH50 at S03+S07 only", f50_ab.contribution_official.sum()),
        ("Discount given on INVALID promo orders (delivered)", -inv.discount_amount.sum()),
        ("  of which FRESH50 below its Rs800 minimum", -f50_inv.discount_amount.sum()),
        ("Below-cost SKUs: loss before any discount (delivered)", negsku.loc["TOTAL", "loss_before_discount"]),
    ]
    imp = pd.DataFrame(rows, columns=["item", "total_over_data_period"])
    imp["per_month"] = (imp.total_over_data_period / months).round(0)
    imp["arithmetic"] = [f"{t:,.0f} / {months:.1f} months" for t in imp.total_over_data_period]
    res["findings_rupee_impact"] = imp.set_index("item")

    # print headline numbers
    print("\n========== HEADLINES ==========")
    print(f"Data period: {o.order_ts.min().date()} to {o.order_ts.max().date()} = {months*30:.0f} days = {months:.2f} months")
    print(f"Official contribution, all orders      : Rs {o.contribution_official.sum():,.0f}")
    print(f"  promo orders                         : Rs {pv.contribution_official.sum():,.0f}  ({pv.contribution_official.sum()/D[D.promo_code!='NONE'].net_amount.sum():.2%} of delivered net)")
    print(f"  non-promo orders                     : Rs {o[~promo].contribution_official.sum():,.0f}  ({o[~promo].contribution_official.sum()/D[D.promo_code=='NONE'].net_amount.sum():.2%})")
    print(f"Discount on delivered promo orders     : Rs {D.discount_amount.sum():,.0f}")
    print(f"Promo contribution per Rs1 of discount : {pv.contribution_official.sum()/D.discount_amount.sum():.3f}")
    print(f"FRESH50 orders in S03+S07              : {len(f50_ab):,} of {len(f50):,} ({len(f50_ab)/len(f50):.0%})")
    print(imp.to_string())

    out.mkdir(parents=True, exist_ok=True)
    for name, df in res.items():
        df.to_csv(out / f"{name}.csv")
    return res


# ----------------------------------------------------------------------------
def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--data", default="./data")
    ap.add_argument("--out", default="./output")
    a = ap.parse_args()
    data, out = Path(a.data), Path(a.out)
    out.mkdir(parents=True, exist_ok=True)

    p = clean_products(data / "products_messy.csv")
    pr = clean_promos(data / "promo_codes_messy.csv")
    oi = clean_items(data / "order_items_messy.csv", p)
    o = enrich_orders(data / "orders_clean.csv", pr)
    reconcile(o, oi)

    days = (o.order_ts.max().normalize() - o.order_ts.min().normalize()).days + 1
    months = days / 30.0  # agree this with Lane 1 (rent x months)
    findings(o, oi, p, months, out)

    p.to_csv(out / "products_clean.csv", index=False)
    pr.to_csv(out / "promo_codes_clean.csv", index=False)
    oi.to_csv(out / "order_items_clean.csv", index=False)
    o.to_csv(out / "orders_enriched.csv", index=False)
    pd.DataFrame(LOG).to_csv(out / "cleaning_log_lane3.csv", index=False)
    print(f"\nWrote cleaned files, cleaning log and findings to {out.resolve()}")


if __name__ == "__main__":
    main()
