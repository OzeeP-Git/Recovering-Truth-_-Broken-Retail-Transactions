# Recovering Truth from Broken Retail Transaction Data

## Project Overview

This project addresses a critical real-world data science challenge: recovering and analyzing highly corrupted retail transaction data resulting from a failed ERP migration. The primary objective was to determine if any meaningful analytical insights or machine learning predictions could be derived from this fragmented, inconsistent, and logically unreliable data. Instead of applying naive cleaning, a trust-aware data recovery pipeline was designed to align heterogeneous schemas, convert corrupted fields, detect logical contradictions, classify data reliability, and preserve inherent uncertainty.

## Data Description

Three datasets were provided, each exhibiting severe quality issues:

1.  `transactions_dump_partA.csv`: Transactional logs from POS system A.
2.  `transactions_dump_partB.csv`: Transactional logs from POS system B, with schema mismatches compared to Part A.
3.  `store_master_corrupt.csv`: Store metadata containing textual corruption and inconsistencies.

**Initial Data Challenges:**
*   Mixed column names and inconsistent representations across transaction files.
*   Numeric fields stored as text (e.g., "ten", "free", "₹124.29").
*   Dates in multiple formats, including many "invalid_date" entries.
*   Logical inconsistencies: negative quantities, discounts exceeding unit prices.
*   Significant missing values (40-60%) in key columns.
*   Duplicate invoices and noisy categorical spellings.

## Methodology

### 1. Intelligent Data Ingestion & Schema Alignment

*   **Schema Alignment**: The two transaction files were aligned into a common, canonical schema.
*   **Custom Parsers**: Robust custom functions were developed to handle various data types and inconsistencies:
    *   `parse_price()`: Removed currency symbols (₹/$) and converted "free" to `0`.
    *   `parse_quantity()`: Converted word-based numbers (e.g., "ten") to integers.
    *   `clean_category()`: Standardized noisy product category labels (e.g., "grocry" to "grocery").
    *   `clean_channel()`: Normalized sales channel values (e.g., "online" to "online").
    *   **Robust Date Parsing**: Handled multiple date formats with error coercion.
*   **Store Master Handling**: The `store_master_corrupt.csv` file was treated as auxiliary metadata, cleaning textual years and standardizing city names.

### 2. Tier-Based Data Validation (Trust Model)

To manage uncertainty without outright deletion, a three-tier trust model was implemented:

*   **Tier-1 (Clean)**: Rows where all core fields were valid and logically consistent, fully usable for analysis.
*   **Tier-2 (Suspicious)**: Rows that were incomplete (missing key values) but contained no logical contradictions. Usable for aggregate reporting but with caution.
*   **Tier-3 (Impossible)**: Rows violating fundamental business logic (e.g., negative quantity, price, or discount > price), deemed unusable.

**Results of Tiering:**

| Dataset | Tier-1 (Clean) | Tier-2 (Suspicious) | Tier-3 (Impossible) |
| :------ | :------------- | :------------------ | :------------------ |
| Part A  | ~14.7k         | ~14.8k              | ~170k               |
| Part B  | ~17.7k         | ~15.5k              | ~166k               |

*Insight: Only approximately 15% of the raw transaction logs were found to be truly reliable, a significant finding in itself.*

### 3. Conditional Data Recovery

*   **Discount Imputation**: With 66% missing values, naive imputation was avoided. Discount patterns were learned *only* from Tier-1 data (grouped by `product_category` and `month` using statistical mode) and then used to fill missing values *only* in Tier-2 rows. This recovered 5,574 values, reducing missingness to ~44% while preserving data honesty.
*   **Store ID Assumption & Correction**: An initial assumption to normalize store IDs (e.g., 'store_1', 'S-001', 'Store-01' to a single ID) was later revised. Further inspection revealed these were distinct stores. The pipeline was corrected by reverting normalization and re-merging metadata using exact store identifiers.

## Exploratory Data Analysis (EDA)

EDA was performed primarily on the Tier-1 data:

*   **Log Transformation**: Raw `unit_price` and `quantity` distributions were heavily right-skewed. A `log(1+x)` transformation was applied to visualize central tendencies, identify genuine outliers, and stabilize variance for modeling.
*   **Relationship Analysis**: Scatter plots and a correlation heatmap revealed very weak linear correlations between `unit_price`, `discount`, `quantity`, and `month`, suggesting that demand was likely driven by external factors not present in the dataset.
*   **Business Questions**: Store and channel analysis were deemed unreliable due to significant missing metadata. Discounts showed a limited positive impact on quantity.

## Modeling Approach

The objective was to assess the predictability of sales (`quantity`). Features used for modeling were `unit_price_clean`, `discount_clean`, and `month`.

### 1. Initial Regression Attempt

*   **Target**: `quantity_clean`
*   **Models**: Linear Regression, Random Forest Regressor
*   **Results**: Both models yielded R² values near zero or negative, with high RMSE. This indicated that the available features were insufficient for accurate quantitative forecasting.

### 2. Reformulation to Classification

Given the poor regression performance, the task was reframed to predict `high_demand` vs. `low_demand`.

*   **Target**: `high_demand` (binary: `1` if `quantity > median(quantity)`, `0` otherwise)
*   **Models**: Logistic Regression, Random Forest Classifier
*   **Results**:
    *   **Logistic Regression**: Accuracy 62.9%, but the model primarily predicted the majority class, with very low recall for the minority class.
    *   **Random Forest Classifier**: Accuracy 54.5%, with a class-1 recall of 0.29.

*Conclusion: Even with reformulation, predictive power remained weak, confirming the limited signal in the cleaned data for robust demand prediction.*

## Key Learnings & Insights

*   **Data Cleaning as Reasoning**: Cleaning is not just formatting; it's about understanding and reasoning through data integrity issues.
*   **Analytical Usability**: The majority of raw logs were analytically unusable, highlighting the severity of the ERP migration failure.
*   **Honest Recovery**: Preserving uncertainty and honestly reporting data quality limitations is more valuable than fabricating artificial accuracy.
*   **Model Performance Reflects Reality**: Poor model performance accurately reflected the inherent noise and missing information in the data, rather than any flaw in the modeling approach.

## Limitations

*   Significant loss of granular `product_category`, `store`, and `sales_channel` information due to high corruption.
*   Absence of critical external factors like promotional calendars, customer data, and stock levels.
*   Persistent heavy missingness in the `discount` field even after imputation.

## Future Work

To enable more effective sales forecasting, future efforts should focus on:

*   Collecting reliable product and store dimension data.
*   Integrating inventory and promotion management systems.
*   Incorporating event-based features (holidays, marketing campaigns).
*   Implementing robust data logging standards to prevent future corruption.

## Conclusion

This project successfully demonstrated that when data is severely corrupted, the primary achievement lies in **truth recovery** rather than achieving high predictive accuracy. Through a meticulously designed pipeline involving tier-based validation, conditional imputation, log-transformed EDA, and iterative correction of assumptions, an honest analytical framework was established. The final outcome provided critical feedback to the organization: meaningful demand prediction was not feasible with the available features, thereby guiding future data collection priorities and strategic investments in data quality.
