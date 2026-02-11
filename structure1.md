Enterprise MCA Eligibility Engine Architecture
I’ll design a comprehensive, scalable solution that transforms your three hardcoded views into a rules-driven eligibility framework.
🏗️ Layered Architecture Overview

┌─────────────────────────────────────────────────────┐
│ Layer 4: ELIGIBILITY ROLLUP                         │
│ └─ deal_eligibility_summary                         │
│    Final pass/fail + scoring per deal/company       │
└─────────────────────────────────────────────────────┘
                      ▲
┌─────────────────────────────────────────────────────┐
│ Layer 3: RULES EVALUATION                           │
│ └─ deal_rule_evaluation                             │
│    Binary result (1/0) per rule per deal            │
└─────────────────────────────────────────────────────┘
                      ▲
┌─────────────────────────────────────────────────────┐
│ Layer 2: DEAL ATTRIBUTES                            │
│ └─ deal_attributes (normalized source)              │
│    Centralized deal metrics (FICO, TIB, etc.)       │
└─────────────────────────────────────────────────────┘
                      ▲
┌─────────────────────────────────────────────────────┐
│ Layer 1: RULES CONFIGURATION                        │
│ ├─ eligibility_rules_catalog                        │
│ ├─ investment_company_rule_config                   │
│ └─ rule_threshold_mappings                          │
└─────────────────────────────────────────────────────┘


📋 Layer 1: Rules Configuration Tables
1.1 eligibility_rules_catalog
Master list of all possible eligibility checks.

CREATE TABLE eligibility_rules_catalog (
    rule_id                 VARCHAR(50) PRIMARY KEY,
    rule_name               VARCHAR(200) NOT NULL,
    rule_category           VARCHAR(50),  -- 'credit', 'business', 'deal_terms', 'industry'
    rule_description        VARCHAR(500),
    evaluation_logic_type   VARCHAR(50),  -- 'threshold', 'allowed_list', 'custom_sql'
    attribute_name          VARCHAR(100), -- Links to deal_attributes column
    data_type               VARCHAR(20),  -- 'numeric', 'string', 'date'
    is_active               BOOLEAN DEFAULT TRUE,
    created_date            TIMESTAMP,
    updated_date            TIMESTAMP
);

-- Example rows
INSERT INTO eligibility_rules_catalog VALUES
('FICO_MIN', 'Minimum FICO Score', 'credit', 'Borrower FICO must meet threshold', 'threshold', 'fico_score', 'numeric', TRUE, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP),
('TIB_MIN', 'Minimum Time in Business', 'business', 'Business must be operational for minimum months', 'threshold', 'months_in_business', 'numeric', TRUE, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP),
('CREDIT_GRADE_ALLOWED', 'Allowed Credit Grades', 'credit', 'Credit grade must be in allowed list', 'allowed_list', 'credit_grade', 'string', TRUE, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP),
('INDUSTRY_RESTRICTED', 'Restricted Industries', 'industry', 'Industry code not in restricted list', 'allowed_list', 'naics_code', 'string', TRUE, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP),
('MAX_TERM_MONTHS', 'Maximum Term Length', 'deal_terms', 'Deal term cannot exceed threshold', 'threshold', 'term_months', 'numeric', TRUE, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);


1.2 investment_company_rule_config
Which rules apply to each investment company and their weight/criticality.

CREATE TABLE investment_company_rule_config (
    config_id               VARCHAR(100) PRIMARY KEY,
    investment_company_id   VARCHAR(50) NOT NULL,
    rule_id                 VARCHAR(50) NOT NULL,
    is_required             BOOLEAN DEFAULT TRUE,     -- Hard stop vs. soft warning
    rule_weight             DECIMAL(5,2) DEFAULT 1.0, -- For scoring systems
    failure_severity        VARCHAR(20),              -- 'critical', 'major', 'minor'
    effective_date          DATE NOT NULL,
    expiration_date         DATE,
    is_active               BOOLEAN DEFAULT TRUE,
    
    FOREIGN KEY (rule_id) REFERENCES eligibility_rules_catalog(rule_id)
);

-- Example: InvestCo A has stricter FICO requirements
INSERT INTO investment_company_rule_config VALUES
('IC_A_FICO', 'INVESTCO_A', 'FICO_MIN', TRUE, 2.0, 'critical', '2024-01-01', NULL, TRUE),
('IC_B_FICO', 'INVESTCO_B', 'FICO_MIN', TRUE, 1.0, 'major', '2024-01-01', NULL, TRUE),
('IC_C_FICO', 'INVESTCO_C', 'FICO_MIN', FALSE, 0.5, 'minor', '2024-01-01', NULL, TRUE);


1.3 rule_threshold_mappings
Actual thresholds and allowed values per company per rule.

CREATE TABLE rule_threshold_mappings (
    mapping_id              VARCHAR(100) PRIMARY KEY,
    investment_company_id   VARCHAR(50) NOT NULL,
    rule_id                 VARCHAR(50) NOT NULL,
    threshold_type          VARCHAR(20),  -- 'min', 'max', 'range', 'equals', 'allowed_list', 'excluded_list'
    
    -- Numeric thresholds
    numeric_min_value       DECIMAL(18,4),
    numeric_max_value       DECIMAL(18,4),
    
    -- String/categorical values
    allowed_values          VARCHAR(5000), -- JSON array or CSV
    excluded_values         VARCHAR(5000),
    
    -- Date thresholds
    date_threshold          DATE,
    
    effective_date          DATE NOT NULL,
    expiration_date         DATE,
    is_active               BOOLEAN DEFAULT TRUE,
    
    FOREIGN KEY (rule_id) REFERENCES eligibility_rules_catalog(rule_id)
);

-- Examples
INSERT INTO rule_threshold_mappings VALUES
-- FICO thresholds vary by company
('IC_A_FICO_MIN', 'INVESTCO_A', 'FICO_MIN', 'min', 680, NULL, NULL, NULL, NULL, '2024-01-01', NULL, TRUE),
('IC_B_FICO_MIN', 'INVESTCO_B', 'FICO_MIN', 'min', 650, NULL, NULL, NULL, NULL, '2024-01-01', NULL, TRUE),
('IC_C_FICO_MIN', 'INVESTCO_C', 'FICO_MIN', 'min', 620, NULL, NULL, NULL, NULL, '2024-01-01', NULL, TRUE),

-- Credit grades allowed (JSON array format)
('IC_A_GRADES', 'INVESTCO_A', 'CREDIT_GRADE_ALLOWED', 'allowed_list', NULL, NULL, '["A","B","C"]', NULL, NULL, '2024-01-01', NULL, TRUE),
('IC_B_GRADES', 'INVESTCO_B', 'CREDIT_GRADE_ALLOWED', 'allowed_list', NULL, NULL, '["A","B","C","D"]', NULL, NULL, '2024-01-01', NULL, TRUE),

-- Restricted industries (NAICS codes)
('IC_A_INDUSTRY', 'INVESTCO_A', 'INDUSTRY_RESTRICTED', 'excluded_list', NULL, NULL, NULL, '["7132","7211","9999"]', NULL, '2024-01-01', NULL, TRUE);


📊 Layer 2: Deal Attributes View
Normalized, single source of truth for all deal metrics.

CREATE VIEW deal_attributes AS
SELECT 
    d.deal_id,
    d.investment_company_id,
    d.deal_date,
    d.merchant_id,
    
    -- Credit attributes
    c.fico_score,
    c.credit_grade,
    c.credit_score,
    
    -- Business attributes
    b.months_in_business,
    b.naics_code,
    b.business_type,
    b.annual_revenue,
    
    -- Deal terms
    d.funded_amount,
    d.term_months,
    d.factor_rate,
    d.payment_frequency,
    
    -- Risk metrics
    r.debt_to_income,
    r.utilization_rate,
    r.default_probability_score,
    
    -- Timestamps
    CURRENT_TIMESTAMP AS evaluation_timestamp
    
FROM deals d
LEFT JOIN credit_profiles c ON d.merchant_id = c.merchant_id AND c.is_current = TRUE
LEFT JOIN business_profiles b ON d.merchant_id = b.merchant_id
LEFT JOIN risk_scores r ON d.deal_id = r.deal_id;


⚙️ Layer 3: Rules Evaluation View
This is where the magic happens—evaluating each rule against each deal.

CREATE VIEW deal_rule_evaluation AS
WITH rule_configs AS (
    -- Get active rules for each investment company
    SELECT 
        rc.rule_id,
        rc.rule_name,
        rc.rule_category,
        rc.attribute_name,
        rc.evaluation_logic_type,
        rc.data_type,
        icc.investment_company_id,
        icc.is_required,
        icc.rule_weight,
        icc.failure_severity,
        tm.threshold_type,
        tm.numeric_min_value,
        tm.numeric_max_value,
        tm.allowed_values,
        tm.excluded_values
    FROM eligibility_rules_catalog rc
    INNER JOIN investment_company_rule_config icc 
        ON rc.rule_id = icc.rule_id 
        AND icc.is_active = TRUE
        AND CURRENT_DATE BETWEEN icc.effective_date AND COALESCE(icc.expiration_date, '9999-12-31')
    LEFT JOIN rule_threshold_mappings tm 
        ON rc.rule_id = tm.rule_id 
        AND icc.investment_company_id = tm.investment_company_id
        AND tm.is_active = TRUE
        AND CURRENT_DATE BETWEEN tm.effective_date AND COALESCE(tm.expiration_date, '9999-12-31')
)

SELECT 
    da.deal_id,
    da.investment_company_id,
    rc.rule_id,
    rc.rule_name,
    rc.rule_category,
    rc.is_required,
    rc.rule_weight,
    rc.failure_severity,
    
    -- Dynamic evaluation based on rule type
    CASE rc.rule_id
        -- Numeric threshold comparisons
        WHEN 'FICO_MIN' THEN 
            CASE WHEN da.fico_score >= rc.numeric_min_value THEN 1 ELSE 0 END
        WHEN 'TIB_MIN' THEN 
            CASE WHEN da.months_in_business >= rc.numeric_min_value THEN 1 ELSE 0 END
        WHEN 'MAX_TERM_MONTHS' THEN 
            CASE WHEN da.term_months <= rc.numeric_max_value THEN 1 ELSE 0 END
        WHEN 'MAX_FUNDED_AMT' THEN 
            CASE WHEN da.funded_amount <= rc.numeric_max_value THEN 1 ELSE 0 END
            
        -- Allowed list checks (using JSON functions for Redshift)
        WHEN 'CREDIT_GRADE_ALLOWED' THEN 
            CASE WHEN JSON_SERIALIZE(JSON_PARSE(rc.allowed_values)) LIKE '%' || da.credit_grade || '%' 
                 THEN 1 ELSE 0 END
                 
        -- Excluded list checks
        WHEN 'INDUSTRY_RESTRICTED' THEN 
            CASE WHEN JSON_SERIALIZE(JSON_PARSE(rc.excluded_values)) NOT LIKE '%' || da.naics_code || '%' 
                 THEN 1 ELSE 0 END
                 
        -- Range checks
        WHEN 'FACTOR_RATE_RANGE' THEN 
            CASE WHEN da.factor_rate BETWEEN rc.numeric_min_value AND rc.numeric_max_value 
                 THEN 1 ELSE 0 END
                 
        ELSE NULL -- Add more rules as needed
    END AS rule_passed_flag,
    
    -- Capture actual values for audit trail
    CASE rc.attribute_name
        WHEN 'fico_score' THEN da.fico_score::VARCHAR
        WHEN 'months_in_business' THEN da.months_in_business::VARCHAR
        WHEN 'credit_grade' THEN da.credit_grade
        WHEN 'naics_code' THEN da.naics_code
        WHEN 'term_months' THEN da.term_months::VARCHAR
        WHEN 'funded_amount' THEN da.funded_amount::VARCHAR
        ELSE NULL
    END AS actual_value,
    
    -- Capture threshold for comparison
    CASE rc.threshold_type
        WHEN 'min' THEN rc.numeric_min_value::VARCHAR
        WHEN 'max' THEN rc.numeric_max_value::VARCHAR
        WHEN 'allowed_list' THEN rc.allowed_values
        WHEN 'excluded_list' THEN rc.excluded_values
        ELSE NULL
    END AS threshold_value,
    
    da.evaluation_timestamp
    
FROM deal_attributes da
CROSS JOIN rule_configs rc
WHERE da.investment_company_id = rc.investment_company_id;


📈 Layer 4: Eligibility Rollup View
Aggregate rule results into final eligibility determination.

CREATE VIEW deal_eligibility_summary AS
WITH rule_results AS (
    SELECT 
        deal_id,
        investment_company_id,
        
        -- Overall flags
        MIN(CASE WHEN is_required = TRUE THEN rule_passed_flag ELSE 1 END) AS all_required_rules_passed,
        MIN(rule_passed_flag) AS all_rules_passed,
        
        -- Counts
        COUNT(*) AS total_rules_evaluated,
        SUM(rule_passed_flag) AS rules_passed_count,
        SUM(CASE WHEN is_required = TRUE THEN 1 ELSE 0 END) AS required_rules_count,
        SUM(CASE WHEN is_required = TRUE AND rule_passed_flag = 1 THEN 1 ELSE 0 END) AS required_rules_passed_count,
        
        -- Weighted scoring
        SUM(rule_passed_flag * rule_weight) AS weighted_pass_score,
        SUM(rule_weight) AS max_possible_score,
        
        -- Failure details
        LISTAGG(
            CASE WHEN rule_passed_flag = 0 THEN rule_name ELSE NULL END, 
            '; '
        ) WITHIN GROUP (ORDER BY failure_severity, rule_name) AS failed_rules_list,
        
        MAX(CASE WHEN rule_passed_flag = 0 AND failure_severity = 'critical' THEN 1 ELSE 0 END) AS has_critical_failure,
        
        MAX(evaluation_timestamp) AS evaluation_timestamp
        
    FROM deal_rule_evaluation
    GROUP BY deal_id, investment_company_id
)

SELECT 
    rr.*,
    
    -- Final eligibility determination
    CASE 
        WHEN all_required_rules_passed = 1 THEN 1 
        ELSE 0 
    END AS is_eligible,
    
    -- Risk score (0-100)
    ROUND((weighted_pass_score / NULLIF(max_possible_score, 0)) * 100, 2) AS eligibility_score,
    
    -- QuickSight-friendly flags
    CASE WHEN all_required_rules_passed = 1 THEN 'ELIGIBLE' ELSE 'INELIGIBLE' END AS eligibility_status,
    
    CASE 
        WHEN has_critical_failure = 1 THEN 'CRITICAL_FAILURE'
        WHEN all_required_rules_passed = 0 THEN 'REQUIRED_RULE_FAILURE'
        WHEN all_rules_passed = 0 THEN 'SOFT_FAILURE'
        ELSE 'PASS'
    END AS failure_reason_category

FROM rule_results;


🔄 Generic Rule Evaluation Function (Alternative Approach)
For even more flexibility, use a SQL UDF or Python UDF in Redshift:

CREATE OR REPLACE FUNCTION evaluate_rule(
    rule_type VARCHAR,
    actual_value VARCHAR,
    threshold_min DECIMAL,
    threshold_max DECIMAL,
    allowed_list VARCHAR,
    excluded_list VARCHAR
)
RETURNS INTEGER
STABLE
AS $$
    import json
    
    if actual_value is None:
        return 0
    
    if rule_type == 'min':
        return 1 if float(actual_value) >= threshold_min else 0
    
    elif rule_type == 'max':
        return 1 if float(actual_value) <= threshold_max else 0
    
    elif rule_type == 'range':
        val = float(actual_value)
        return 1 if threshold_min <= val <= threshold_max else 0
    
    elif rule_type == 'allowed_list':
        allowed = json.loads(allowed_list) if allowed_list else []
        return 1 if actual_value in allowed else 0
    
    elif rule_type == 'excluded_list':
        excluded = json.loads(excluded_list) if excluded_list else []
        return 1 if actual_value not in excluded else 0
    
    else:
        return None
$$ LANGUAGE plpythonu;


Then simplify your evaluation view:

CREATE VIEW deal_rule_evaluation_v2 AS
SELECT 
    da.deal_id,
    rc.rule_id,
    rc.rule_name,
    evaluate_rule(
        rc.threshold_type,
        da.attribute_values[rc.attribute_name], -- Using a JSON column or dynamic lookup
        tm.numeric_min_value,
        tm.numeric_max_value,
        tm.allowed_values,
        tm.excluded_values
    ) AS rule_passed_flag
FROM deal_attributes da
CROSS JOIN rule_configs rc
...


🚀 Scaling to N Investment Companies
Zero Code Changes Required
To add a new investment company:

-- 1. Add company-specific rule configurations
INSERT INTO investment_company_rule_config 
SELECT 
    'IC_D_' || rule_id AS config_id,
    'INVESTCO_D' AS investment_company_id,
    rule_id,
    is_required,
    rule_weight,
    failure_severity,
    CURRENT_DATE AS effective_date,
    NULL AS expiration_date,
    TRUE AS is_active
FROM investment_company_rule_config
WHERE investment_company_id = 'INVESTCO_A'; -- Clone from template

-- 2. Set company-specific thresholds
INSERT INTO rule_threshold_mappings 
VALUES ('IC_D_FICO_MIN', 'INVESTCO_D', 'FICO_MIN', 'min', 640, NULL, ...);

-- 3. Done! Views automatically include new company


Rule Reusability
	∙	Rules defined once in catalog
	∙	Each company “subscribes” to rules via config table
	∙	Thresholds fully parameterized

⚡ Redshift Performance Optimization
1. Materialized Views for Layer 3 & 4

CREATE MATERIALIZED VIEW mv_deal_rule_evaluation
DISTSTYLE KEY
DISTKEY(deal_id)
SORTKEY(investment_company_id, deal_id, rule_category)
AS SELECT * FROM deal_rule_evaluation;

-- Refresh strategy
REFRESH MATERIALIZED VIEW mv_deal_rule_evaluation;


2. Partitioning by Date

-- Add partition key to deal_attributes
CREATE TABLE deal_attributes_partitioned (
    deal_id VARCHAR,
    evaluation_date DATE,
    ...
)
PARTITION BY (evaluation_date);


3. Compression & Distribution

-- Optimize configuration tables
ALTER TABLE eligibility_rules_catalog 
    ALTER COLUMN rule_id ENCODE LZO,
    ALTER COLUMN rule_description ENCODE LZO;

-- Replicate small config tables across all nodes
CREATE TABLE eligibility_rules_catalog
DISTSTYLE ALL;


4. Columnar Storage Best Practices
	∙	Only SELECT columns you need in downstream views
	∙	Avoid SELECT * in production views
	∙	Use COPY command for bulk config updates

📊 QuickSight Integration Pattern
Eligibility Dashboard Dataset

CREATE VIEW quicksight_eligibility_dashboard AS
SELECT 
    d.deal_id,
    d.merchant_name,
    d.deal_date,
    d.funded_amount,
    es.investment_company_id,
    es.is_eligible,
    es.eligibility_score,
    es.eligibility_status,
    es.total_rules_evaluated,
    es.rules_passed_count,
    es.failed_rules_list,
    es.failure_reason_category,
    
    -- Drill-down capability
    dre.rule_name,
    dre.rule_category,
    dre.rule_passed_flag,
    dre.actual_value,
    dre.threshold_value
    
FROM deal_eligibility_summary es
INNER JOIN deals d ON es.deal_id = d.deal_id
LEFT JOIN deal_rule_evaluation dre 
    ON es.deal_id = dre.deal_id 
    AND es.investment_company_id = dre.investment_company_id;


QuickSight Calculated Fields

Pass Rate = {rules_passed_count} / {total_rules_evaluated}
Eligibility Trend = last({is_eligible}, [{deal_date} DESC])


🔍 Auditability & Compliance Features
1. Rule Change History Table

CREATE TABLE rule_config_audit_log (
    audit_id            BIGINT IDENTITY(1,1) PRIMARY KEY,
    table_name          VARCHAR(100),
    record_id           VARCHAR(100),
    change_type         VARCHAR(20), -- 'INSERT', 'UPDATE', 'DELETE'
    changed_by          VARCHAR(100),
    change_timestamp    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    old_values          VARCHAR(MAX), -- JSON
    new_values          VARCHAR(MAX)  -- JSON
);

-- Trigger or stored procedure to capture changes
CREATE PROCEDURE log_rule_change(...)
AS
BEGIN
    INSERT INTO rule_config_audit_log ...;
END;


2. Evaluation Snapshot Table
For regulatory reporting, persist point-in-time evaluations:

CREATE TABLE deal_eligibility_snapshot (
    snapshot_id         BIGINT IDENTITY(1,1) PRIMARY KEY,
    deal_id             VARCHAR(50),
    investment_company_id VARCHAR(50),
    snapshot_date       DATE,
    is_eligible         INTEGER,
    eligibility_score   DECIMAL(5,2),
    failed_rules_json   VARCHAR(MAX),
    evaluation_details  VARCHAR(MAX) -- Full rule results as JSON
)
DISTKEY(deal_id)
SORTKEY(snapshot_date, deal_id);

-- Daily snapshot job
INSERT INTO deal_eligibility_snapshot
SELECT 
    deal_id,
    investment_company_id,
    CURRENT_DATE,
    is_eligible,
    eligibility_score,
    JSON_SERIALIZE(failed_rules_list),
    JSON_SERIALIZE(...)
FROM deal_eligibility_summary;


3. Rule Lineage View

CREATE VIEW rule_lineage AS
SELECT 
    rc.rule_id,
    rc.rule_name,
    icc.investment_company_id,
    tm.threshold_type,
    tm.numeric_min_value,
    tm.effective_date,
    COUNT(DISTINCT dre.deal_id) AS deals_evaluated,
    SUM(dre.rule_passed_flag) AS deals_passed,
    AVG(dre.rule_passed_flag) AS pass_rate
FROM eligibility_rules_catalog rc
LEFT JOIN investment_company_rule_config icc ON rc.rule_id = icc.rule_id
LEFT JOIN rule_threshold_mappings tm ON rc.rule_id = tm.rule_id
LEFT JOIN deal_rule_evaluation dre ON rc.rule_id = dre.rule_id
GROUP BY 1,2,3,4,5,6;


🔗 Downstream Integration
Borrowing Base Calculation

CREATE VIEW borrowing_base_eligible_deals AS
SELECT 
    es.investment_company_id,
    SUM(d.funded_amount) AS total_eligible_exposure,
    SUM(d.outstanding_balance) AS total_eligible_balance,
    COUNT(DISTINCT es.deal_id) AS eligible_deal_count,
    AVG(es.eligibility_score) AS avg_eligibility_score
FROM deal_eligibility_summary es
INNER JOIN deals d ON es.deal_id = d.deal_id
WHERE es.is_eligible = 1
    AND d.deal_status = 'ACTIVE'
GROUP BY es.investment_company_id;


Concentration Testing

CREATE VIEW concentration_limits AS
SELECT 
    es.investment_company_id,
    da.naics_code,
    da.credit_grade,
    SUM(d.funded_amount) AS concentration_amount,
    SUM(d.funded_amount) / SUM(SUM(d.funded_amount)) OVER (PARTITION BY es.investment_company_id) AS concentration_pct,
    MAX(cl.max_concentration_limit) AS limit_threshold
FROM deal_eligibility_summary es
INNER JOIN deals d ON es.deal_id = d.deal_id
INNER JOIN deal_attributes da ON es.deal_id = da.deal_id
LEFT JOIN concentration_limits_config cl 
    ON es.investment_company_id = cl.investment_company_id 
    AND da.naics_code = cl.naics_code
WHERE es.is_eligible = 1
GROUP BY 1,2,3;


📋 Implementation Checklist
Phase 1: Foundation (Weeks 1-2)
	∙	Create configuration tables (Layer 1)
	∙	Migrate existing rules to catalog
	∙	Set up rule_threshold_mappings for 3 current companies
	∙	Create deal_attributes normalized view
Phase 2: Core Engine (Weeks 3-4)
	∙	Build deal_rule_evaluation view
	∙	Test rule evaluation logic for all rule types
	∙	Create deal_eligibility_summary rollup
	∙	Validate against existing views (should match 100%)
Phase 3: Optimization (Week 5)
	∙	Create materialized views
	∙	Set up refresh schedules
	∙	Optimize distribution keys and sort keys
	∙	Performance test with full dataset
Phase 4: Integration (Week 6)
	∙	Build QuickSight datasets
	∙	Create borrowing base integration views
	∙	Set up audit logging
	∙	Create change management procedures
Phase 5: Migration (Week 7)
	∙	Parallel run with old views
	∙	Stakeholder validation
	∙	Cutover to new system
	∙	Deprecate old views

🎯 Key Benefits Summary



|Benefit                 |Old Approach           |New Architecture        |
|------------------------|-----------------------|------------------------|
|**Code Duplication**    |3x hardcoded views     |Single rules engine     |
|**Adding Company**      |Weeks (copy/paste/test)|Hours (INSERT rows)     |
|**Changing Threshold**  |Code deployment        |Config UPDATE           |
|**Auditability**        |Limited                |Full rule-level tracking|
|**Business User Access**|None                   |Can modify thresholds   |
|**Testing**             |Hard to isolate        |Test rules independently|
|**Documentation**       |Tribal knowledge       |Self-documenting catalog|

This architecture gives you a production-ready, enterprise-grade eligibility engine that’s maintainable, auditable, and scales effortlessly. The rules-driven approach means your data team manages data, not code deployments.​​​​​​​​​​​​​​​​
