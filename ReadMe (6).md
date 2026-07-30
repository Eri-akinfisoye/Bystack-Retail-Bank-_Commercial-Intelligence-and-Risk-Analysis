# Financial Insights System — Retail Bank (Bystack)
### SQL-Driven Customer & Transaction Intelligence | Microsoft SQL Server

---

## Table of Contents
- [Business Problem](#business-problem)
- [Project Objective](#project-objective)
- [Data Structure](#data-structure)
- [ER Diagram](#er-diagram)
- [Executive Summary](#executive-summary)
- [Key Findings](#key-findings)
  - [Finding 1: High-Value Customer Concentration](#finding-1-high-value-customer-concentration)
  - [Finding 2: Dormant Account Crisis](#finding-2-dormant-account-crisis)
  - [Finding 3: Investment Engagement Gap](#finding-3-investment-engagement-gap)
  - [Finding 4: Transaction Volume Trends](#finding-4-transaction-volume-trends)
- [Strategic Recommendations](#strategic-recommendations)
- [Analytical Decisions](#analytical-decisions)
- [Tools & Technology](#tools--technology)
- [Author](#author)

---

## Business Problem

Bystack is a retail banking institution offering both transactional and investment services. With digital banking adoption increasing and competition rising, understanding customer financial behaviour has become critical for retention, revenue growth, and product strategy.

Despite holding data spanning over two decades of customer activity, Bystack lacked a structured analytical view of:
- Which customers drive the most value — and how exposed the business is to losing them
- How many registered customers are actually engaged with the platform
- Whether investment products are being adopted broadly or concentrated in a small segment
- How transaction volumes have shifted over time and what that signals operationally

This project addresses all four questions using pure SQL analysis on Bystack's core banking database.

---

## Project Objective

Deliver a SQL-driven intelligence report that enables Bystack's Marketing, Customer Insight, and Finance teams to make data-backed decisions on customer segmentation, retention planning, and product optimisation.

Specific goals:
- Identify the top 10% of customers by total financial value
- Summarise account activity across deposits, withdrawals, and balances
- Map investment product engagement across the customer base
- Analyse transaction volume trends from 2011–2023

---

## Data Structure

The database consists of **nine interconnected tables** reflecting core banking operations. Four tables were used in this analysis; five were out of scope for the defined business questions.

| Table | Used | Description |
|---|---|---|
| `FB.Customers` | ✅ | Customer demographic and profile details |
| `FB.Accounts` | ✅ | Account type, balance, and ownership metadata |
| `FB.Transactions` | ✅ | Deposits, withdrawals, payments, and transfers |
| `FB.Investments` | ✅ | Investment type and capital amount per customer |
| `FB.Branches` | ❌ | Branch identification and location data |
| `FB.Employees` | ❌ | Employee and staffing records |
| `FB.CreditCards` | ❌ | Credit card usage and limits |
| `FB.Loans` | ❌ | Loan balances and interest rates |
| `FB.Payments` | ❌ | Card-based and bill payment processing |

> Tables marked ❌ were excluded because they fell outside the defined analytical scope for this project.

---

## ER Diagram

> *[Insert ER Diagram screenshot here]*

The four active tables are linked through `CustomerID` and `AccountID` as the primary join keys. A customer can hold multiple accounts; each account can have multiple transactions. Investment records are linked directly at the customer level, not the account level.

---

## Executive Summary

Analysis of Bystack's customer, account, transaction, and investment data (2011–2023) across 1,000 registered customers reveals a business with strong value concentration in a small customer segment, a large inactive base representing untapped opportunity, and transaction patterns that suggest both peak periods and concerning decline windows.

**Four headline findings drive the strategic direction:**

| # | Finding | Implication |
|---|---|---|
| 1 | Top 63 customers hold **38% of total value** | Extreme revenue concentration risk |
| 2 | **39% of registered users** have zero balance and no transactions | Massive onboarding dropout problem |
| 3 | **38% of customers** hold no investment products | Significant cross-sell opportunity |
| 4 | Transaction volume peaked in **2020**, with notable drops in 2015 and 2018–2019 | Operational and market signals worth investigating |

The business is simultaneously over-reliant on a small sophisticated investor segment and under-serving a large inactive cohort. Both represent immediate, addressable opportunities.

---

## Key Findings

### Finding 1: High-Value Customer Concentration

**Objective:** Identify the top 10% of customers by combined account balance and investment value.

```sql
-- Step 1: Calculate total value per customer (balance + investments)
WITH CustomerTotalValue AS (
    SELECT
        A.CustomerID,
        CONCAT(C.FirstName, ' ', C.LastName) AS FullName,
        ISNULL(SUM(A.Balance), 0) AS TotalBalance,
        ISNULL(SUM(I.Amount), 0) AS TotalInvestment,
        (ISNULL(SUM(A.Balance), 0) + ISNULL(SUM(I.Amount), 0)) AS TotalValue
    FROM FB.Accounts A
    LEFT JOIN FB.Investments I ON A.CustomerID = I.CustomerID
    LEFT JOIN FB.Customers C ON A.CustomerID = C.CustomerID
    GROUP BY A.CustomerID, CONCAT(C.FirstName, ' ', C.LastName)
),
-- Step 2: Rank customers into 10 equal bands by total value
RankedValue AS (
    SELECT
        CustomerID,
        FullName,
        TotalValue,
        NTILE(10) OVER (ORDER BY TotalValue DESC) AS ValueBand
    FROM CustomerTotalValue
)
-- Step 3: Return top band only
SELECT * FROM RankedValue WHERE ValueBand = 1;
```

> *[Insert query result screenshot here]*

**Findings:**
- Only **63 of 1,000 customers (6.3%)** qualify as top 10% by total value — yet they collectively hold **38% of Bystack's total customer value**
- The average total value of this segment is **nearly 4× the general customer base**, confirming strong financial concentration
- **84% of high-value customers** hold more than one investment account; **78% diversify across multiple investment types** — indicating high financial sophistication and long-term planning behaviour
- This segment is ideal for premium advisory services and relationship manager assignment — and represents the highest retention risk if left unmanaged

---

### Finding 2: Dormant Account Crisis

**Objective:** Summarise customer account activity including deposits, withdrawals, and current balance.

```sql
SELECT
    C.CustomerID,
    ISNULL(SUM(A.Balance), 0) AS Balance,
    SUM(CASE WHEN T.TransactionType = 'Deposit' THEN T.Amount ELSE 0 END) AS TotalDeposit,
    SUM(CASE WHEN T.TransactionType = 'Withdrawal' THEN T.Amount ELSE 0 END) AS TotalWithdrawal
FROM FB.Customers C
LEFT JOIN FB.Accounts A ON C.CustomerID = A.CustomerID
LEFT JOIN FB.Transactions T ON A.AccountID = T.AccountID
GROUP BY C.CustomerID;
```

> *[Insert query result screenshot here]*

**Findings:**
- **39% of registered customers** hold a zero account balance with no recorded deposits or withdrawals
- This pattern points to three likely causes: abandoned sign-ups, onboarding friction that prevents first-time activity, or accounts that became dormant after initial registration
- While this cohort contributes nothing to current financial activity, it represents a **high-potential conversion segment** — the infrastructure cost of reactivating an existing account is significantly lower than acquiring a new customer
- This finding has direct implications for marketing spend efficiency: targeting dormant accounts with activation campaigns may yield better ROI than new acquisition

---

### Finding 3: Investment Engagement Gap

**Objective:** Retrieve total investment value and product types per customer.

```sql
-- Step 1: Aggregate total investment per customer
WITH TotalInvestments AS (
    SELECT
        C.CustomerID,
        ISNULL(SUM(I.Amount), 0) AS TotalInvestmentAmount
    FROM FB.Customers C
    LEFT JOIN FB.Investments I ON C.CustomerID = I.CustomerID
    GROUP BY C.CustomerID
),
-- Step 2: Get distinct investment types per customer
DistinctTypes AS (
    SELECT DISTINCT
        C.CustomerID,
        I.InvestmentType
    FROM FB.Customers C
    INNER JOIN FB.Investments I ON C.CustomerID = I.CustomerID
)
-- Step 3: Combine total amount with aggregated investment type list
SELECT
    T.CustomerID,
    ROUND(T.TotalInvestmentAmount, 2) AS TotalInvestmentAmount,
    STRING_AGG(D.InvestmentType, ', ') WITHIN GROUP (ORDER BY D.InvestmentType ASC) AS InvestmentTypes
FROM TotalInvestments T
LEFT JOIN DistinctTypes D ON T.CustomerID = D.CustomerID
GROUP BY T.CustomerID, T.TotalInvestmentAmount
ORDER BY T.TotalInvestmentAmount DESC;
```

> *[Insert query result screenshot here]*

**Findings:**
- **38% of customers have not engaged with any investment product**, suggesting awareness gaps, perceived complexity, or low confidence in Bystack's investment offerings
- Among active investors, **Stocks and ETFs account for 49% of investment activity** — reflecting a strong preference for familiar, accessible instruments
- Only **24% of investors participate in multiple investment types**, meaning the majority are concentrated in a single product — an indicator of both product education opportunity and portfolio risk for those customers
- Cross-selling from single-product to multi-product investors is the clearest near-term revenue lever available within the existing customer base

---

### Finding 4: Transaction Volume Trends (2011–2023)

**Objective:** Calculate transaction volume by month and transaction type across a 13-year period.

```sql
SELECT
    YEAR(TransactionDate) AS Year,
    MONTH(TransactionDate) AS Month,
    TransactionType,
    ROUND(ISNULL(SUM(Amount), 0), 2) AS TotalTransactionVolume
FROM FB.Transactions
WHERE TransactionDate >= '2011-01-01'
  AND TransactionDate < '2024-01-01'
GROUP BY YEAR(TransactionDate), MONTH(TransactionDate), TransactionType;
```

> *[Insert query result screenshot here]*

**Findings:**
- Transaction types are **remarkably balanced**: Payments (27.45%), Transfers (25.93%), Deposits (23.88%), Withdrawals (22.74%) — indicating diverse and stable platform usage rather than dependency on a single transaction category
- **Yearly trend shows fluctuation rather than linear growth:**
  - 2011–2013: Stable baseline activity
  - 2015: Notable volume drop (~12.6% decline) — cause unidentified; warrants investigation
  - 2016–2020: Recovery and peak — 2020 marks the highest transaction activity on record
  - 2021–2023: Stabilisation phase with minimal growth movement
- The 2020 peak likely reflects digital banking acceleration during the COVID-19 period — a structural shift rather than a temporary spike
- The 2015 and 2018–2019 decline periods are the most operationally significant: understanding the root causes is essential before attributing them to market conditions vs. internal service failures

---

## Strategic Recommendations

### 1. High-Value Customer Retention Program
- Assign dedicated relationship managers to the top 63 customers
- Introduce tiered premium membership (Gold / Platinum / Signature) with differentiated service channels
- Offer personalised investment advisory for customers currently concentrated in a single product type
- Implement early warning monitoring for this segment — any behavioural signal of disengagement should trigger proactive outreach

### 2. Dormant Account Reactivation
- Deploy automated onboarding follow-up sequences targeting the 39% zero-balance cohort
- Introduce welcome incentives (e.g., deposit bonuses, fee waivers) to drive first-time transaction activity
- Conduct a root cause audit of the onboarding flow to identify specific drop-off points
- Segment the dormant cohort by registration date to distinguish recent abandonment from long-term inactivity — the intervention strategy differs for each

### 3. Investment Product Adoption
- Develop simplified financial education resources to lower the knowledge barrier for non-investors
- Promote alternative products (bonds, mutual funds) to customers currently concentrated in Stocks/ETFs
- Run limited-time fee reduction campaigns to incentivise first-time investors
- Use the multi-investment customers (24%) as case studies for peer-influenced marketing within the platform

### 4. Transaction Trend Investigation
- Commission a retrospective review of the 2015 decline and 2018–2019 dip — identify whether causes were internal (service issues, fee changes) or external (market conditions)
- Study the 2020 peak drivers to understand which services or campaigns accelerated growth, then assess replicability
- Introduce cashback or loyalty incentives for recurring payment transactions to protect volume in the highest-share category
- Build a monthly transaction monitoring dashboard (Power BI recommended) to move from retrospective to real-time trend detection

---

## Analytical Decisions

**Why only 4 of 9 tables were used:**
The remaining five tables (Branches, Employees, CreditCards, Loans, Payments) contained data relevant to operational and credit risk analysis — outside the defined scope of customer segmentation and transaction intelligence. Including them without a defined business question would have introduced noise rather than insight. They remain available for a Phase 2 credit risk or operational efficiency analysis.

**Why LEFT JOIN was used consistently for customer aggregations:**
Using INNER JOIN on accounts or investments would silently exclude customers with no financial activity — precisely the dormant segment that became Finding 2. LEFT JOIN ensures the full 1,000-customer base is preserved as the analytical denominator, making zero-activity customers visible rather than hidden.

**Why NTILE(10) was chosen for segmentation:**
NTILE produces equal-sized bands based on actual data distribution, making it more robust than arbitrary threshold cuts (e.g., "top $50,000") which would be sensitive to outliers and dataset-specific. It also scales cleanly if the customer base grows.

**Why this project was kept as pure SQL:**
This analysis was intentionally scoped as a SQL-only project to demonstrate query construction, relational data modelling, and business reasoning independently of visualisation tools. The findings are structured to feed directly into a Power BI dashboard as a natural next phase.

---

## Tools & Technology

| Tool | Purpose |
|---|---|
| Microsoft SQL Server (SSMS) | Query writing, execution, and result validation |
| Excel | Initial dataset structure review and data cleaning |
| ERDPlus / Draw.io | Entity-Relationship Diagram design |

---

## Author

**Akinfisoye Erioluwa**
Data & Business Analytics | SQL · Power BI · Excel · DAX

[GitHub](https://github.com/Eri-akinfisoye) · [LinkedIn](#) · [NexaLink Churn Analysis Project](https://github.com/Eri-akinfisoye/NexaLink-Churn-Analysis)

---

*This project is part of a growing portfolio of SQL and Power BI analytics work. See also: NexaLink Telecom Churn Analysis, Dangote Cement KPI Automation System (DCP4).*
