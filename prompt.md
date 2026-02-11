MCA Eligibility Engine Requirements Prompt
Business Context
We currently operate three separate eligibility views, one per investment company, that determine MCA (Merchant Cash Advance) deal eligibility by assigning binary pass/fail flags (1/0) across multiple eligibility checks including time in business, FICO score, credit grade, industry classification (NAICS codes), and deal term length.
Core Problem Statement
The existing architecture has duplicated logic across investment companies, hardcoded thresholds, and no centralized rules management. We need to replace this with a single, scalable, enterprise-grade eligibility engine.
Primary Requirements
1. Single Source of Truth
	∙	Eliminate duplicated logic across investment companies
	∙	Centralize all eligibility rule definitions and evaluations
	∙	Support reusable rule components across different configurations
2. Rules-Driven Configuration (Data, Not Code)
	∙	All eligibility rules must be data-configurable, not hardcoded in SQL or application logic
	∙	Business users should be able to modify thresholds and allowed values without code deployments
	∙	Support versioning and effective dating of rule changes
3. Investment Company & Tranche Hierarchy
	∙	Critical: Investment companies can have multiple “tranches” (e.g., Senior Tranche, Mezzanine Tranche, Junior Tranche)
	∙	Each tranche may have:
	∙	Completely different eligibility rules from other tranches
	∙	The same rules but different thresholds
	∙	Overlapping rules with some shared and some unique criteria
	∙	The system must support the hierarchy: Investment Company → Tranche → Rule Configuration
4. Complex Rule Logic Support
The engine must support sophisticated multi-condition rules such as:
Example Rule: “Minimum FICO ≥ 680 AND Time in Business ≥ 24 months for deals where NAICS3 industry code is in [‘722’, ‘811’, ‘531’] AND credit grade is in [‘A’, ‘A+’, ‘AA’]”
This requires:
	∙	Composite conditions (multiple attributes evaluated together)
	∙	Conditional eligibility (rules that only apply when certain criteria are met)
	∙	Industry-specific thresholds (different requirements for different NAICS codes or groupings)
	∙	Credit-grade tiering (varying rules based on credit quality)
	∙	Nested logic (AND/OR combinations of multiple conditions)
5. Rule-Level Auditability
	∙	Track which specific rules passed or failed for each deal
	∙	Capture actual values vs. threshold values at evaluation time
	∙	Maintain complete audit trail of:
	∙	When rules were evaluated
	∙	What the thresholds were at the time
	∙	Why a deal passed or failed
	∙	Historical changes to rule configurations
6. Aggregated Eligibility Determination
	∙	Produce a final eligibility flag (pass/fail) per deal per tranche
	∙	Support both:
	∙	Required rules (hard stops - must pass)
	∙	Weighted rules (contribute to scoring - may be advisory)
	∙	Enable composite scoring for risk ranking even among eligible deals
7. Downstream Integration
Must integrate cleanly with:
	∙	Borrowing base calculations (sum of eligible deal balances)
	∙	Concentration testing (limits by industry, geography, credit grade)
	∙	Portfolio reporting and analytics
	∙	Deal origination systems (real-time eligibility checks)
8. Technology Stack Requirements
	∙	Platform: Amazon Redshift (data warehouse)
	∙	BI Tool: Amazon QuickSight (must support interactive dashboards)
	∙	Performance: Handle thousands of deals evaluated against dozens of rules efficiently
	∙	Scalability: Support growth from 3 to N investment companies and multiple tranches per company
Additional Functional Requirements
Rule Types to Support
	1.	Simple threshold rules
	∙	Minimum/maximum numeric values (FICO ≥ 650, Term ≤ 24 months)
	∙	Date-based rules (business established before/after date)
	2.	List-based rules
	∙	Allowed values (credit grades in [‘A’, ‘B’, ‘C’])
	∙	Excluded values (restricted NAICS codes, state exclusions)
	3.	Range rules
	∙	Values within acceptable range (factor rate between 1.15 and 1.45)
	4.	Composite/conditional rules
	∙	Multi-attribute conditions with AND/OR logic
	∙	Context-dependent thresholds (e.g., different FICO minimums for different industries)
	∙	Rules that only apply when certain conditions are met
	5.	Derived metric rules
	∙	Calculated fields (debt-to-income ratio, utilization percentage)
	∙	Cross-attribute validations
Tranche-Specific Scenarios
Example Tranche Structure:

InvestCo Alpha
├── Senior Tranche (conservative, highest priority)
│   ├── FICO ≥ 700
│   ├── Time in Business ≥ 36 months
│   ├── Credit Grade in ['AA', 'A+']
│   └── Max Term: 12 months
│
├── Mezzanine Tranche (moderate risk)
│   ├── FICO ≥ 660
│   ├── Time in Business ≥ 24 months
│   ├── Credit Grade in ['A', 'B+', 'B']
│   └── Max Term: 18 months
│
└── Equity Tranche (higher risk tolerance)
    ├── FICO ≥ 620
    ├── Time in Business ≥ 12 months
    ├── Credit Grade in ['B', 'C+', 'C']
    └── Max Term: 24 months


Industry-Specific Rule Example
Rule: Restaurant (NAICS3 = ‘722’) deals in Credit Grade ‘A’ or higher must have:
	∙	FICO ≥ 680 AND Time in Business ≥ 24 months
But this same tranche may have different requirements for other industries:
	∙	Construction (NAICS3 = ‘236’): FICO ≥ 700 AND Time in Business ≥ 36 months
	∙	Retail (NAICS3 = ‘44X’): FICO ≥ 660 AND Time in Business ≥ 18 months
Non-Functional Requirements
Performance
	∙	Eligibility evaluation for 10,000+ deals should complete within acceptable timeframes for reporting
	∙	Support real-time or near-real-time eligibility checks for deal origination
	∙	Materialized view refresh strategies for batch processing
Maintainability
	∙	Schema should be self-documenting
	∙	Rule configurations should be human-readable
	∙	Changes should be non-breaking (backward compatible)
Data Quality
	∙	Handle NULL values gracefully
	∙	Support data validation at rule configuration time
	∙	Flag incomplete deal records that cannot be evaluated
Security & Compliance
	∙	Track who changed what and when (audit logging)
	∙	Support point-in-time reconstruction of eligibility determinations
	∙	Maintain historical snapshots for regulatory reporting
Desired Deliverables
Please design a comprehensive architecture that includes:
	1.	Complete table/view layer structure
	∙	Configuration tables (rules catalog, thresholds, tranche mappings)
	∙	Data transformation layers
	∙	Evaluation engine
	∙	Aggregation/rollup views
	2.	Detailed schemas
	∙	Table definitions with appropriate data types
	∙	Primary/foreign key relationships
	∙	Grain specifications (one row per what?)
	∙	Distribution and sort key recommendations for Redshift
	3.	Rule evaluation methodology
	∙	How to evaluate simple rules efficiently
	∙	How to handle complex composite/conditional rules
	∙	Pattern for extensibility (adding new rule types)
	∙	Approach for NAICS3-level industry groupings with grade-specific thresholds
	4.	Tranche hierarchy implementation
	∙	How to model investment company → tranche → rule relationships
	∙	How deals get assigned to tranches
	∙	How to handle tranches with overlapping but different rules
	5.	Aggregation strategy
	∙	How to roll up rule results to final pass/fail
	∙	How to handle required vs. weighted rules
	∙	How to produce meaningful eligibility scores
	6.	Scalability approach
	∙	How the architecture handles growth from 3 to 50+ investment companies
	∙	How to add new tranches without code changes
	∙	How to add new rule types as business evolves
	7.	Performance optimization
	∙	Redshift-specific best practices (distribution, sorting, compression)
	∙	Materialized view strategies
	∙	Query patterns for QuickSight dashboards
	8.	Auditability features
	∙	Audit trail design
	∙	Point-in-time reconstruction capability
	∙	Change management workflows
	9.	Integration patterns
	∙	How downstream borrowing base calculations consume eligibility results
	∙	How concentration limits leverage the engine
	∙	APIs or views for deal origination systems
	10.	Implementation roadmap
	∙	Phased approach to migration from current state
	∙	Parallel run strategy
	∙	Validation methodology
	∙	Rollback considerations
Success Criteria
The final architecture should:
	∙	✅ Eliminate all duplicated eligibility logic
	∙	✅ Support adding new investment companies and tranches with zero code changes
	∙	✅ Enable business users to modify thresholds through data updates only
	∙	✅ Support complex multi-condition rules (FICO + TIB for specific NAICS3 + Grade combinations)
	∙	✅ Provide complete rule-level audit trails
	∙	✅ Integrate seamlessly with Redshift and QuickSight
	∙	✅ Scale to 20+ investment companies with 2-5 tranches each
	∙	✅ Process evaluations for 50,000+ deals efficiently
	∙	✅ Be maintainable by data analysts, not just engineers​​​​​​​​​​​​​​​​
