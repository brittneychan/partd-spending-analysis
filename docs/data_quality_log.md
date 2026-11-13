# Data Quality Log

| # | Check | Result | Action taken |
|---|---|---|---|
| 1 | Spending file: total rows vs. "Overall" rows | 14,536 total, 3,625 Overall | Kept Overall to avoid double counting |
| 2 | Drug match between files (dim_drug "Found In") | Both 3,625 / Spending only 0 / Geo only 7 | Page 4 uses geography file only |
| 3 | Beneficiaries are unique PER DRUG | n/a | Never summed across drugs; use claims and fills instead |
| 4 | Control total: 2024 spending, Excel vs. Power BI | Excel $288,667,842,058 / Power BI $288,667,842,058.44 | Matched within rounding |

# Tables in Power BI Model
| Table | Type | One row = | Key column(s) | Main columns |
|---|---|---|---|---|
| fact_drug_year | Fact | one drug in one year (2020-2024) | DrugKey, Year | Spending, Dosage Units, Claims, Beneficiaries, Outlier Flag |
| fact_state_drug | Fact | one drug in one state (2024) | DrugKey, State | Drug Cost, Claims, 30-Day Fills, Prescribers, Beneficiaries |
| dim_drug | Dimension | one drug | DrugKey | Drug Label, Brand Name, Generic Name, Drug Type, Source Type, Negotiation Round, Opioid, Antibiotic, Antipsychotic, Found In |
| dim_year | Dimension | one year | Year | Year |
| dim_state | Dimension | one state or territory | State | Area Type |
