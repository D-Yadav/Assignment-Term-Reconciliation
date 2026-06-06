# Term Sheet Reconciliation Assignment

## Overview
This repository contains an end-to-end automated reconciliation workflow for the assignment data set. The solution parses term sheet PDFs, extracts tables to Excel, uses LLM-based field extraction with evidence capture, validates the extracted payload against a strict schema, compares the structured output against booking extracts from CSV and JSON, and runs a second automated LLM pass to adjudicate mismatches as either `False Alert` or `True Mismatch`.

The implementation was built and tested in Databricks using a single notebook-driven workflow and writes reproducible output artifacts for reviewer inspection.

## Repository Contents
Recommended repository structure:

* `README.md`
* `REPORT.md`
* `notebooks/Term Sheet Reconciliation.ipynb`
* `src/` for extracted helper functions if you export notebook logic into Python modules
* `data/term_sheets/` with sample term sheets
* `data/bookings/` with sample CSV and JSON booking extracts
* `outputs/term_sheet_extracts/` for generated Excel extracts
* `outputs/reconciliation/` for generated CSV outputs
* `requirements.txt`
* `.env.example`

## Environment Setup
### Python and runtime
* Python 3.11 or later recommended
* Databricks Runtime 17.3 or later recommended
* Tested on Azure Databricks classic cluster

### Python dependencies
Install the required dependencies before running the workflow:

* `pandas`
* `pydantic`
* `openpyxl`
* `lxml`

If you are running in Databricks, install notebook-scoped dependencies first and restart Python if needed.

### API and model requirements
The workflow uses fully automated calls to Databricks SQL AI functions:

* `ai_parse_document` for document parsing
* `ai_query` for structured extraction and mismatch adjudication

If you adapt this outside Databricks, configure a supported LLM provider with a public developer API and expose credentials through environment variables or secret management. Do not hardcode API keys in code.

### Configuration
Set up the following configuration before execution:

* Assignment base path for the input files
* Output path for Excel extracts
* Output path for reconciliation artifacts
* Access to the selected LLM endpoints or AI SQL functions

Recommended environment variables for a portable Python implementation:

* `ASSIGNMENT_BASE`
* `TERM_SHEET_OUTPUT_DIR`
* `RECON_OUTPUT_DIR`
* `DATABRICKS_HOST`
* `DATABRICKS_TOKEN`

## Input Files
The workflow expects these input groups:

* Sample term sheets in PDF or Word format
* Booking extracts in CSV format
* Booking extracts in JSON format

Expected fields in booking extracts include identifiers and instrument attributes such as:

* `ISIN`
* `Issuer`
* `Coupon`
* `Currency`
* `Maturity`
* `IssueDate`
* `SettlementDate`
* `IssueAmount`
* `IssuePrice`
* `NominalAmountPerBond`
* `InterestPaymentDate`
* `InterestPaymentFrequency`
* `DayCountFraction`
* `BusinessDayConvention`
* `BusinessDayLocation`
* `AmortizationType`
* `MinimumSubscription`
* `Parent`
* `Notional`

## Output Files
A successful run produces:

* Excel extracts for each parsed term sheet
* `validated_extractions.csv`
* `reconciliation_results.csv`
* `reconciliation_summary.csv`

The detailed reconciliation output includes:

* document key
* trade id
* source format
* field name
* booking value
* extracted value
* evidence
* first-pass comparison result
* final adjudicated status
* reason

## End-to-End Execution
Following your preferences, here is the exact workflow sequence used in the notebook.

### Step 1: Install dependencies
Install required packages and restart Python if the environment does not already contain them.

### Step 2: Configure paths
Set the assignment input directory, Excel export directory, reconciliation output directory, and any temporary working folders.

### Step 3: Parse the PDFs
Run document parsing on the term sheet files and collect both document text and extracted HTML tables.

### Step 4: Export tables to Excel
Write each extracted table set to a dedicated Excel workbook for reviewer validation.

### Step 5: Extract fields with an LLM
Run a first LLM pass to return JSON only, including the requested fields plus exact evidence snippets for each field.

### Step 6: Validate and normalize the extraction
Normalize ISINs, dates, numeric values, business-day locations, INR crore expressions, and similar field variants. Validate the normalized payload with a strict schema. Mark failures as parsing errors.

### Step 7: Ingest booking extracts
Read both CSV and JSON booking files, tag their source format, and map booking columns to the canonical reconciliation schema.

### Step 8: Reconcile fields
Compare every target field between booking data and the validated extraction. Mark exact or normalized equality as `Match`; otherwise mark as `Mismatch`.

### Step 9: Adjudicate mismatches with a second LLM pass
For mismatches only, run a second automated LLM step that classifies each as `False Alert` or `True Mismatch` and provides a reason.

### Step 10: Save outputs
Write the validated extraction, detailed reconciliation results, and summary counts to CSV files.

## Running the Notebook End to End in Databricks
1. Open `Term Sheet Reconciliation.ipynb`.
2. Confirm the input paths point to the included sample files.
3. Run the notebook top to bottom.
4. Verify that Excel extracts are created for each term sheet.
5. Verify that the three CSV outputs are created and non-empty.
6. Review summary counts and sample detailed mismatch rows.

## Reproducibility Notes
* The workflow is fully automated with no manual extraction or manual classification steps.
* The same mock data and setup steps should reproduce the generated outputs.
* Reviewers should only need environment setup plus a full notebook run.
* For public repository submission, include sample inputs and a requirements file so the workflow can be rerun immediately after cloning.

## Public Repository Submission Checklist
Before submitting the public GitHub repository, ensure it contains:

* all source code and notebooks
* sample term sheets and booking extracts
* this `README.md`
* the concise assignment report
* dependency specification such as `requirements.txt`
* configuration template such as `.env.example`
* generated example outputs if allowed by the assignment

## Suggested GitHub Publishing Flow
1. Create a new public GitHub repository.
2. Add the notebook, source files, sample data, documentation, and dependency files.
3. Commit all files with a clear message.
4. Push to the public remote repository.
5. Verify that a fresh clone contains all files required to run the workflow.
6. Re-run the workflow from the documented setup steps to confirm reproducibility.
