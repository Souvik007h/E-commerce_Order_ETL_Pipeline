
# Business Rules — E-Commerce Order ETL Pipeline

## 1. Purpose

This document defines the business rules used by the E-Commerce Order ETL Pipeline, implemented using Apache Hop 2.19.0.

The pipeline processes order transactions and customer master data, validates data quality, identifies duplicate orders, enriches transactions with customer information, calculates financial values, and generates structured output files for reporting.

The objective is to ensure that only eligible, validated orders reach the final output while invalid, duplicate, and ineligible records are handled separately.

## 2. Input Data

The pipeline consumes two CSV files.

| File              | Description                                                                                  |
| ----------------- | -------------------------------------------------------------------------------------------- |
| `orders.csv`    | Contains order identifiers, customer references, product names, quantities, and unit prices. |
| `customers.csv` | Contains customer identifiers, customer names, cities, and customer statuses.                |

The original input files must remain unchanged. All transformations and generated outputs are written separately.

## 3. Order Validation Rules

### 3.1 Required Fields

The following fields are required for each order:

- `order_id`
- `customer_id`
- `product`
- `quantity`
- `unit_price`

Records with missing or blank required fields must be rejected.

### 3.2 Identifier Format

Order IDs must start with uppercase `O`, followed by one or more digits.

**Regex:**

`^O[0-9]+$`

Customer IDs must start with uppercase `C`, followed by one or more digits.

**Regex:**

`^C[0-9]+$`

Examples:

| Value      | Expected result     |
| ---------- | ------------------- |
| `O1001`  | Valid order ID      |
| `O12345` | Valid order ID      |
| `C101`   | Valid customer ID   |
| `O12A`   | Invalid order ID    |
| `C10A`   | Invalid customer ID |

Identifier validation checks the format only. It does not establish whether the referenced customer exists.

### 3.3 Quantity Validation

The `quantity` field must be numeric, represent a whole number, and be greater than zero.

Records with missing, non-numeric, zero, negative, or fractional quantities must be rejected.

### 3.4 Unit Price Validation

The `unit_price` field must be numeric and greater than zero.

Records with missing, non-numeric, zero, or negative unit prices must be rejected.

### 3.5 Duplicate Order Detection

Duplicate orders are identified using `order_id`.

- The order stream is sorted by `order_id` before duplicate detection.
- One occurrence of each order ID is retained.
- Additional occurrences are routed to the duplicate output.
- Orders are not considered duplicates merely because they share a customer, product, quantity, or price.

The implementation must use the duplicate-handling behavior configured in the Apache Hop pipeline.

## 4. Customer Master Data Validation

Customer master records are validated before they are used for order enrichment.

### 4.1 Required Customer Fields

The following fields are required:

- `customer_id`
- `customer_name`
- `city`
- `customer_status`

Missing or blank required values must be rejected.

### 4.2 Customer Identifier Format

Customer IDs must match the pattern:

`^C[0-9]+$`

### 4.3 Customer Status

The accepted customer statuses are:

- `ACTIVE`
- `INACTIVE`

Status values should be standardized before validation. Missing or unexpected statuses should be rejected with an appropriate reason.

### 4.4 Duplicate Customer IDs

Customer IDs should be unique in the customer master data.

Duplicate customer identifiers must be identified and resolved or rejected according to the configured business rule before the lookup is used for enrichment.

## 5. Customer Enrichment Rules

Validated orders are enriched using the customer master dataset by matching:

`orders.customer_id = customers.customer_id`

The lookup adds the following customer attributes:

- `customer_name`
- `city`
- `customer_status`

### 5.1 Unknown Customers

If an order references a customer ID that does not exist in the customer master data, the order is rejected.

Rejection code: `CUSTOMER_NOT_FOUND`

### 5.2 Inactive Customers

Orders associated with customers whose status is `INACTIVE` are rejected.

Rejection code: `CUSTOMER_INACTIVE`

### 5.3 Eligible Customers

Only orders associated with an `ACTIVE` customer may proceed to the final order-processing stage.

Unexpected or missing customer statuses must not be treated as valid active customers.

## 6. Financial Calculation Rules

Financial calculations are performed only on orders that pass the applicable validation and customer eligibility checks.

### 6.1 Order Value

The order value is calculated as:

`order_value = quantity × unit_price`

Example:

| Quantity | Unit price | Order value |
| -------: | ---------: | ----------: |
|        3 |       1500 |        4500 |

### 6.2 Tax Calculation

An illustrative tax rate of 18% is used for this project.

`tax_amount = order_value × 0.18`

### 6.3 Final Amount

The final amount is calculated as:

`final_amount = order_value + tax_amount`

Example:

| Field            |   Value |
| ---------------- | ------: |
| `order_value`  | 4500.00 |
| `tax_amount`   |  810.00 |
| `final_amount` | 5310.00 |

The tax rate is a configurable project assumption used to demonstrate financial calculations. It does not represent tax advice or a determination of the legally applicable tax treatment.

## 7. Order Value Classification

Orders may be categorized using the following thresholds, based on `order_value` before tax.

| Condition                       | Category   |
| ------------------------------- | ---------- |
| `order_value < 1000`          | `SMALL`  |
| `1000 <= order_value < 10000` | `MEDIUM` |
| `order_value >= 10000`        | `LARGE`  |

These thresholds are illustrative business rules and may be adjusted as required.

## 8. Rejection Handling

Invalid records must be routed to rejection outputs rather than the final valid-order output.

Suggested rejection codes include:

| Rejection code              | Description                                                    |
| --------------------------- | -------------------------------------------------------------- |
| `MISSING_ORDER_ID`        | Order identifier is missing or blank.                          |
| `INVALID_ORDER_ID`        | Order identifier does not match the required format.           |
| `MISSING_CUSTOMER_ID`     | Customer identifier is missing or blank.                       |
| `INVALID_CUSTOMER_ID`     | Customer identifier does not match the required format.        |
| `MISSING_PRODUCT`         | Product name is missing or blank.                              |
| `INVALID_QUANTITY`        | Quantity is missing, non-numeric, non-positive, or fractional. |
| `INVALID_UNIT_PRICE`      | Unit price is missing, non-numeric, or non-positive.           |
| `CUSTOMER_NOT_FOUND`      | Customer ID does not exist in the customer master.             |
| `CUSTOMER_INACTIVE`       | Customer status is`INACTIVE`.                                |
| `INVALID_CUSTOMER_STATUS` | Customer status is missing or unrecognized.                    |
| `DUPLICATE_ORDER_ID`      | Additional occurrence of an order ID.                          |

These are the intended rejection codes. Only codes implemented by the current pipeline should be described as operationally supported.

## 9. Output Files

| Output file                     | Purpose                                                                                                       |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `valid_orders.csv`            | Valid, deduplicated orders for eligible customers, enriched with customer details and financial calculations. |
| `rejected_orders.csv`         | Invalid orders and orders rejected during customer validation or enrichment, with rejection reasons.          |
| `duplicate_orders.csv`        | Additional records identified as duplicates by order ID.                                                      |
| `product_revenue_summary.csv` | Product-level aggregated metrics, if the reporting pipeline is implemented.                                   |

Output filenames may vary according to the configured pipeline paths.

## 10. Data Integrity and Operational Requirements

- Source CSV files must remain unchanged.
- Output files should be generated using the configured overwrite behavior.
- Output schemas should be consistent and documented.
- Invalid records must not reach the final valid-order output.
- Duplicate records must remain separate from ordinary rejected records.
- Numeric calculations must use suitable numeric data types.
- Generated outputs should be verified against the business rules.
- Pipeline failures and unexpected input conditions should be investigated rather than silently ignored.

## 11. Reconciliation Principle

Once the processing order and rejection categories have been defined, every input order row should be accounted for.

For mutually exclusive final categories:

`Input rows = Valid rows + Rejected rows + Duplicate rows`

This equation must be adjusted if the implementation routes duplicates or invalid records in a different order. Reconciliation counts should be verified using actual pipeline execution results.

## 12. Scope and Limitations

This project uses CSV files and synthetic sample data to demonstrate visual ETL development with Apache Hop 2.19.0.

The documented business rules are intended for learning and demonstration. Production deployment would require additional considerations, including access control, operational monitoring, schema evolution, transactional loading, audit logging, and formal data governance.
