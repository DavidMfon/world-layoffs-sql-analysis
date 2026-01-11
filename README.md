# world_layoffs_sql_analysis
---

### Project Overview
---

This project focuses on building a structured SQL workflow for cleaning and exploring the World Layoffs dataset. The objective is to prepare high quality, reliable data that can be used for exploratory data analysis (EDA) and future visualization.
The project covers database creation, raw data staging, systematic data cleaning, and exploratory analysis to uncover trends in layoffs across companies, industries, countries, and time.

### Data Sources 
---

World Layoffs Dataset
[Download here](https://drive.google.com/drive/folders/1-9vxtwceLcslbfu7xfRjMiV0fbeBbJy8) 

### Tools
---

- **SQL (MySQL)**: Used for data cleaning, transformation, and exploratory data analysis

### Data Cleaning
---

In the initial data preparation phase, the following steps were performed:

**1. Data Loading and Inspection**

- The dataset was imported into SQL.

- A staging table (layoffs_staging) was created to preserve the raw data.

- This ensured that the original dataset remained intact in case of errors during cleaning.

**2. Removing Duplicates**

Duplicates were identified using window functions.


**Step 1: Assign Row Numbers**

```sql
SELECT *,
ROW_NUMBER() OVER (
PARTITION BY company, industry, total_laid_off, percentage_laid_off, `date`
) AS row_num
FROM layoffs_staging;
Step 2: Identify Duplicate Records
WITH duplicate_cte AS (
    SELECT *,
    ROW_NUMBER() OVER (
        PARTITION BY company, industry, total_laid_off, percentage_laid_off, `date`
    ) AS row_num
    FROM layoffs_staging
)
SELECT *
FROM duplicate_cte
WHERE row_num > 1;
```

**Step 3: Create a Second Staging Table**

- A new table layoffs_staging2 was created.

- Cleaned records were inserted after manually verifying duplicates.

**Step 4: Delete Duplicates**

```sql
DELETE
FROM layoffs_staging2
WHERE row_num > 1;
```

**3. Standardization of Data**

This step involved fixing inconsistencies and formatting issues.

**a. Removing Extra Spaces and Standardizing Text**

```sql
SELECT DISTINCT country, TRIM(TRAILING '. ' FROM country)
FROM layoffs_staging2
ORDER BY 1;
Code 2:
UPDATE layoffs_staging2
SET country = TRIM(TRAILING '. ' FROM country)
WHERE country LIKE 'United States%';
```

**b. Converting Date Column**

The date column was converted from text to a proper date format.

```sql
ALTER TABLE layoffs_staging2
MODIFY COLUMN `date` DATE;
```

**4. Handling Null and Blank Values**

Null and blank values were identified and handled appropriately.

**i. Identifying Null Values**

```sql
SELECT *
FROM layoffs_staging2
WHERE total_laid_off IS NULL
AND percentage_laid_off IS NULL;
```

**ii. Replacing Blank Industry Values**

```sql
UPDATE layoffs_staging2
SET industry = NULL
WHERE industry = ' ';
```

**iii. Populating Missing Industry Values Using Self Joints**


```sql
SELECT t1.industry, t2.industry
FROM layoffs_staging2 t1
JOIN layoffs_staging2 t2
ON t1.company = t2.company
WHERE (t1.industry IS NULL OR t1.industry = ' ')
AND t2.industry IS NOT NULL;
```

**5. Removing Columns and Rows**

**i. Rows with no meaningful layoff data were removed.**

```sql
DELETE
FROM layoffs_staging2
WHERE total_laid_off IS NULL
AND percentage_laid_off IS NULL;
ii. The helper column row_num was also dropped.
ALTER TABLE layoffs_staging2
DROP COLUMN row_num;
```
---

## Exploratory Data Analysis
---

After cleaning, exploratory analysis was conducted to understand trends and patterns.

**1. Maximum Layoffs**

```sql
SELECT MAX(total_laid_off), MAX(percentage_laid_off)
FROM layoffs_staging2;
```

**2. Total Layoffs by Company**

```sql
SELECT company, SUM(total_laid_off) AS total_layoffs
FROM layoffs_staging2
GROUP BY company
ORDER BY total_layoffs DESC;
```

**3. Layoffs by Month**

```sql
SELECT SUBSTRING(`date`, 1, 7) AS `month`,
SUM(total_laid_off) AS total_layoffs
FROM layoffs_staging2
WHERE SUBSTRING(`date`, 1, 7) IS NOT NULL
GROUP BY `month`
ORDER BY `month` ASC;
```

**4. Rolling Total of Layoffs**

```sql
WITH Rolling_Total AS (
    SELECT SUBSTRING(`date`, 1, 7) AS `month`,
    SUM(total_laid_off) AS total_layoffs
    FROM layoffs_staging2
    GROUP BY `month`
)
SELECT `month`,
total_layoffs,
SUM(total_layoffs) OVER (ORDER BY `month`) AS rolling_total
FROM Rolling_Total;
```

**5. Layoffs by Company and Year**

```sql
SELECT company, YEAR(`date`) AS year, SUM(total_laid_off) AS total_layoffs
FROM layoffs_staging2
GROUP BY company, year
ORDER BY total_layoffs DESC;
```

**6. Ranking Companies by Layoffs (Dense Ranking)**

```sql
WITH Company_Year AS (
    SELECT company, YEAR(`date`) AS years,
    SUM(total_laid_off) AS total_laid_off
    FROM layoffs_staging2
    GROUP BY company, years
),
Company_Year_Rank AS (
    SELECT *,
    RANK() OVER (PARTITION BY years ORDER BY total_laid_off DESC) AS ranking
    FROM Company_Year
    WHERE years IS NOT NULL
)
SELECT *
FROM Company_Year_Rank
WHERE ranking <= 5;
```

### Data Analysis
---

*Key analytical techniques used in this project include:*

- Window functions (ROW_NUMBER, RANK, SUM OVER)

- Common Table Expressions (CTEs)

- Date aggregation and rolling totals

- Company, industry, country, and time based analysis

### Results and Findings

- Certain companies experienced significantly higher layoffs compared to others.

- Layoffs varied widely across industries, with tech related industries being heavily affected.

- Layoff trends showed clear spikes during specific periods.

- Rolling totals revealed cumulative workforce reductions over time.

- Some years recorded concentrated layoffs among a small number of companies.

Recommendations

- Companies should improve workforce planning and risk assessment to reduce sudden mass layoffs.

- Industry level monitoring can help anticipate downturns earlier.

- Diversification of revenue streams may reduce dependency on volatile market conditions.

- Policymakers and organizations can use these insights to develop better employment stabilization strategies.
