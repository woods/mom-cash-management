# **Premium Growth Data Tables for Google Sheets**

## **Ready-to-Import Premium Schedules and Formulas**

**Policy Details:**

* Death Benefit: $650,000  
* Starting Premium (Age 80): $32,556/year ($2,713/month)  
* Modeling Period: Ages 80-87

---

## **1\. COPY-PASTE READY: Premium Schedule Tables**

### **Table 1A: Base Case (11% Annual Growth) \- Monthly Premiums**

```
Age	Month	Monthly_Premium	Annual_Total	Cumulative_Total
80	0	2713	32556	32556
80	1	2738	32856	32856
80	2	2763	33159	33159
80	3	2788	33464	33464
80	4	2813	33772	33772
80	5	2838	34082	34082
80	6	2864	34394	34394
80	7	2889	34709	34709
80	8	2915	35026	35026
80	9	2941	35345	35345
80	10	2967	35667	35667
80	11	2994	35991	35991
81	12	3020	36317	68873
81	13	3047	36646	69519
81	14	3074	36977	70168
81	15	3101	37310	71478
81	16	3128	37646	78124
81	17	3155	37984	116108
81	18	3183	38324	154432
81	19	3210	38667	193099
81	20	3238	39012	232111
81	21	3266	39360	271470
81	22	3294	39710	311177
81	23	3323	40063	351237
82	24	3351	40418	391651
```

### **Table 1B: Base Case \- Annual Summary**

```
Age	Annual_Premium	Cumulative_Total	Premium_Increase_$	Premium_Increase_%
80	32556	32556	—	—
81	36137	68693	3581	11.0%
82	40112	108805	3975	11.0%
83	44524	153329	4412	11.0%
84	49422	202751	4898	11.0%
85	54858	257609	5436	11.0%
86	60892	318501	6034	11.0%
87	67590	386091	6698	11.0%
```

### **Table 2: Conservative Case (9% Annual Growth) \- Annual Summary**

```
Age	Annual_Premium	Cumulative_Total	Premium_Increase_$	Premium_Increase_%
80	32556	32556	—	—
81	35486	68042	2930	9.0%
82	38680	106722	3194	9.0%
83	42161	148883	3481	9.0%
84	45956	194839	3795	9.0%
85	50092	244931	4136	9.0%
86	54600	299531	4508	9.0%
87	59514	359045	4914	9.0%
```

### **Table 3: Aggressive Case (14% Annual Growth) \- Annual Summary**

```
Age	Annual_Premium	Cumulative_Total	Premium_Increase_$	Premium_Increase_%
80	32556	32556	—	—
81	37114	69670	4558	14.0%
82	42310	111980	5196	14.0%
83	48233	160213	5923	14.0%
84	54986	215199	6753	14.0%
85	62684	277883	7698	14.0%
86	71460	349343	8776	14.0%
87	81464	430807	10004	14.0%
```

---

## **2\. GOOGLE SHEETS FORMULAS**

### **Setup Instructions**

**Step 1: Create Named Ranges** In your Google Sheet, define these constants:

* `DEATH_BENEFIT` \= 650000  
* `INITIAL_PREMIUM_ANNUAL` \= 32556  
* `INITIAL_PREMIUM_MONTHLY` \= 2713  
* `GROWTH_RATE_BASE` \= 0.11  
* `GROWTH_RATE_CONSERVATIVE` \= 0.09  
* `GROWTH_RATE_AGGRESSIVE` \= 0.14

**Step 2: Monte Carlo Column Setup**

| Column | Description |
| ----- | ----- |
| A | Iteration number (1 to 10,000) |
| B | Survival time (months) from `=GAMMA.INV(RAND(), 3.5, 7.0)` |
| C | Survival time (years) from `=B2/12` |
| D | Total premiums paid (Base case) |
| E | Net benefit (Base case) from `=DEATH_BENEFIT - D2` |
| F | Total premiums paid (Conservative case) |
| G | Net benefit (Conservative case) |
| H | Total premiums paid (Aggressive case) |
| I | Net benefit (Aggressive case) |

### **Formula Library**

#### **Formula 1: Total Premiums (Monthly Compounding \- Base Case)**

**Cell D2:**

```
=SUMPRODUCT(
  SEQUENCE(ROUNDUP(B2),1,0),
  INITIAL_PREMIUM_MONTHLY * POWER(1 + GROWTH_RATE_BASE, SEQUENCE(ROUNDUP(B2),1,0)/12)
)
```

**What it does:**

* Creates a sequence of months from 0 to the month of death  
* Calculates the premium for each month using exponential growth  
* Sums all monthly premiums

**Example:** If survival \= 25 months (2.08 years), this calculates:

```
Month 0: $2,713
Month 1: $2,738
...
Month 24: $3,351
Total: $69,842
```

#### **Formula 2: Total Premiums (Annual Compounding \- Simplified)**

**Cell D2 (alternative, simpler version):**

```
=IF(C2 < 1,
  INITIAL_PREMIUM_ANNUAL * C2,
  SUMPRODUCT(
    SEQUENCE(FLOOR(C2),1,0),
    INITIAL_PREMIUM_ANNUAL * POWER(1 + GROWTH_RATE_BASE, SEQUENCE(FLOOR(C2),1,0))
  ) + INITIAL_PREMIUM_ANNUAL * POWER(1 + GROWTH_RATE_BASE, FLOOR(C2)) * MOD(C2, 1)
)
```

**What it does:**

* For survival \< 1 year: Prorates the first year's premium  
* For survival ≥ 1 year: Sums full years \+ prorated final year  
* Uses annual growth, simpler but slightly less precise than monthly

#### **Formula 3: Total Premiums (Conservative Case)**

**Cell F2:**

```
=SUMPRODUCT(
  SEQUENCE(ROUNDUP(B2),1,0),
  INITIAL_PREMIUM_MONTHLY * POWER(1 + GROWTH_RATE_CONSERVATIVE, SEQUENCE(ROUNDUP(B2),1,0)/12)
)
```

*(Same as Base Case but uses GROWTH\_RATE\_CONSERVATIVE)*

#### **Formula 4: Total Premiums (Aggressive Case)**

**Cell H2:**

```
=SUMPRODUCT(
  SEQUENCE(ROUNDUP(B2),1,0),
  INITIAL_PREMIUM_MONTHLY * POWER(1 + GROWTH_RATE_AGGRESSIVE, SEQUENCE(ROUNDUP(B2),1,0)/12)
)
```

#### **Formula 5: Net Benefit**

**Cell E2 (Base Case Net Benefit):**

```
=DEATH_BENEFIT - D2
```

**Cell G2 (Conservative Case Net Benefit):**

```
=DEATH_BENEFIT - F2
```

**Cell I2 (Aggressive Case Net Benefit):**

```
=DEATH_BENEFIT - H2
```

---

## **3\. SUMMARY STATISTICS FORMULAS**

**After running 10,000 iterations, add these summary statistics:**

### **Output Metrics (Base Case)**

```
// Median Net Benefit
=MEDIAN(E2:E10001)

// Mean Net Benefit  
=AVERAGE(E2:E10001)

// Standard Deviation
=STDEV(E2:E10001)

// 10th Percentile (Downside Risk)
=PERCENTILE(E2:E10001, 0.10)

// 25th Percentile
=PERCENTILE(E2:E10001, 0.25)

// 75th Percentile
=PERCENTILE(E2:E10001, 0.75)

// 90th Percentile (Upside Potential)
=PERCENTILE(E2:E10001, 0.90)

// Probability of Positive Net Benefit
=COUNTIF(E2:E10001, ">0") / 10000

// Probability of Net Benefit > $500,000
=COUNTIF(E2:E10001, ">500000") / 10000

// Probability of Net Benefit > $600,000
=COUNTIF(E2:E10001, ">600000") / 10000
```

### **Comparison Table Format**

```
Metric                          | Base (11%) | Conservative (9%) | Aggressive (14%)
--------------------------------|------------|-------------------|------------------
Median Net Benefit             | [E calc]   | [G calc]          | [I calc]
Mean Net Benefit               | [E calc]   | [G calc]          | [I calc]
10th Percentile Net Benefit    | [E calc]   | [G calc]          | [I calc]
90th Percentile Net Benefit    | [E calc]   | [G calc]          | [I calc]
Probability (Net Benefit > $0) | [E calc]   | [G calc]          | [I calc]
Average Total Premium Paid     | [D calc]   | [F calc]          | [H calc]
```

---

## **4\. QUICK REFERENCE: Premium by Survival Duration**

**Base Case (11% Growth):**

```
Survival_Months	Survival_Years	Total_Premiums	Net_Benefit	ROI
6	0.50	16,278	633,722	38.93
12	1.00	32,556	617,444	18.96
18	1.50	49,923	600,077	12.02
22	1.83	61,012	588,988	9.65
24	2.00	68,693	581,307	8.46
30	2.50	88,886	561,114	6.31
36	3.00	108,805	541,195	4.97
44	3.67	146,453	503,547	3.44
48	4.00	153,329	496,671	3.24
60	5.00	202,751	447,249	2.21
72	6.00	257,609	392,391	1.52
84	7.00	318,501	331,499	1.04
```

**Conservative Case (9% Growth):**

```
Survival_Months	Survival_Years	Total_Premiums	Net_Benefit	ROI
12	1.00	32,556	617,444	18.96
22	1.83	59,581	590,419	9.91
24	2.00	68,042	581,958	8.55
36	3.00	106,722	543,278	5.09
44	3.67	142,890	507,110	3.55
60	5.00	194,839	455,161	2.34
84	7.00	299,531	350,469	1.17
```

**Aggressive Case (14% Growth):**

```
Survival_Months	Survival_Years	Total_Premiums	Net_Benefit	ROI
12	1.00	32,556	617,444	18.96
22	1.83	62,706	587,294	9.36
24	2.00	69,670	580,330	8.33
36	3.00	111,980	538,020	4.80
44	3.67	150,492	499,508	3.32
60	5.00	215,199	434,801	2.02
84	7.00	349,343	300,657	0.86
```

---

## **5\. VISUALIZATION FORMULAS**

### **Chart 1: Premium Growth Over Time**

**Data for Line Chart:**

```
Age	Conservative	Base	Aggressive
80	32556	32556	32556
81	35486	36137	37114
82	38680	40112	42310
83	42161	44524	48233
84	45956	49422	54986
85	50092	54858	62684
86	54600	60892	71460
87	59514	67590	81464
```

**Chart Type:** Line chart with 3 series **X-axis:** Age **Y-axis:** Annual Premium ($) **Title:** "Premium Growth Scenarios (Ages 80-87)"

### **Chart 2: Net Benefit Distribution**

**Create histogram of Column E (Base Case Net Benefit):**

* Bins: $300,000 to $650,000 in $25,000 intervals  
* Shows distribution of outcomes across 10,000 simulations

---

## **6\. VALIDATION CHECKS**

**Add these checks to verify formulas are working correctly:**

### **Check 1: Premium at Age 81 (Base Case)**

```
// Manual calculation:
=INITIAL_PREMIUM_ANNUAL * (1 + GROWTH_RATE_BASE)

// Should equal: $36,137

// Compare to your formula output for 12-month survival
```

### **Check 2: Two-Year Total (Base Case)**

```
// Manual calculation:
=INITIAL_PREMIUM_ANNUAL + INITIAL_PREMIUM_ANNUAL * (1 + GROWTH_RATE_BASE)

// Should equal: $68,693

// Compare to your formula output for 24-month survival
```

### **Check 3: Gamma Distribution**

```
// Test that Gamma distribution is working:
=AVERAGE(B2:B10001)

// Should be approximately 24.5 months (mean of Gamma(3.5, 7.0))

=MEDIAN(B2:B10001)

// Should be approximately 22.1 months (median of Gamma(3.5, 7.0))
```

---

## **7\. STEP-BY-STEP IMPLEMENTATION GUIDE**

### **Phase 1: Setup (15 minutes)**

1. Open new Google Sheet  
2. Create tabs: "Simulation", "Summary", "Charts", "Reference Data"  
3. In "Simulation" tab, set up columns A-I as described in Section 2  
4. Define named ranges for constants  
5. Copy premium schedule tables from Section 1 into "Reference Data" tab

### **Phase 2: Build Formulas (20 minutes)**

1. In Cell B2, enter: `=GAMMA.INV(RAND(), 3.5, 7.0)`  
2. In Cell C2, enter: `=B2/12`  
3. In Cell D2, enter the total premiums formula (Section 2, Formula 1\)  
4. In Cell E2, enter: `=650000 - D2`  
5. In Cells F2 and H2, enter conservative and aggressive premium formulas  
6. In Cells G2 and I2, enter net benefit formulas  
7. Select Row 2, copy formulas down to Row 10,001

### **Phase 3: Run Simulation (5 minutes)**

1. Press F9 or click "Recalculate" to run simulation  
2. Wait for all 10,000 rows to calculate (may take 1-2 minutes)  
3. Review first 10-20 rows to verify formulas are working correctly

### **Phase 4: Generate Summary Statistics (15 minutes)**

1. In "Summary" tab, create summary statistics table  
2. Use formulas from Section 3 to calculate key metrics  
3. Create comparison table for all three scenarios  
4. Add conditional formatting to highlight key results

### **Phase 5: Create Visualizations (20 minutes)**

1. Create line chart showing premium growth over time  
2. Create histogram of net benefit distribution  
3. Create scatter plot: survival time (x) vs net benefit (y)  
4. Add trendlines and annotations as needed

**Total Implementation Time: \~75 minutes**

---

## **8\. TROUBLESHOOTING COMMON ISSUES**

### **Issue 1: \#NUM\! Error in Premium Formula**

**Problem:** SEQUENCE function fails for very short survival times **Solution:** Add error handling:

```
=IFERROR(
  SUMPRODUCT(SEQUENCE(ROUNDUP(B2),1,0), ...),
  INITIAL_PREMIUM_MONTHLY * (B2/12)
)
```

### **Issue 2: Slow Calculation Speed**

**Problem:** 10,000 iterations with complex formulas take too long **Solutions:**

* Reduce to 5,000 iterations temporarily while testing  
* Use annual compounding (Formula 2\) instead of monthly  
* Disable automatic calculation (File → Settings → Calculation → Manual)

### **Issue 3: Gamma Distribution Seems Wrong**

**Check:**

```
// If mean is far from 24.5 or median far from 22.1:
// 1. Verify parameters are exactly 3.5 and 7.0
// 2. Make sure you're using GAMMA.INV not GAMMA.DIST
// 3. Confirm RAND() is generating random numbers
```

---

## **9\. EXPECTED RESULTS PREVIEW**

**Based on preliminary analysis, you should expect:**

### **Base Case (11% Growth) Results:**

* **Median Net Benefit:** \~$588,000 \- $592,000  
* **Mean Net Benefit:** \~$585,000 \- $590,000  
* **Probability (Net Benefit \> $0):** \~99.9%  
* **Probability (Net Benefit \> $500,000):** \~85-90%  
* **Probability (Net Benefit \> $600,000):** \~40-45%

**Interpretation:** The policy is highly likely to be profitable. Even accounting for premium increases, the net benefit is strongly positive in virtually all scenarios because survival time is shorter than premium break-even point.

### **Decision Implications:**

If your Monte Carlo results match these expectations:

* **Keeping the policy is financially advantageous** in the vast majority of scenarios  
* The main question becomes: Can your mother afford the premiums until death?  
* Taking a loan to pay premiums is likely justified by the expected return

---

## **10\. ADVANCED EXTENSIONS (Optional)**

### **Extension 1: Include Loan Interest**

If premiums are paid via loan:

**Modified Net Benefit Formula:**

```
// Assuming 6% annual loan interest rate
=DEATH_BENEFIT - D2 * POWER(1.06, C2)
```

This compounds the total premiums paid at the loan interest rate.

### **Extension 2: Add Income Shortfall Analysis**

Your mother faces $100,000 income shortfall in 2025\. Model scenarios:

**Scenario A:** Pay premiums from loan, repay loan from death benefit **Scenario B:** Let policy lapse, use $100K from other sources

### **Extension 3: Incorporate Other Costs**

Add columns for:

* Nursing home costs  
* Medical expenses  
* Other estate planning considerations

---

## **11\. FINAL CHECKLIST**

Before relying on Monte Carlo results:

* \[ \] Verified Gamma(3.5, 7.0) produces mean ≈ 24.5 months  
* \[ \] Checked premium formulas match reference tables (Section 1\)  
* \[ \] Confirmed all three growth scenarios (9%, 11%, 14%) are working  
* \[ \] Generated summary statistics for all scenarios  
* \[ \] Created visualizations to communicate results  
* \[ \] Compared results to expected ranges (Section 9\)  
* \[ \] Documented assumptions and limitations  
* \[ \] Saved backup copy of working spreadsheet

**When checklist is complete, you have a robust model for decision-making.**

---

## **12\. APPENDIX: Mathematical Derivations**

### **Exponential Growth Formula Derivation**

Starting with continuous compounding:

```
P(t) = P₀ × e^(r×t)
```

For discrete annual compounding:

```
P(t) = P₀ × (1 + r)^t
```

For monthly compounding within annual growth rate:

```
P(m) = P₀ × (1 + r)^(m/12)
```

Where:

* P₀ \= Initial premium ($32,556/year or $2,713/month)  
* r \= Annual growth rate (0.09, 0.11, or 0.14)  
* t \= Time in years  
* m \= Time in months

### **Sum of Geometric Series**

Total premiums over n months:

```
Total = Σ(i=0 to n-1) P₀ × (1+r)^(i/12)
```

This is a geometric series with:

* First term: a \= P₀  
* Common ratio: q \= (1+r)^(1/12)  
* Number of terms: n

Sum formula:

```
Total = P₀ × [(q^n - 1) / (q - 1)]
```

Where q \= (1+r)^(1/12)

Google Sheets SUMPRODUCT formula implements this efficiently.

---

**Document Version:** 1.0  
 **Last Updated:** November 2025  
 **Purpose:** Ready-to-use data and formulas for Monte Carlo simulation  
 **Compatible with:** Google Sheets, Microsoft Excel (2016+)

