# 📊 Retail360 — Qlik Sense BI Dashboards

> Business Intelligence layer for the **Retail360 Data Lakehouse** pipeline. Five interactive Qlik Sense dashboards connect directly to Databricks Unity Catalog Gold layer tables, delivering executive and operational analytics across Sales, Profitability, Customers, and Inventory.

---

## 🔗 Related Repository

| Layer | Repository |
|-------|-----------|
| Data Engineering (Databricks) | [Retail360-Spark-Declarative-Pipeline-Databricks](https://github.com/KarthcikYegambaram/Retail360-Spark-Declarative-Pipeline-Databricks) |
| BI & Dashboards (Qlik Sense) | ⬅️ This repository |

---

## 📌 Project Overview

**Domain:** Retail Sales Analytics  
**Data Source:** Databricks Unity Catalog — `workspace.retail360` (Gold Layer)  
**BI Tool:** Qlik Sense  
**Records:** ~100,000 sales transactions across 4 regions, 6 product categories  
**Date Range:** Jun 2019 – Jan 2024  
**Last Refreshed:** 08-Jun-2026

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│          DATABRICKS UNITY CATALOG (Gold Layer)              │
│                                                             │
│   gold_dim_customer   gold_dim_product   gold_dim_date      │
│                  └──────────┬───────────┘                   │
│                      gold_fact_sales                        │
└──────────────────────┬──────────────────────────────────────┘
                       │  JDBC (Databricks SQL Warehouse)
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              QLIK SENSE LOAD SCRIPTS                        │
│                                                             │
│  01_extract_to_qvd  →  dim_customer.qvd                     │
│                     →  dim_product.qvd                      │
│                     →  fact_sales.qvd                       │
│                                                             │
│  02_load_from_qvd   →  In-memory data model                 │
│  03_bridge_calendar →  BridgeCalendar (3 date types)        │
│  04_master_calendar →  CanonicalCalendar (R12M, YTD flags)  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│             QLIK SENSE DASHBOARDS (5 Sheets)                │
│                                                             │
│  Executive Overview  →  Sales Performance                   │
│  Profitability Analysis  →  Customer Insights               │
│  Inventory & Operations                                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 📊 Dashboard Sheets

| # | Sheet | Key KPIs | Primary Audience |
|---|-------|----------|-----------------|
| 1 | **Executive Overview** | Revenue 2.51G, Profit 375.5M, Margin 15%, MoM Growth -0.17% | C-Suite |
| 2 | **Sales Performance** | Net Sales 1.88G, Orders 100k, AOV 25.08k, Growth 2.1% | Sales Managers |
| 3 | **Profitability Analysis** | Total Profit 375.5M, Margin 15%, Loss Orders 0 | Finance Team |
| 4 | **Customer Insights** | Active Customers 100k, Rev/Customer 18.78k, Top Segment: CONSUMER | CRM / Marketing |
| 5 | **Inventory & Operations** | Avg Ship Days 4, Discount 25.1%, SLA Compliance 71.5% | Ops / Supply Chain |

---

## 🗂️ Data Model

### Source Tables (Databricks Gold Layer)

| Table | Type | Fields Loaded |
|-------|------|--------------|
| `gold_dim_customer` | SCD Type II Dimension | customer_id, full_name, segment, city_type, region, outlet_type, state, country |
| `gold_dim_product` | SCD Type II Dimension | product_id, product_name, sub_category, category_of_goods |
| `gold_fact_sales` | Fact | order_id, sales, quantity, discount, profit, net_sales, profit_margin_pct, shipping_days, ship_mode, order_date, ship_date, sales_date |

### Multi-Date Bridge Pattern

The fact table contains **three date fields** (order_date, ship_date, sales_date). A canonical bridge table resolves this for Qlik's associative model:

```
fact_sales ──→ BridgeCalendar (order_id + LinkDate + DateType)
                      │
                      ▼
               CanonicalCalendar (LinkDate + all time intelligence flags)
```

### Calendar Flags Available

| Flag | Description |
|------|-------------|
| `Rolling12Months` | Last 12 months from max date |
| `AllYearYTDFlag_` | Year-to-date |
| `AllYearQTDFlag_` | Quarter-to-date |
| `AllYearMTDFlag_` | Month-to-date |
| `PrevMTDFlag_` | Previous month-to-date |
| `R12MFlag_` | Rolling 12 months (binary) |
| `PrevR12MFlag_` | Prior rolling 12 months (for MoM/YoY comparison) |

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|-----------|
| BI Platform | Qlik Sense |
| Data Source | Databricks Unity Catalog (Gold Layer) |
| Connection | Databricks JDBC / SQL Warehouse |
| Intermediate Storage | QVD files (lib://DataFiles/) |
| Data Model | Star Schema + Bridge Calendar |
| Load Script | QlikView Script (.qvs) |

---

## ⚙️ Load Script Architecture

```
01_extract_to_qvd.qvs
│  LIB CONNECT TO 'Databricks_dbc-...'
│  SELECT from workspace.retail360.gold_dim_customer
│  SELECT from workspace.retail360.gold_dim_product
│  SELECT from workspace.retail360.gold_fact_sales
│  STORE each table as .qvd → lib://DataFiles/
│
02_load_from_qvd.qvs
│  LOAD dim_customer, dim_product, fact_sales from .qvd files
│  (Fast reload — no live DB hit needed on subsequent runs)
│
03_bridge_calendar.qvs
│  CONCATENATE 3 date types (order/ship/sales) into BridgeCalendar
│  Derive vMinDate / vMaxDate variables
│
04_master_calendar.qvs
   Generate CanonicalCalendar with full time intelligence
   YTD / QTD / MTD / R12M / PrevR12M flags
```

---

## 🚀 How to Set Up

### Prerequisites
- Qlik Sense (Cloud or Enterprise)
- Databricks workspace with Unity Catalog enabled
- `workspace.retail360` Gold layer tables populated

### Steps

1. **Configure connection** — Create a Databricks JDBC data connection in Qlik named `Databricks_dbc-<your-workspace-id>.cloud.databricks.com`
2. **Create DataFiles folder** — Ensure `lib://DataFiles/` exists in your Qlik space for QVD storage
3. **Run full reload** — Execute scripts in order: 01 → 02 → 03 → 04
4. **Subsequent reloads** — Skip script 01 (use cached QVDs) unless Gold layer has been updated
5. **Schedule reload** — Set Qlik reload task to run after each Databricks pipeline execution

---

## 📐 Global Filters (Available on All Sheets)

| Sheet | Filters |
|-------|---------|
| Executive Overview | Year, QuarterYear, MonthYear, category, region, city_type |
| Sales Performance | Year, QuarterYear, MonthYear, category, region, segment |
| Profitability Analysis | Year, QuarterYear, MonthYear, outlet_type, ship_mode, segment |
| Customer Insights | Year, QuarterYear, MonthYear, region, state, outlet_type |
| Inventory & Operations | Year, QuarterYear, MonthYear, ship_mode, category_of_goods, region |

---

## 👤 Author

**Karthick** — Data Engineer / BI Developer  
Built as a portfolio project — BI layer on top of the Retail360 Databricks Lakehouse pipeline.
