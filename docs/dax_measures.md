# DAX Measures

Model: `powerbi/PartD_Spending.pbix`. All measures live in the `_Measures` table.
Values below are from my model with Base Year = 2020, Current Year = 2024, Substitution Rate = 50%.

## Helper tables (Modeling > New table)

~~~dax
Base Year = SELECTCOLUMNS(dim_year, "Base Year", dim_year[Year])

Current Year = SELECTCOLUMNS(dim_year, "Current Year", dim_year[Year])

Bridge Step = DATATABLE(
    "Step", STRING, "Sort", INTEGER,
    { {"Price per unit", 1}, {"Volume (units)", 2}, {"New drugs", 3},
      {"Drugs no longer used", 4}, {"Other", 5} })
~~~

`Substitution Rate`: what-if parameter (Modeling > New parameter), 0% to 100%, step 5%, default 50%.

## 1 Core

| Measure | What it shows | My value |
|---|---|---|
| Total Spend | Gross Part D spending (2024) | $288.7 B |
| Total Claims | Prescription fills (2024) | $1.71 B |
| Spend per Claim | Average cost per fill (2024) | $168.65 |
| Spend YoY % | Growth vs. prior year (2024) | 4.66% |

~~~dax
Base Yr =
SELECTEDVALUE('Base Year'[Base Year],
    CALCULATE(MIN(dim_year[Year]), REMOVEFILTERS(dim_year)))

Current Yr =
SELECTEDVALUE('Current Year'[Current Year],
    CALCULATE(MAX(dim_year[Year]), REMOVEFILTERS(dim_year)))

Total Spend = SUM(fact_drug_year[Spending])

Total Claims = SUM(fact_drug_year[Claims])

Total Units = SUM(fact_drug_year[Dosage Units])

Spend per Claim = DIVIDE([Total Spend], [Total Claims])

Spend YoY % =
VAR yr = MAX(dim_year[Year])
VAR prev = CALCULATE([Total Spend], dim_year[Year] = yr - 1)
RETURN IF(NOT ISBLANK(prev), DIVIDE([Total Spend] - prev, prev))
~~~

## 2 Base vs Current

| Measure | What it shows | My value |
|---|---|---|
| Spend (Base) | 2020 spending | $194.1 B |
| Spend (Current) | 2024 spending | $288.7 B |
| Spend Change | 2024 minus 2020 | $94.58 B |
| Spend CAGR | Average yearly growth | 10.43% |
| Claims Change % | Growth in fills | 17.69% |
| Unit Price Change % | Growth in spending per unit | 25.62% |

~~~dax
Spend (Base) = VAR b = [Base Yr] RETURN CALCULATE([Total Spend], dim_year[Year] = b)

Spend (Current) = VAR c = [Current Yr] RETURN CALCULATE([Total Spend], dim_year[Year] = c)

Claims (Base) = VAR b = [Base Yr] RETURN CALCULATE([Total Claims], dim_year[Year] = b)

Claims (Current) = VAR c = [Current Yr] RETURN CALCULATE([Total Claims], dim_year[Year] = c)

Units (Base) = VAR b = [Base Yr] RETURN CALCULATE([Total Units], dim_year[Year] = b)

Units (Current) = VAR c = [Current Yr] RETURN CALCULATE([Total Units], dim_year[Year] = c)

Spend Change = [Spend (Current)] - [Spend (Base)]

Spend Change % = DIVIDE([Spend Change], [Spend (Base)])

Spend CAGR =
VAR b = [Base Yr]
VAR c = [Current Yr]
VAR s0 = [Spend (Base)]
VAR s1 = [Spend (Current)]
RETURN IF(c > b && s0 > 0 && s1 > 0, (s1 / s0) ^ (1 / (c - b)) - 1)

Claims Change % = DIVIDE([Claims (Current)] - [Claims (Base)], [Claims (Base)])

Unit Price (Base) = DIVIDE([Spend (Base)], [Units (Base)])

Unit Price (Current) = DIVIDE([Spend (Current)], [Units (Current)])

Unit Price Change % = DIVIDE([Unit Price (Current)] - [Unit Price (Base)], [Unit Price (Base)])

Spend per Claim (Current) = DIVIDE([Spend (Current)], [Claims (Current)])
~~~

## 3 Drivers (price vs. volume)

Change in spend = (units change x base price) + (price change x current units), per drug.

| Measure | What it shows | My value |
|---|---|---|
| Price Effect | Change from higher or lower spending per unit | - $498.1 M |
| Volume Effect | Change from more or fewer units | $63.73 B |
| New Drug Effect | Drugs with spending in 2024 but not 2020 | $31.34 B |
| Exited Drug Effect | Drugs with spending in 2020 but not 2024 | $0 |
| Other Effect | Leftover (missing units) | $213.6 K (<0.01% of change) |

~~~dax
Price Effect =
VAR b = [Base Yr]
VAR c = [Current Yr]
RETURN
SUMX(
    VALUES(dim_drug[DrugKey]),
    VAR s0 = CALCULATE(SUM(fact_drug_year[Spending]), dim_year[Year] = b)
    VAR u0 = CALCULATE(SUM(fact_drug_year[Dosage Units]), dim_year[Year] = b)
    VAR s1 = CALCULATE(SUM(fact_drug_year[Spending]), dim_year[Year] = c)
    VAR u1 = CALCULATE(SUM(fact_drug_year[Dosage Units]), dim_year[Year] = c)
    RETURN IF(s0 > 0 && s1 > 0 && u0 > 0 && u1 > 0,
        (DIVIDE(s1, u1) - DIVIDE(s0, u0)) * u1)
)

Volume Effect =
VAR b = [Base Yr]
VAR c = [Current Yr]
RETURN
SUMX(
    VALUES(dim_drug[DrugKey]),
    VAR s0 = CALCULATE(SUM(fact_drug_year[Spending]), dim_year[Year] = b)
    VAR u0 = CALCULATE(SUM(fact_drug_year[Dosage Units]), dim_year[Year] = b)
    VAR s1 = CALCULATE(SUM(fact_drug_year[Spending]), dim_year[Year] = c)
    VAR u1 = CALCULATE(SUM(fact_drug_year[Dosage Units]), dim_year[Year] = c)
    RETURN IF(s0 > 0 && s1 > 0 && u0 > 0 && u1 > 0,
        (u1 - u0) * DIVIDE(s0, u0))
)

New Drug Effect =
VAR b = [Base Yr]
VAR c = [Current Yr]
RETURN
SUMX(
    VALUES(dim_drug[DrugKey]),
    VAR s0 = CALCULATE(SUM(fact_drug_year[Spending]), dim_year[Year] = b)
    VAR s1 = CALCULATE(SUM(fact_drug_year[Spending]), dim_year[Year] = c)
    RETURN IF(s0 = 0 && s1 > 0, s1)
)

Exited Drug Effect =
VAR b = [Base Yr]
VAR c = [Current Yr]
RETURN
SUMX(
    VALUES(dim_drug[DrugKey]),
    VAR s0 = CALCULATE(SUM(fact_drug_year[Spending]), dim_year[Year] = b)
    VAR s1 = CALCULATE(SUM(fact_drug_year[Spending]), dim_year[Year] = c)
    RETURN IF(s0 > 0 && s1 = 0, -s0)
)

Other Effect =
[Spend Change] - [Price Effect] - [Volume Effect] - [New Drug Effect] - [Exited Drug Effect]

Other % of Change = DIVIDE([Other Effect], [Spend Change])

Bridge Value =
SWITCH(SELECTEDVALUE('Bridge Step'[Step]),
    "Price per unit", [Price Effect],
    "Volume (units)", [Volume Effect],
    "New drugs", [New Drug Effect],
    "Drugs no longer used", [Exited Drug Effect],
    "Other", [Other Effect])

Bridge Title =
"What drove the " & FORMAT([Spend Change] / 1e9, "$#,##0.0") & "B change in spending, "
    & [Base Yr] & "–" & [Current Yr]
~~~

## 4 Generic and Savings

| Measure | What it shows | My value |
|---|---|---|
| Generic Fill Rate (Current) | Share of 2024 claims that are generic | 87.39% |
| Brand Share of Spend (Current) | Share of 2024 spending on brands | 87.30% |
| Savings at 100% Substitution | If every multi-source brand unit were generic | $16.44 B |
| Substitution Savings | At a 50% switch rate | $8.219 B |
| Negotiated Share of Spend | Spending on Medicare-negotiated drugs | 35.88% |
| Outlier Drugs (Current) | Drugs CMS flagged as outliers in 2024 | 395 |

~~~dax
Claims Share (Current) =
DIVIDE([Claims (Current)],
    CALCULATE([Claims (Current)], REMOVEFILTERS(dim_drug[Drug Type], dim_drug[Source Type])))

Spend Share (Current) =
DIVIDE([Spend (Current)],
    CALCULATE([Spend (Current)], REMOVEFILTERS(dim_drug[Drug Type], dim_drug[Source Type])))

Generic Fill Rate (Current) =
DIVIDE(CALCULATE([Claims (Current)], dim_drug[Drug Type] = "Generic"), [Claims (Current)])

Brand Share of Spend (Current) =
DIVIDE(CALCULATE([Spend (Current)], dim_drug[Drug Type] = "Brand"), [Spend (Current)])

Savings at 100% Substitution =
VAR c = [Current Yr]
RETURN
SUMX(
    CALCULATETABLE(
        VALUES(dim_drug[DrugKey]),
        KEEPFILTERS(dim_drug[Source Type] = "Multi-source brand"),
        KEEPFILTERS(dim_drug[Exclude From Savings] = FALSE())
    ),
    VAR molecule = CALCULATE(SELECTEDVALUE(dim_drug[Generic Key]))
    VAR brandSpend = CALCULATE(SUM(fact_drug_year[Spending]), dim_year[Year] = c)
    VAR brandUnits = CALCULATE(SUM(fact_drug_year[Dosage Units]), dim_year[Year] = c)
    VAR genericPrice =
        CALCULATE(
            DIVIDE(SUM(fact_drug_year[Spending]), SUM(fact_drug_year[Dosage Units])),
            REMOVEFILTERS(dim_drug),
            dim_drug[Generic Key] = molecule,
            dim_drug[Drug Type] = "Generic",
            dim_year[Year] = c
        )
    VAR brandPrice = DIVIDE(brandSpend, brandUnits)
    RETURN IF(genericPrice > 0 && brandPrice > genericPrice,
        (brandPrice - genericPrice) * brandUnits)
)

Substitution Savings = [Savings at 100% Substitution] * [Substitution Rate Value]

Brand-to-Generic Price Ratio =
VAR c = [Current Yr]
VAR molecule = SELECTEDVALUE(dim_drug[Generic Key])
VAR genericPrice =
    CALCULATE(
        DIVIDE(SUM(fact_drug_year[Spending]), SUM(fact_drug_year[Dosage Units])),
        REMOVEFILTERS(dim_drug),
        dim_drug[Generic Key] = molecule,
        dim_drug[Drug Type] = "Generic",
        dim_year[Year] = c
    )
RETURN IF(SELECTEDVALUE(dim_drug[Source Type]) = "Multi-source brand",
    DIVIDE([Unit Price (Current)], genericPrice))

Negotiated Drug Spend (Current) =
CALCULATE([Spend (Current)], dim_drug[Negotiation Round] <> "Not selected")

Negotiated Share of Spend = DIVIDE([Negotiated Drug Spend (Current)], [Spend (Current)])

Outlier Drugs (Current) =
VAR c = [Current Yr]
RETURN CALCULATE(COUNTROWS(fact_drug_year), fact_drug_year[Outlier Flag] = 1, dim_year[Year] = c)
~~~

## 5 State (2024 only)

| Measure | What it shows | My value |
|---|---|---|
| US Generic Fill Rate | 50 states + DC | 87.38% |
| Highest Cost per 30-Day Fill | State and value | DC ($154.17) |
| Lowest State Generic Fill Rate | State and value | DC (84.72%) |
| Highest State Generic Fill Rate | State and value | Minnesota (89.19%) |

~~~dax
State Drug Cost = SUM(fact_state_drug[Drug Cost])

State Claims = SUM(fact_state_drug[Claims])

State 30-Day Fills = SUM(fact_state_drug[30-Day Fills])

Cost per 30-Day Fill = DIVIDE([State Drug Cost], [State 30-Day Fills])

State Generic Fill Rate =
DIVIDE(CALCULATE([State Claims], dim_drug[Drug Type] = "Generic"), [State Claims])

US Generic Fill Rate =
CALCULATE([State Generic Fill Rate], REMOVEFILTERS(dim_state), dim_state[Area Type] = "State or DC")

Generic Fill Gap vs US (pts) = ([State Generic Fill Rate] - [US Generic Fill Rate]) * 100

Opioid Share of Claims =
DIVIDE(CALCULATE([State Claims], dim_drug[Opioid] = "Y"), [State Claims])
~~~

## Checks

- Control total: Total Spend (2024) matches Excel SUMIFS: Excel $288.7B / Power BI $288.7B
- Other % of Change is small: 0%