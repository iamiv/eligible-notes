I'll design a comprehensive QuickSight solution for your multi-investor ABS/structured finance dashboard with industry-specific eligibility rules.

## **Data Model Enhancement**

### **Parameters Setup**

```javascript
// Core Parameters
1. selected_investor (Single Value)
   - Values: Investor A, Investor B, Investor C, All Investors
   - Default: Investor A

2. include_unallocated (Single Value - Yes/No)
   - Default: Yes

3. selected_sub_tranche (Single Value)
   - Values: All, A1, A2, A3, B1, B2, etc.
   - Default: All

4. scenario_mode (Single Value)
   - Values: Base, Scenario 1, Scenario 2, Scenario 3
   - Default: Base

5. deals_to_add (Multi-Value String)
   - Selected deal_ids to pledge from Unallocated

6. deals_to_remove (Multi-Value String)
   - Selected deal_ids to unpledge

7. view_mode (Single Value)
   - Values: Summary, Eligible Only, Ineligible Only, All Deals
   - Default: Summary
```

## **Critical Calculated Fields**

### **1. Deal-Level Filtering Logic**

```sql
-- CALC: is_in_scope
-- Determines if deal should be included in current view
ifelse(
  ${selected_investor} = 'All Investors', 1,
  OR(
    {investor_name} = ${selected_investor},
    AND(
      {investor_name} = 'Unallocated',
      ${include_unallocated} = 'Yes'
    )
  ), 1,
  0
)

-- CALC: is_sub_tranche_match
ifelse(
  ${selected_sub_tranche} = 'All', 1,
  {sub_tranche} = ${selected_sub_tranche}, 1,
  0
)

-- CALC: effective_investor
-- For scenario analysis - which investor owns this deal in current scenario
ifelse(
  contains(${deals_to_add}, {deal_id}), ${selected_investor},
  contains(${deals_to_remove}, {deal_id}), 'Unallocated',
  {investor_name}
)

-- CALC: is_pledged_in_scenario
ifelse(
  {effective_investor} = ${selected_investor}, 1,
  0
)

-- CALC: scenario_balance
-- Balance to use in concentration calculations
ifelse(
  AND(
    {is_in_scope} = 1,
    {is_sub_tranche_match} = 1,
    {is_pledged_in_scenario} = 1
  ),
  {current_principal},
  0
)
```

### **2. Industry-Specific Eligibility Rules**

```sql
-- CALC: naics3_code
-- Extract first 3 digits of NAICS
left({naics_code}, 3)

-- CALC: required_fico_by_industry_grade
-- Dynamic FICO requirement based on industry and credit grade
ifelse(
  AND({naics3_code} = '722', {credit_grade} IN ('A', 'A+', 'A-')), 680,
  AND({naics3_code} = '236', {credit_grade} IN ('A', 'A+', 'A-')), 700,
  AND({naics3_code} IN ('441', '442', '443', '444', '445'), {credit_grade} IN ('A', 'A+', 'A-')), 660,
  
  AND({naics3_code} = '722', {credit_grade} IN ('B', 'B+', 'B-')), 650,
  AND({naics3_code} = '236', {credit_grade} IN ('B', 'B+', 'B-')), 670,
  AND({naics3_code} IN ('441', '442', '443', '444', '445'), {credit_grade} IN ('B', 'B+', 'B-')), 630,
  
  AND({naics3_code} = '722', {credit_grade} IN ('C', 'C+', 'C-')), 620,
  AND({naics3_code} = '236', {credit_grade} IN ('C', 'C+', 'C-')), 640,
  AND({naics3_code} IN ('441', '442', '443', '444', '445'), {credit_grade} IN ('C', 'C+', 'C-')), 600,
  
  640  -- Default requirement
)

-- CALC: required_tib_by_industry_grade
-- Dynamic Time in Business requirement (months)
ifelse(
  AND({naics3_code} = '722', {credit_grade} IN ('A', 'A+', 'A-')), 24,
  AND({naics3_code} = '236', {credit_grade} IN ('A', 'A+', 'A-')), 36,
  AND({naics3_code} IN ('441', '442', '443', '444', '445'), {credit_grade} IN ('A', 'A+', 'A-')), 18,
  
  AND({naics3_code} = '722', {credit_grade} IN ('B', 'B+', 'B-')), 30,
  AND({naics3_code} = '236', {credit_grade} IN ('B', 'B+', 'B-')), 42,
  AND({naics3_code} IN ('441', '442', '443', '444', '445'), {credit_grade} IN ('B', 'B+', 'B-')), 24,
  
  24  -- Default requirement
)

-- CALC: passes_fico_test
ifelse(
  {fico_score} >= {required_fico_by_industry_grade}, 1, 0
)

-- CALC: passes_tib_test
ifelse(
  {time_in_business_months} >= {required_tib_by_industry_grade}, 1, 0
)

-- CALC: passes_basic_eligibility
-- Combine all basic tests
ifelse(
  AND(
    {passes_fico_test} = 1,
    {passes_tib_test} = 1,
    {is_current} = 1,  -- No delinquencies
    {current_principal} > 0,
    {maturity_date} > now()
  ), 1, 0
)

-- CALC: eligibility_failures
-- List of failed tests for drill-down
concat(
  ifelse({passes_fico_test} = 0, 'FICO Below Threshold; ', ''),
  ifelse({passes_tib_test} = 0, 'TIB Below Threshold; ', ''),
  ifelse({is_current} = 0, 'Not Current; ', ''),
  ifelse({current_principal} <= 0, 'Zero Balance; ', ''),
  ifelse({maturity_date} <= now(), 'Matured; ', '')
)

-- CALC: is_eligible
-- Final eligibility including concentration pre-checks
{passes_basic_eligibility}
```

### **3. Concentration Calculations**

```sql
-- CALC: total_pool_balance
sumOver(
  {scenario_balance},
  [],
  PRE_AGG
)

-- CALC: single_obligor_balance
sumOver(
  {scenario_balance},
  [{obligor_id}],
  PRE_AGG
)

-- CALC: single_obligor_concentration
{single_obligor_balance} / {total_pool_balance}

-- CALC: obligor_concentration_breach
ifelse(
  {single_obligor_concentration} > 0.10, -- 10% limit
  1, 0
)

-- CALC: industry_balance
sumOver(
  {scenario_balance},
  [{naics3_code}],
  PRE_AGG
)

-- CALC: industry_concentration
{industry_balance} / {total_pool_balance}

-- CALC: industry_concentration_limit
-- Industry-specific limits
ifelse(
  {naics3_code} = '722', 0.15,  -- Restaurants: 15% max
  {naics3_code} = '236', 0.12,  -- Construction: 12% max
  {naics3_code} IN ('441', '442', '443', '444', '445'), 0.20,  -- Retail: 20% max
  0.25  -- Default: 25% max
)

-- CALC: industry_breach
ifelse(
  {industry_concentration} > {industry_concentration_limit}, 1, 0
)

-- CALC: geographic_balance
sumOver(
  {scenario_balance},
  [{state_code}],
  PRE_AGG
)

-- CALC: geographic_concentration
{geographic_balance} / {total_pool_balance}

-- CALC: top_10_rank
denseRank(
  [{current_principal} DESC],
  [{deal_id}]
)

-- CALC: is_top_10_deal
ifelse({top_10_rank} <= 10, 1, 0)

-- CALC: top_10_balance
sumOver(
  ifelse({is_top_10_deal} = 1, {scenario_balance}, 0),
  [],
  PRE_AGG
)

-- CALC: top_10_concentration
{top_10_balance} / {total_pool_balance}

-- CALC: weighted_avg_fico
sumOver({scenario_balance} * {fico_score}, [], PRE_AGG) / 
{total_pool_balance}

-- CALC: weighted_avg_rate
sumOver({scenario_balance} * {interest_rate}, [], PRE_AGG) / 
{total_pool_balance}

-- CALC: weighted_avg_life_years
sumOver(
  {scenario_balance} * dateDiff({maturity_date}, now(), 'YYYY'),
  [],
  PRE_AGG
) / {total_pool_balance}
```

### **4. Sub-Tranche Allocation**

```sql
-- CALC: sub_tranche_target_pct
-- Investor A's sub-tranche targets
ifelse(
  AND({investor_name} = 'Investor A', {sub_tranche} = 'A1'), 0.40,
  AND({investor_name} = 'Investor A', {sub_tranche} = 'A2'), 0.35,
  AND({investor_name} = 'Investor A', {sub_tranche} = 'A3'), 0.25,
  
  AND({investor_name} = 'Investor B', {sub_tranche} = 'B1'), 0.60,
  AND({investor_name} = 'Investor B', {sub_tranche} = 'B2'), 0.40,
  
  1.00  -- Default: 100% for investors without sub-tranches
)

-- CALC: sub_tranche_actual_balance
sumOver(
  {scenario_balance},
  [{investor_name}, {sub_tranche}],
  PRE_AGG
)

-- CALC: investor_total_balance
sumOver(
  {scenario_balance},
  [{investor_name}],
  PRE_AGG
)

-- CALC: sub_tranche_actual_pct
{sub_tranche_actual_balance} / {investor_total_balance}

-- CALC: sub_tranche_variance
{sub_tranche_actual_pct} - {sub_tranche_target_pct}

-- CALC: sub_tranche_status
ifelse(
  abs({sub_tranche_variance}) <= 0.02, 'On Target',
  {sub_tranche_variance} > 0, 'Over-Allocated',
  'Under-Allocated'
)
```

### **5. Comparison Metrics (Base vs Scenario)**

```sql
-- CALC: base_balance
-- Balance without scenario modifications
ifelse(
  AND(
    {is_in_scope} = 1,
    {investor_name} = ${selected_investor}
  ),
  {current_principal},
  0
)

-- CALC: scenario_delta_balance
{scenario_balance} - {base_balance}

-- CALC: scenario_delta_count
-- Count of deals added/removed
sumOver(
  ifelse({scenario_delta_balance} != 0, 1, 0),
  [],
  PRE_AGG
)

-- CALC: deals_added_count
sumOver(
  ifelse({scenario_delta_balance} > 0, 1, 0),
  [],
  PRE_AGG
)

-- CALC: deals_removed_count
sumOver(
  ifelse({scenario_delta_balance} < 0, 1, 0),
  [],
  PRE_AGG
)
```

## **Dashboard Structure**

### **Sheet 1: Executive Summary**

```
┌─────────────────────────────────────────────────────────────┐
│ CONTROLS                                                    │
│ [Investor ▼] [Include Unallocated: Yes ▼] [Scenario ▼]    │
└─────────────────────────────────────────────────────────────┘

┌──────────────────┬──────────────────┬──────────────────────┐
│ TOTAL POOL       │ ELIGIBLE         │ SCENARIO IMPACT      │
│ $XXX.XM          │ $XXX.XM (XX%)   │ ▲ $XX.XM            │
│ X,XXX Deals      │ X,XXX Deals     │ +XX / -XX Deals     │
└──────────────────┴──────────────────┴──────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ CONCENTRATION TESTS                                         │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ Test              Current  Limit   Status   Delta   │   │
│ │ Single Obligor     8.5%    10.0%   ✓ OK    +0.2%   │   │
│ │ Top 10            45.2%    50.0%   ✓ OK    +1.5%   │   │
│ │ Industry: 722     16.3%    15.0%   ✗ BREACH +2.1%  │   │
│ │ Industry: 236     11.8%    12.0%   ✓ OK    -0.3%   │   │
│ │ Geographic: CA    22.1%    25.0%   ✓ OK    +0.8%   │   │
│ └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ PORTFOLIO CHARACTERISTICS                                   │
│                                                             │
│  Weighted Avg FICO: 685 (req: 680)                        │
│  Weighted Avg Rate: 8.25%                                  │
│  Weighted Avg Life: 3.2 years                              │
│  Count by Grade: [Bar Chart]                               │
│  Industry Mix: [Treemap]                                   │
└─────────────────────────────────────────────────────────────┘
```

### **Sheet 2: Investor A Analytics**

```
┌─────────────────────────────────────────────────────────────┐
│ INVESTOR A OVERVIEW                                         │
│ [Sub-Tranche: All ▼] [View: Summary ▼]                    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ SUB-TRANCHE ALLOCATION                                      │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ Tranche  Target   Actual  Variance  Status          │   │
│ │ A1       40.0%    41.2%   +1.2%     Over ⚠         │   │
│ │ A2       35.0%    33.8%   -1.2%     Under ⚠        │   │
│ │ A3       25.0%    25.0%    0.0%     On Target ✓    │   │
│ └─────────────────────────────────────────────────────┘   │
│                                                             │
│ [Waterfall Chart showing allocation variance]              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ ELIGIBILITY BREAKDOWN                                       │
│                                                             │
│  Eligible Deals:      X,XXX ($XXX.XM)                     │
│  Ineligible Deals:    XXX ($XX.XM)                        │
│                                                             │
│  Top Ineligibility Reasons:                                │
│  • FICO Below Threshold: XXX deals ($XX.XM)               │
│  • TIB Below Threshold: XX deals ($X.XM)                  │
│  • Not Current: XX deals ($X.XM)                          │
│                                                             │
│  [Stacked Bar: Eligible vs Ineligible by Industry]        │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ CONCENTRATION HEATMAP (by Sub-Tranche)                     │
│                                                             │
│         A1      A2      A3                                 │
│  722   🟢15%   🟡14%   🟢12%                              │
│  236   🟢 8%   🟢10%   🟢 9%                              │
│  44X   🟢18%   🟢16%   🟡20%                              │
│                                                             │
│  🟢 = Within Limit  🟡 = Warning  🔴 = Breach             │
└─────────────────────────────────────────────────────────────┘
```

### **Sheet 3: Scenario Builder (What-If Analysis)**

```
┌─────────────────────────────────────────────────────────────┐
│ SCENARIO BUILDER                                            │
│ Current Scenario: [Scenario 1 ▼] [Save] [Reset] [Export]  │
└─────────────────────────────────────────────────────────────┘

┌────────────────────┬────────────────────────────────────────┐
│ AVAILABLE POOL     │ CURRENT SELECTION                      │
│ (Unallocated)      │                                        │
│                    │ Base: X,XXX deals ($XXX.XM)           │
│ [Search/Filter]    │ + Added: XX deals ($XX.XM)            │
│                    │ - Removed: XX deals ($XX.XM)          │
│ ┌────────────────┐ │ = Total: X,XXX deals ($XXX.XM)       │
│ │ Deal  Balance │ │                                        │
│ │ [Add Button]  │ │ ┌────────────────────────────────┐    │
│ │ D001  $1.2M   │ │ │ Deal   Action    Impact        │    │
│ │ D002  $0.8M   │ │ │ D050   Added    Oblg Conc +1%  │    │
│ │ D003  $1.5M   │ │ │ D051   Added    Ind722 +0.5%   │    │
│ │ ...           │ │ │ D100   Removed  Top10 -2%      │    │
│ └────────────────┘ │ └────────────────────────────────┘    │
│                    │                                        │
│ Bulk Actions:      │ [Remove All Selected]                 │
│ [Add Filtered]     │                                        │
│ [Upload CSV]       │                                        │
└────────────────────┴────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ IMPACT ANALYSIS                                             │
│                                                             │
│  Before → After Comparison:                                │
│                                                             │
│  [Side-by-side concentration bars showing delta]           │
│  [Line chart: Metric trends across scenarios]              │
│                                                             │
│  New Breaches: X                                           │
│  Resolved Breaches: X                                      │
└─────────────────────────────────────────────────────────────┘
```

### **Sheet 4: Deal-Level Explorer**

```
┌─────────────────────────────────────────────────────────────┐
│ DEAL-LEVEL ANALYSIS                                         │
│ [Show: All ▼] [Industry ▼] [Grade ▼] [Eligibility ▼]     │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ DEALS TABLE (Paginated - showing 50 of X,XXX)             │
│ ┌───────────────────────────────────────────────────────┐ │
│ │ Deal  Obligor Industry Grade FICO TIB  Balance Status │ │
│ │       Required  Actual                                │ │
│ ├───────────────────────────────────────────────────────┤ │
│ │ D001  Acme    722-A   A+   680   685 ✓  24  30 ✓     │ │
│ │       Rest.                 680       24              │ │
│ │       $1.2M                         Eligible ✓        │ │
│ ├───────────────────────────────────────────────────────┤ │
│ │ D002  ABC     236-B   B    700   675 ✗  42  36 ✗     │ │
│ │       Const.                700       42              │ │
│ │       $0.8M           Ineligible: FICO, TIB ✗        │ │
│ └───────────────────────────────────────────────────────┘ │
│                                                             │
│ [Export to Excel] [< Prev] [Page 1 of 240] [Next >]      │
└─────────────────────────────────────────────────────────────┘

[Click on deal row → Opens drill-through sheet with:]
- Full deal details
- Contribution to each concentration test
- Historical performance
- Related obligor deals
- Eligibility test details
```

### **Sheet 5: Industry Deep Dive**

```
┌─────────────────────────────────────────────────────────────┐
│ INDUSTRY ANALYSIS                                           │
│ Selected Industry: [722 - Restaurants ▼]                   │
└─────────────────────────────────────────────────────────────┘

┌──────────────────┬──────────────────┬──────────────────────┐
│ INDUSTRY SUMMARY │ ELIGIBILITY RULES│ CONCENTRATION        │
│                  │                  │                      │
│ Total: $XX.XM    │ Grade A:         │ Current: 16.3%      │
│ XX.X% of Pool    │  FICO ≥ 680     │ Limit: 15.0%        │
│ XXX Deals        │  TIB ≥ 24 mo    │ Status: BREACH 🔴   │
│                  │                  │                      │
│ Avg FICO: XXX    │ Grade B:         │ [Gauge Chart]       │
│ Avg TIB: XX mo   │  FICO ≥ 650     │                      │
│                  │  TIB ≥ 30 mo    │                      │
└──────────────────┴──────────────────┴──────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ GRADE DISTRIBUTION                                          │
│ [Stacked bar showing A/B/C grade split with pass/fail]    │
│                                                             │
│ A Grade: XXX deals (XX% eligible)                          │
│ B Grade: XXX deals (XX% eligible)                          │
│ C Grade: XXX deals (XX% eligible)                          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ RECOMMENDATIONS                                             │
│                                                             │
│ To reduce concentration to 15%:                            │
│ • Remove XX deals ($X.XM)                                  │
│ • Suggested deals to remove: [Top concentrators]          │
│                                                             │
│ Available replacement deals from Unallocated:              │
│ • XX deals in other industries ($X.XM)                    │
└─────────────────────────────────────────────────────────────┘
```

## **Advanced Features Implementation**

### **Feature 1: Smart Deal Selector**

```sql
-- CALC: deal_impact_score
-- Scores deals based on multiple factors for optimization
(
  ({current_principal} / {total_pool_balance}) * 0.3 +  -- Size impact
  (1 - {single_obligor_concentration}) * 0.2 +  -- Diversification benefit
  (1 - {industry_concentration}) * 0.2 +  -- Industry diversification
  ({fico_score} / 850) * 0.15 +  -- Quality
  ({interest_rate} / 15) * 0.15  -- Yield contribution
) * 100

-- CALC: deal_recommendation
-- Flags deals that would improve portfolio
ifelse(
  AND(
    {is_eligible} = 1,
    {investor_name} = 'Unallocated',
    {deal_impact_score} > 70,
    {obligor_concentration_breach} = 0,  -- Won't create breach
    {industry_breach} = 0
  ),
  'Recommended',
  ifelse(
    {is_eligible} = 1,
    'Available',
    'Ineligible'
  )
)
```

### **Feature 2: Breach Alerting**

```sql
-- CALC: breach_summary
concat(
  ifelse({obligor_concentration_breach} = 1, '⚠ Obligor Breach; ', ''),
  ifelse({industry_breach} = 1, '⚠ Industry Breach; ', ''),
  ifelse({top_10_concentration} > 0.50, '⚠ Top 10 Breach; ', ''),
  ifelse({geographic_concentration} > 0.25, '⚠ Geo Breach; ', '')
)

-- CALC: breach_severity
-- 0 = OK, 1 = Warning, 2 = Breach, 3 = Critical
ifelse(
  strlen({breach_summary}) = 0, 0,
  ifelse(
    OR(
      {single_obligor_concentration} > 0.12,  -- 20% over limit
      {industry_concentration} > {industry_concentration_limit} * 1.2
    ), 3,
    ifelse(strlen({breach_summary}) > 50, 2, 1)
  )
)

-- Use conditional formatting:
-- breach_severity = 0: Green
-- breach_severity = 1: Yellow
-- breach_severity = 2: Orange
-- breach_severity = 3: Red
```

### **Feature 3: Historical Tracking**

If you create daily snapshots in your dataset:

```sql
-- CALC: concentration_trend_7d
{single_obligor_concentration} - 
avgOver(
  {single_obligor_concentration},
  [{obligor_id}],
  [{as_of_date} ASC],
  7,
  PRE_AGG
)

-- CALC: concentration_direction
ifelse(
  {concentration_trend_7d} > 0.01, '↑ Increasing',
  ifelse({concentration_trend_7d} < -0.01, '↓ Decreasing', '→ Stable')
)
```

### **Feature 4: Export Functionality**

Add URL actions:
```javascript
// Create export URL parameter
CALC: export_url
concat(
  'https://your-api.com/export?',
  'investor=', ${selected_investor},
  '&scenario=', ${scenario_mode},
  '&added=', toString(${deals_to_add}),
  '&removed=', toString(${deals_to_remove})
)
```

## **Implementation Checklist**

### **Phase 1: Foundation (Week 1)**
- ✅ Create all parameters
- ✅ Build core calculated fields (filtering, eligibility)
- ✅ Test with sample investor
- ✅ Create Executive Summary sheet

### **Phase 2: Investor Sheets (Week 2)**
- ✅ Build Investor A analytics sheet
- ✅ Implement sub-tranche logic
- ✅ Clone for Investors B & C
- ✅ Add concentration heatmaps

### **Phase 3: Scenario Builder (Week 3)**
- ✅ Create scenario parameter controls
- ✅ Build Available/Selected pools
- ✅ Implement impact analysis
- ✅ Add save/load scenario functionality

### **Phase 4: Advanced Features (Week 4)**
- ✅ Deal-level explorer with pagination
- ✅ Industry deep dive sheets
- ✅ Smart recommendations
- ✅ Breach alerting
- ✅ Export functionality

### **Phase 5: Polish & UAT (Week 5)**
- ✅ Performance optimization (SPICE)
- ✅ Mobile responsiveness
- ✅ User acceptance testing
- ✅ Documentation

Would you like me to:
1. Create the actual QuickSight JSON for specific calculated fields?
2. Design the SQL for a more sophisticated eligibility rules table?
3. Build a Lambda function for complex scenario persistence?
4. Create a data governance framework for this dashboard?
