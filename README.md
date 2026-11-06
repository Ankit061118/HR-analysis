# HR Employee Attrition Analysis

An HR analytics project exploring employee attrition with SQL and a Power BI dashboard. The analysis compares attrition across departments and age groups, summarizes exit reasons, calculates salary compa-ratios, and flags current employees with a simple risk score.

## Dashboard

![HR Employee Analysis Dashboard](<Screenshot 2026-11-06 193930.png>)

The dashboard screenshot shows headline cards for total employees, employees who left, attrition rate, average salary, and attrition cost, along with breakdowns by department, age band, tenure, and gender. The displayed cards report approximately 3K employees, 481 leavers, 16.03% attrition, and 5.78K average salary. The dashboard also displays 16M attrition cost; its calculation is not documented in the supplied dataset columns.

## Project files

- `final1.csv` — 3,000 employee records used for the analysis.
- `HR_Attrition_Dataset.xlsx` — Excel dataset.
- `hr_attrition_analysis.pbix` — Power BI report.
- `Screenshot 2026-11-06 193930.png` — dashboard screenshot embedded above.
- `Untitled.ipynb` — analysis notebook.

## Dataset overview

The CSV includes employee demographics, department, job level, salary, join and exit dates, employee satisfaction, work arrangements, attrition status, and (for leavers) exit reason. It also contains derived fields such as `Age_Band`, `CompRatio`, and `RiskScore`.

## Insights from `final1.csv`

- **Overall:** 481 of 3,000 employees left, an attrition rate of **16.03%**.
- **Department:** Customer Support has the highest department attrition rate (**21.83%**, 100 of 458), followed by Sales (**20.09%**, 130 of 647). Finance is lowest (**9.21%**, 22 of 239).
- **Age:** The **18–25** group has the highest attrition rate (**29.91%**, 35 of 117). The largest number of leavers is in the **26–35** group (274), which is also the largest cohort.
- **Exit reasons:** Work-Life Balance is the most frequently recorded reason (133 leavers), followed by Management / Culture (99) and Better Compensation (81).
- **Early departures:** 218 leavers exited within 730 days of joining (about **45.3%** of all leavers with usable dates).
- **Interpretation:** Prioritize retention discovery in Customer Support and Sales, investigate early-tenure experience and work-life balance, and treat these aggregate patterns as indicators for follow-up—not proof of cause.

## SQL analysis

The examples below use MySQL syntax. Create/select a database named `e`, load `final1.csv` into a table named `final1`, and ensure `JoinDate` and `ExitDate` are date-compatible columns. If your database or table has a different name, update it accordingly.

```sql
USE e;

SELECT *
FROM final1;

-- Leavers and attrition rate by department
SELECT
    Department,
    SUM(CASE WHEN Attrition = 'Yes' THEN 1 ELSE 0 END) AS Left_Count,
    ROUND(
        SUM(CASE WHEN Attrition = 'Yes' THEN 1 ELSE 0 END) / COUNT(*) * 100,
        2
    ) AS Attrition_Pct
FROM final1
GROUP BY Department
ORDER BY Attrition_Pct DESC;

-- Leavers by age band
SELECT
    CASE
        WHEN Age <= 25 THEN '18-25'
        WHEN Age <= 35 THEN '26-35'
        WHEN Age <= 45 THEN '36-45'
        WHEN Age <= 55 THEN '46-55'
        ELSE '56-65'
    END AS Age_Band,
    COUNT(*) AS Total,
    SUM(CASE WHEN Attrition = 'Yes' THEN 1 ELSE 0 END) AS Left_Count
FROM final1
GROUP BY Age_Band
ORDER BY MIN(Age);

-- Exit reasons by department
SELECT
    ExitReason,
    Department,
    COUNT(*) AS Leavers
FROM final1
WHERE Attrition = 'Yes'
GROUP BY ExitReason, Department
ORDER BY Leavers DESC;

-- Exit reasons across the organization
SELECT
    Department,
    ExitReason,
    COUNT(*) AS Left_Count
FROM final1
WHERE Attrition = 'Yes'
GROUP BY Department, ExitReason
ORDER BY Left_Count DESC;

-- Compa-ratio against the department and job-level salary average
SELECT
    EmployeeID,
    Department,
    JobLevel,
    MonthlySalary,
    ROUND(
        MonthlySalary / AVG(MonthlySalary) OVER (
            PARTITION BY Department, JobLevel
        ),
        2
    ) AS CompRatio
FROM final1;

-- Leavers whose time between joining and exiting was at most 730 days
SELECT COUNT(*) AS Early_Leavers
FROM final1
WHERE Attrition = 'Yes'
  AND JoinDate IS NOT NULL
  AND ExitDate IS NOT NULL
  AND DATEDIFF(ExitDate, JoinDate) <= 730;

-- Heuristic risk score for current employees; return the ten highest scores
WITH scored AS (
    SELECT
        EmployeeID,
        Department,
        JobLevel,
        MonthlySalary,
        Age,
        LastSalaryHikePct,
        MonthlySalary / AVG(MonthlySalary) OVER (
            PARTITION BY Department, JobLevel
        ) AS CompRatio
    FROM final1
    WHERE Attrition = 'No'
)
SELECT
    EmployeeID,
    Department,
    JobLevel,
    ROUND(CompRatio, 2) AS CompRatio,
    (
        CASE WHEN CompRatio < 0.9 THEN 2 ELSE 0 END
        + CASE WHEN Age <= 25 THEN 1 ELSE 0 END
        + CASE WHEN LastSalaryHikePct <= 4 THEN 1 ELSE 0 END
    ) AS RiskScore
FROM scored
ORDER BY RiskScore DESC
LIMIT 10;
```

The risk score is a simple prioritization heuristic based only on the three conditions shown; it is not a validated prediction of whether an employee will leave. The `DATEDIFF` expression uses MySQL's `DATEDIFF(end_date, start_date)` convention.

## Running the analysis

1. Import `final1.csv` into MySQL as `final1`, preserving date columns as dates where possible.
2. Select the target database with `USE e;` (or replace `e` with your database name).
3. Run the SQL statements above.
4. Open `hr_attrition_analysis.pbix` in Power BI Desktop to explore the report. If the report prompts for a data source, point it to the local dataset.
