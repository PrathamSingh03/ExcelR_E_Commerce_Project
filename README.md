# Olist Store Analysis: 5 business KPIs answered and cross-checked in Excel, MySQL, Power BI and Tableau

![Excel](https://img.shields.io/badge/Excel-217346?logo=microsoftexcel&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-00758F?logo=mysql&logoColor=white)
![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?logo=powerbi&logoColor=black)
![Tableau](https://img.shields.io/badge/Tableau-E97627?logo=tableau&logoColor=white)

A team-based course project on the Olist Brazilian e-commerce dataset (9 CSV files, 99,441 orders). The team answered the same five business questions in four tools and compared the results. This repository holds the full analysis. My own contributions are marked **[my work]** and listed in [My role and contributions](#13-my-role-and-contributions).

**Contents:** [1 Summary](#1-summary) · [2 Problem](#2-business-problem-and-objective) · [3 Dataset](#3-dataset) · [4 KPIs](#4-the-5-business-kpis) · [5 Approach](#5-approach-by-module) · [6 Cleaning and modelling](#6-data-cleaning-and-modelling-notes) · [7 Results](#7-results-and-cross-tool-validation) · [8 Lessons](#8-data-quality-lessons) · [9 Insights](#9-key-business-insights) · [10 Screenshots](#10-dashboard-screenshots) · [11 Reproduce](#11-how-to-reproduce) · [12 Skills](#12-skills-demonstrated) · [13 My role](#13-my-role-and-contributions) · [14 Credit](#14-dataset-credit-and-licence-note)

---

## 1. Summary

Five KPIs (weekday vs weekend payments, 5-star orders paid by credit card, pet_shop delivery time, Sao Paulo price and payment values, shipping days vs review score) were built in **Excel, MySQL, Power BI and Tableau** from the same source data. Comparing the four tools caught a wrong number early: an Excel figure of 42,581 for KPI 2 was replaced by 43,981 after SQL and Power BI DAX both returned 43,981.

## 2. Business problem and objective

Olist is a Brazilian online marketplace. The project asks five operational and customer-experience questions about its orders:

- How do payments differ between weekdays and weekends?
- How many 5-star orders were paid by credit card?
- How long does delivery take for the pet_shop category?
- What do customers in Sao Paulo pay on average?
- Does longer shipping time go with lower review scores?

**Objective:** answer each question with a verified number, turn it into a visual and a short business insight, and cross-check every number across four tools so the final figures can be trusted.

## 3. Dataset

**Source:** [Brazilian E-Commerce Public Dataset by Olist (Kaggle)](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce). About 45 MB zipped. Orders span 2016-09-04 to 2018-10-17. The raw data is **not** stored in this repository (see [`data/README.md`](data/README.md)).

| Table (CSV file) | Rows | What it holds |
| --- | ---: | --- |
| `olist_orders_dataset` | 99,441 | One row per order, with status and purchase / delivery timestamps |
| `olist_customers_dataset` | 99,441 | One row per order-level `customer_id` (96,096 unique customers), city, state, zip prefix |
| `olist_order_items_dataset` | 112,650 | One row per item in an order: product, seller, price, freight |
| `olist_order_payments_dataset` | 103,886 | One row per payment record: type, installments, value (99,440 distinct orders) |
| `olist_order_reviews_dataset` | 99,224 | Review score (1 to 5) and optional comment |
| `olist_products_dataset` | 32,951 | Product category and size / weight attributes |
| `olist_sellers_dataset` | 3,095 | Seller city, state, zip prefix |
| `olist_geolocation_dataset` | 1,000,163 | Latitude / longitude, many rows per zip prefix |
| `product_category_name_translation` | 71 | Category name, Portuguese to English |

**Relationship summary**

```text
customers ──customer_id──► orders ◄──order_id── order_items ──product_id──► products ──category name──► category_translation
                             ▲                      │
                             │                      └──seller_id──► sellers
              order_id ──────┴────── order_id
         order_payments       order_reviews

customers ──zip prefix──► geolocation (zip lookup) ◄──zip prefix── sellers
```

| Join column | Connects |
| --- | --- |
| `order_id` | orders, order_items, order_payments, order_reviews |
| `customer_id` | orders, customers |
| `product_id` | order_items, products |
| `seller_id` | order_items, sellers |
| `product_category_name` | products, product_category_name_translation |
| zip code prefix | customers and geolocation, sellers and geolocation |

![Data model](images/data_model_overview.png)

## 4. The 5 business KPIs

| # | KPI |
| --- | --- |
| 1 | Weekday vs weekend payment statistics (by `order_purchase_timestamp`) |
| 2 | Number of orders with review score 5 and payment type credit card |
| 3 | Average number of days to deliver (`order_delivered_customer_date`) for the `pet_shop` category |
| 4 | Average price and payment value for customers in Sao Paulo city |
| 5 | Relationship between shipping days (delivered date minus purchase date) and review scores |

## 5. Approach by module

The same five KPIs were answered in each module, then compared. The modules ran in this order: Excel, SQL and Power BI, Tableau.

### Module 1: Excel (Power Query, Data Model, PivotTables)  → [`excel/`](excel/)

- **[my work]** Cleaned and loaded all 9 tables in Power Query and built the Data Model: 9 relationships plus 2 helper zip-lookup tables.
- KPIs were built as PivotTables from the Data Model, with distinct counts for orders because one order can have several payment rows.
- Key decisions: keep all nulls in orders and reviews (they carry meaning), set 610 blank product categories to `unknown`, never connect the raw geolocation table.
- Not published here: the workbook is about 132 MB and contains the raw data. The Power Query code and method notes are in [`excel/`](excel/).

### Module 2: MySQL  → [`sql/`](sql/)

- **[my work]** Built the `olist_store` database in MySQL 8: 10 tables (9 source tables plus `geo_zip_lookup`), primary keys, 9 foreign keys, data loaded with `LOAD DATA LOCAL INFILE`, row counts checked against the source.
- **[my work]** Cleaning: 610 blank categories updated to `unknown` (after adding an `unknown` row to the translation table so the foreign key holds). Nulls in orders and reviews left as they are.
- **[my work]** KPI 2 query. The other four KPI queries are the team's.
- Files: [`01_schema.sql`](sql/01_schema.sql), [`02_load_data.sql`](sql/02_load_data.sql), [`03_kpi_queries.sql`](sql/03_kpi_queries.sql).

### Module 3: Power BI  → [`powerbi/`](powerbi/)

- **[my work]** Built the data model from a fresh import of the 9 CSV files (independent of the SQL database): 7 core relationships plus 2 geo relationships through two zip-lookup tables, 610 blank categories set to `unknown`, data types checked.
- **[my work]** KPI 2 measures in DAX (`CROSSFILTER ... BOTH` so reviews and payments filter through orders and each order is counted once). The other KPI visuals and the dashboard page are the team's.
- Measures are listed in [`powerbi/dax_measures.md`](powerbi/dax_measures.md).

### Module 4: Tableau  → [`tableau/`](tableau/)

- **[my work]** Built the data model with Tableau Relationships (not joins) across all 9 tables, plus the shared calculated fields (delivery days, weekday / weekend type, order total payment, clean product category).
- The KPI worksheets and the final dashboard are the team's.
- Data source: the 9 tables exported from the Excel-cleaned data. The raw review file did not parse correctly in Tableau's text connector (unescaped quotes and commas inside multi-line review text produced 14 fields instead of 7), so the cleaned tables were used instead.

## 6. Data cleaning and modelling notes

**One-to-many traps (the main modelling risk):**

| Trap | Why it matters | How it was handled |
| --- | --- | --- |
| orders to payments | 103,886 payment rows for 99,440 orders (installments / split payments). A plain count over-counts orders | `COUNT(DISTINCT order_id)` / `DISTINCTCOUNT` everywhere |
| orders to items | One row per item, 112,650 items | Averages for KPI 4 are shown both row-level and per-order (see section 7) |
| orders to reviews | A few orders have more than one review (547 in the SQL copy) | Distinct order counts; Excel KPI 5 keeps the latest review per order |
| zip prefix | Many geolocation rows per zip prefix cannot sit on the "one" side of a relationship | Zip lookup tables with one row per prefix; raw geolocation left unconnected |
| two paths between tables | Excel and Power BI allow only one active filtering path between two tables | Two separate zip lookup tables, one for customers and one for sellers |

**Missing and undelivered data:**

- **2,965 orders have no delivery date** (never delivered). They were kept, not deleted or filled. They are excluded only from the delivery-time KPIs (3 and 5) and stay in the others.
- **610 products have no category**: set to `unknown`. The first Power Query attempt missed one row because it held an empty string rather than a null; the final step checks for null, empty and whitespace.
- **Blank review comments** are real "no comment" information and were left as they are.
- **162 zip prefixes** used by customers (157) and sellers (5) have no match in the raw geolocation table. In MySQL they were added as placeholder rows with null coordinates so the foreign keys hold.
- **Geolocation in Excel.** My cleaning log counts 261,831 duplicate rows in the raw geolocation table. The query saved in the final workbook removes duplicates on `geolocation_city`, which leaves 8,008 rows (raw: 1,000,163), and the zip lookup tables hold 6,011 rows. No KPI uses geolocation. [TODO: confirm the intended rule, see "Open items" at the end of this file.]
- **Row counts:** MySQL holds 99,223 review rows; the source CSV and the Excel / Tableau data hold 99,224. The single missing review belongs to an order paid by boleto, so KPI 2 is unaffected. [TODO: check the load.]

## 7. Results and cross-tool validation

Monetary values are in the dataset's currency, Brazilian reais (R$). [TODO: the project slides and notes display a rupee sign; align the symbol before publishing.]

| KPI | Excel | MySQL | Power BI | Tableau |
| --- | --- | --- | --- | --- |
| **1** Weekday / weekend orders | 76,593 / 22,847 | 76,593 / 22,847 | 76,593 / 22,847 | 76,594 / 22,847 |
| **1** Avg payment, weekday / weekend | 154.44 / 152.95 | 154.44 / 152.95 | 154.44 / 152.95 | 154.44 / 152.95 |
| **2** 5-star orders paid by credit card | ~~42,581~~ (corrected) | **43,981** | **43,981** | **43,981** |
| **3** Avg delivery days, pet_shop | 11.23 | 11.31 | 11.31 | 11.31 |
| **3** Avg delivery days, all orders | not built | 12.50 | 12.50 | 12.50 |
| **4** Avg price, Sao Paulo | 107.53 | 112.58 | 112.58 | 107.5 |
| **4** Avg payment value, Sao Paulo | 135.83 | 138.74 | 138.74 | 135.8 |
| **5** Avg shipping days, score 1 to 5 | 21.32 to 10.68 | 21.25 to 10.63 | 21.25 to 10.63 | 21.25 to 10.62 |
| **5** Correlation | -0.334 | not built | not built | not built |

**Why the tools differ (all differences are small and explained):**

- **KPI 1, Tableau 76,594:** Tableau Relationships keep one order that has no payment record; the inner joins in Excel, SQL and Power BI drop it (99,440 vs 99,441 orders).
- **KPI 2:** see [section 8](#8-data-quality-lessons).
- **KPI 3:** 11.31 is the order-level average in whole calendar days (`DATEDIFF`), 1,688 delivered pet_shop orders. The Excel figure is 11.23. [TODO: I could not reproduce 11.23 from the raw data; an order-level average gives 11.31 in whole days or 11.37 in fractional days.]
- **KPI 4:** two valid averaging methods. Excel and Tableau average over item and payment rows (107.53 / 135.83, 15,540 orders). SQL and Power BI average per order first, then across orders (112.58 / 138.74, on Sao Paulo orders that have both items and payments). The project notes treat the row-level version as the independently verified reference.
- **KPI 5:** Excel uses fractional days and keeps the latest review per order (95,824 orders). SQL, Power BI and Tableau use whole days via `DATEDIFF`; in SQL an order with more than one review is counted under each of its scores, so the per-score counts add up to 96,018.

**KPI 5 by review score**

| Score | Excel days (orders) | MySQL days (orders) | Tableau days |
| ---: | --- | --- | --- |
| 1 | 21.32 (9,353) | 21.25 (9,384) | 21.25 |
| 2 | 16.63 (2,919) | 16.61 (2,938) | 16.60 |
| 3 | 14.26 (7,916) | 14.20 (7,943) | 14.21 |
| 4 | 12.31 (18,894) | 12.25 (18,943) | 12.25 |
| 5 | 10.68 (56,742) | 10.63 (56,810) | 10.62 |

The Power BI chart matches the SQL values for all five scores.

## 8. Data quality lessons

**KPI 2: a validation win.** The first Excel version used `COUNTIFS` on a merged sheet and reported **42,581** orders. When the SQL database was built, my KPI 2 query (`COUNT(DISTINCT order_id)` over reviews joined to payments) returned **43,981**. I then wrote the same measure in Power BI DAX with `CROSSFILTER ... BOTH` and it also returned 43,981. A direct SQL join against the live database confirmed the figure, and the 1,400-order gap (about 3%) was traced to the original Excel lookup logic. **43,981** became the official number and Tableau later reproduced it. Two independent methods agreeing against one outlier is exactly what cross-tool validation is for.

Other problems found and fixed along the way:

- **Hardcoded KPI 5 values.** The first Excel submission had typed-in numbers and an unverified correlation. It was rebuilt with a real PivotTable, Data Model calculated columns and `CORREL`, then reproduced independently.
- **Inflated Power BI chart.** KPI 5 first showed values of 142K to 713K instead of 10 to 21 days, caused by the default SUM aggregation on a raw column. Fixed with an average-based measure at order level.
- **Row-count miscounts.** `wc -l` over-counted the review file (multi-line quoted text) and under-counted a file with no trailing newline. Real counts came from the database after loading.
- **Wrong text in an insight panel.** A stale figure (12.09 days) and unfilled template placeholders were replaced with the verified numbers before sign-off.
- **Ambiguous filter path.** Two accidental relationships created a hidden second path between customers and sellers; they were removed.

## 9. Key business insights

| KPI | Insight | Suggested action |
| --- | --- | --- |
| 1 | Weekdays carry 77.02% of orders; weekend volume is about 3.4x lower, but average payment barely changes (154.44 vs 152.95) | Weight ad spend and logistics towards weekdays; test weekend flash sales |
| 2 | Credit card is the dominant payment type among 5-star orders. In the Power BI payment-type breakdown it is 75.39%; on distinct orders it is 43,981 of 57,075 five-star orders with a payment record (77.06%). About 44% of all orders are both 5-star and credit card | Keep credit card checkout fast and reliable |
| 3 | pet_shop orders take about 11 days (11.31), slightly faster than the 12.50-day average for all orders | Monitor pet_shop delivery times and work with sellers and logistics partners on predictable timelines |
| 4 | Sao Paulo is 15,540 of 99,441 orders (about 15.6%). In Power BI its per-order averages (112.58 price, 138.74 payment) are below those of all other cities (128.39 and 161.50) | Prioritise Sao Paulo in regional logistics and marketing |
| 5 | Longer shipping goes with lower reviews: 21.32 days for 1-star vs 10.68 days for 5-star, correlation -0.334 | Set a delivery target (the notes use 14 days) and monitor the logistics partners |

## 10. Dashboard screenshots

**KPI evidence**

| Excel | Power BI | MySQL |
| --- | --- | --- |
| ![Excel KPI 1](images/excel_kpi1_weekday_weekend.png) | ![Power BI KPI 2](images/powerbi_kpi2_5star_credit_card.png) | ![SQL KPI 2](images/sql_kpi2_output.png) |
| ![Excel KPI 3](images/excel_kpi3_pet_shop.png) | ![Power BI KPI 3](images/powerbi_kpi3_pet_shop.png) | ![SQL KPI 5](images/sql_kpi5_output.png) |
| ![Excel KPI 5](images/excel_kpi5_shipping_vs_review.png) | ![Power BI KPI 5](images/powerbi_kpi5_shipping_vs_review.png) | ![SQL KPI 4](images/sql_kpi4_output.png) |

**Tableau dashboard**

![Tableau dashboard](images/tableau_dashboard.png)

**Excel dashboard:** [TODO: add `images/excel_dashboard.png` after the "count of orders by payment method" chart is fixed (see Open items).]

**Power BI dashboard:** [TODO: add `images/powerbi_dashboard.png` after the On-Time Rate card and the payment-type donut are fixed (see Open items).]

Presentation: [`docs/Olist_Store_Analysis_Presentation.pptx`](docs/Olist_Store_Analysis_Presentation.pptx). Working notes: [`docs/project_notes.md`](docs/project_notes.md).

## 11. How to reproduce

**MySQL (8.0)**

1. Download the Kaggle dataset and unzip the 9 CSV files (see [`data/README.md`](data/README.md)).
2. Create the schema: `mysql -u <user> -p < sql/01_schema.sql`
3. Open `sql/02_load_data.sql`, replace `/path/to/olist_csv/` with your folder, enable `local_infile` (the file header explains how), then run it: `mysql --local-infile=1 -u <user> -p < sql/02_load_data.sql`
4. Run `sql/03_kpi_queries.sql` and compare with the results in section 7.

**Power BI**

1. Open `powerbi/Olist_Store_Analysis_Project__POWER_BI_.pbix` in Power BI Desktop (large file: see [`powerbi/README.md`](powerbi/README.md)). [TODO: add the file or a `.pbit` template.]
2. Home, Transform data, Data source settings, Change Source: point each CSV to your copy.
3. Refresh.

**Tableau**

1. Open `tableau/Olist_Store_Analysis_Project__TABLEAU_.twbx` in Tableau Public / Desktop, or the published version: [TODO: Tableau Public link].
2. The workbook includes its data extract. To rebuild, connect the 9 tables with Relationships as listed in [`tableau/README.md`](tableau/README.md).

**Excel:** the workbook is not published. Re-create the cleaning with [`excel/power_query_cleaning.m`](excel/power_query_cleaning.m) and the notes in [`excel/README.md`](excel/README.md).

## 12. Skills demonstrated

- **SQL (MySQL 8):** schema design, primary and foreign keys, `LOAD DATA`, joins, CTEs, subqueries, `DATEDIFF`, distinct counting, query validation
- **Power BI:** Power Query, data modelling and relationships, DAX (`CALCULATE`, `CROSSFILTER`, `AVERAGEX`, `DISTINCTCOUNT`), KPI cards, combo / bar / donut / gauge visuals
- **Excel:** Power Query, Power Pivot Data Model, PivotTables, calculated columns, `COUNTIFS`, `CORREL`
- **Tableau:** relationship-based data model, calculated fields, LOD expression (`FIXED`), dashboards
- **Data quality:** handling one-to-many joins, nulls with meaning, duplicate and parsing issues; reconciling the same KPI across four tools
- **Communication:** KPI-to-insight write-ups and a final presentation

## 13. My role and contributions

This was a team-based course project. I built the foundation in each module and the KPI 2 analysis; other KPI visuals and the final dashboards were built by teammates and are included here as part of the full analysis.

| Module | My work |
| --- | --- |
| Excel | Power Query cleaning of all 9 tables; Data Model with 9 relationships and 2 helper lookup tables; QA review of the Excel-stage KPI submissions |
| MySQL | Built the database (10 tables, 9 foreign keys), loaded and verified the data, cleaned the 610 blank categories, exported the database; **KPI 2 query** |
| Power BI | Data model (fresh CSV import, relationships, two zip-lookup tables); **KPI 2 DAX measures and page** |
| Tableau | Data model across all 9 tables with Relationships, plus the shared calculated fields |

Team work: KPI 1, 3, 4 and 5 queries and visuals, the Excel and Power BI dashboards, the Tableau worksheets and dashboard, and the presentations.

## 14. Dataset credit and licence note

Data: *Brazilian E-Commerce Public Dataset by Olist*, published on Kaggle by Olist. Check the licence on the [Kaggle dataset page](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) before reusing or redistributing the data. [TODO: confirm the licence terms and copy them here; I believe it is CC BY-NC-SA 4.0 but have not verified it.] The `.pbix` and `.twbx` files embed data, so the same terms apply to them.

Original code and documentation in this repository: [TODO: choose a licence, for example MIT.]

This is a course project and is not affiliated with Olist.

---

### Open items (delete this block before publishing)

1. **Excel geolocation cleaning.** Notes say 261,831 duplicate rows were removed; the saved query removes duplicates on `geolocation_city` and leaves 8,008 rows. Decide which is intended and fix the query or the wording (the presentation slide 6 also says "261,831 duplicate rows removed").
2. **Excel dashboard bug.** The "count of orders by payment method" chart totals 99,224 (the review row count), not payments. No KPI is affected. Fix or leave the dashboard screenshot out.
3. **Power BI dashboard.** The screenshot shows an On-Time Rate card reading 12K and a payment-type donut reading 71 for each type. Fix or leave the screenshot out.
4. **Power BI KPI 1 chart.** The line label on the weekend bar shows 156.54, while the card shows 152.95. Retake the screenshot.
5. **KPI 3 Excel value (11.23)** could not be reproduced; see section 7.
6. **Currency symbol:** rupee sign in project files, reais in the data.
7. **Review rows:** 99,223 in MySQL vs 99,224 in the CSV.
8. **File sizes and licence:** the workbook (about 132 MB) cannot be pushed to GitHub; the `.pbix` (about 61 MB) needs Git LFS or a Release; confirm the Kaggle licence.
