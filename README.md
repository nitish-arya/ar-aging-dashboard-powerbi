# AR Aging Dashboard (Power BI)

An Accounts Receivable (AR) analytics dashboard for a medical billing scenario. It shows how much money is still outstanding, how old it is, which payers hold it, and why claims get denied.

> **Note:** All data is synthetic (dummy) and created for practice. No real patient or company data is used. Aging is calculated as of **15-Oct-2026**.

## Business Problem

AR teams need to know where unpaid claims are stuck so they can focus follow-up where it matters most. This dashboard answers:

- How much is billed, collected and still outstanding?
- How old is the outstanding amount (aging buckets)?
- Which payers hold the most unpaid claims?
- What are the top denial reasons, and how much do they cost?
- How do billed and collected amounts trend by month?

## Tools Used

- **Excel**: raw dataset (`AR_practice_dataset.xlsx`)
- **Power Query (Power BI)**: data cleaning and new columns
- **DAX**: measures and KPIs
- **Power BI Desktop**: report design, slicer, interactions

## Dataset

810 raw rows, 9 columns: Claim ID, Patient ID (fake), Payer, Billed Amount, Paid Amount, Claim Date, Payment Date, Status (Paid / Denied / Pending), Denial Reason.

The data was intentionally messy to practise cleaning.

## Data Cleaning (Power Query)

| Issue in raw data | How it was handled |
|---|---|
| Duplicate claims (10 rows) | Removed duplicates on Claim ID (810 to 800 rows) |
| Extra spaces and mixed case in Payer and Status | Trim + Capitalize Each Word; fixed "UnitedHealthcare" spelling |
| Blank Payer | Replaced with "Unknown" |
| Claim Date stored as text in two formats | Converted with locale set to English (India) so day/month are not swapped |
| Negative Paid Amounts | Converted to absolute value (status was Paid, so sign was an entry error) |
| Blank Billed Amount | Rows removed (under 1% of data, cannot be reliably estimated). Final: 794 rows |
| Denied claims with no reason | Set to "Not Specified" |
| Blank Paid Amount on Pending claims | Replaced with 0 |

### Custom columns

```
AR Days = Duration.Days(
    Date.From(if [Payment Date] = null then #date(2026,10,15) else [Payment Date])
    - Date.From([Claim Date]))

Aging Bucket = if [AR Days] <= 30 then "0-30"
    else if [AR Days] <= 60 then "31-60"
    else if [AR Days] <= 90 then "61-90"
    else "90+"

Outstanding Amount = if [Status] = "Pending" then [Billed Amount] - [Paid Amount] else 0

Denied Amount = if [Status] = "Denied" then [Billed Amount] else 0
```

**Design decision:** only Pending claims are treated as outstanding. Denied claims are tracked separately because they need an appeal, and the gap between billed and paid on Paid claims is a contractual adjustment, not money still owed.

## DAX Measures

```
Total Billed = SUM(Claims_Raw[Billed Amount])
Total Collected = SUM(Claims_Raw[Paid Amount])
Total Outstanding = SUM(Claims_Raw[Outstanding Amount])
Total Claims = COUNTROWS(Claims_Raw)
Denied Claims = CALCULATE(COUNTROWS(Claims_Raw), Claims_Raw[Status] = "Denied")
Denial Rate = DIVIDE([Denied Claims], [Total Claims])
Collected Rate = DIVIDE([Total Collected], [Total Billed])
Outstanding 90+ % = DIVIDE(
    CALCULATE([Total Outstanding], Claims_Raw[Aging Bucket] = "90+"),
    [Total Outstanding])
```

## Dashboard

**Page 1: AR Overview**
- KPI cards: Total Billed, Total Collected, Total Outstanding, Outstanding 90+ %, Denied Claims, Collected Rate, Denied Rate
- Outstanding by Aging Bucket
- Outstanding by Payer
- Billed vs Collected by Month
- Denied Amount by Reason
- Payer slicer (the Payer chart is excluded from the slicer so it always shows all payers)

**Page 2: Key Insights**: written findings and recommended actions.

*(Add screenshots here: <img width="907" height="500" alt="overview" src="https://github.com/user-attachments/assets/a0dc1f16-3969-4c62-9858-6c8d95f651e1" />
<img width="905" height="509" alt="insights" src="https://github.com/user-attachments/assets/d28d7dc0-6c44-401c-85a5-f02676bdea11" />
)*

## Key Insights

1. **48.8% of outstanding (0.92M of 1.88M) is more than 90 days old** across 114 claims. This is the first priority for follow-up.
2. **Denial rate is 11.59%** (92 of 794 claims, worth 0.76M). Coding Error, Timely Filing Exceeded and Duplicate Claim make up about 56% of the denied amount.
3. **No single payer is the problem.** UnitedHealthcare has the most outstanding (0.32M, about 17%), and the top 3 payers hold about 47%.
4. **Recent months look weak because claims are still pending**, not because collections fell. October has only 15 days of data.

## Recommended Actions

- Work the 90+ day bucket first.
- Add pre-submission checks for coding errors and duplicate claims.
- Track filing deadlines so claims are not lost to Timely Filing.

## Files

- `AR_Dashboard.pbix`: Power BI report
- `AR_practice_dataset.xlsx`: synthetic raw data
- `images/`: dashboard screenshots

## Author

Nitish Arya
