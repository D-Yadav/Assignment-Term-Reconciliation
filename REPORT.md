# Concise Workflow Report

## Objective
The goal of this solution is to automate reconciliation between term sheet documents and booking-system extracts with no manual review steps in the extraction pipeline. The workflow had to parse semi-structured source documents, extract required bond fields, preserve evidence, validate outputs against a schema, compare them with booking files from multiple source formats, and classify mismatches for reviewer-friendly output.

## Workflow Summary
The solution is implemented as an end-to-end Databricks notebook pipeline.

1. Input files are loaded from the assignment folder, including term sheet PDFs and booking extracts in CSV and JSON.
2. `ai_parse_document` is used to parse each term sheet and surface document text plus detected table content.
3. Detected tables are exported into Excel workbooks so reviewers can inspect extracted tabular content independently of the LLM reconciliation output.
4. A first `ai_query` call performs structured field extraction. The prompt requires JSON-only output and asks for all target fields together with an `evidence` object that stores the exact supporting quote for each field.
5. The extracted payload is normalized and validated through a strict Pydantic schema. Numeric parsing handles large-value text patterns such as crore, million, and billion. Date parsing converts detected date values into ISO form where possible. ISIN and text fields are whitespace-normalized before comparison.
6. Booking extracts from both CSV and JSON are ingested and mapped into the same canonical field structure used by the extracted term-sheet payload.
7. Field-by-field reconciliation compares normalized booking values and normalized extracted values.
8. Every mismatch is passed to a second `ai_query` call using a smaller model for adjudication. This step labels mismatches as `False Alert` or `True Mismatch` and records a reason.
9. Final outputs are written to CSV files for validated extractions, detailed reconciliation results, and reconciliation summary counts.

## LLM and API Integration Design
The workflow uses fully automated Databricks SQL AI functions:

* `ai_parse_document` for document parsing and table extraction
* `ai_query` for first-pass structured extraction
* `ai_query` again for mismatch adjudication

This design keeps the pipeline automation-friendly and reproducible inside a single environment. The first LLM call focuses on breadth and evidence capture. The second call is narrower and lower cost because it only processes rows already marked as mismatches. That separation reduces cost while improving interpretability.

## Field Extraction Design
The extraction schema covers the required term-sheet fields, including identifiers, dates, monetary values, day-count fields, payment terms, business-day information, and supporting parent/notional attributes. Each field is normalized before validation or comparison so that formatting differences do not immediately create false mismatches.

Examples of normalization rules include:

* removing spacing from ISIN values
* converting text amounts into numeric values
* handling Indian crore notation
* converting recognized dates into ISO format
* splitting multi-location business-day fields into normalized lists
* trimming explanatory parenthetical suffixes from issuer and parent names where needed

The schema-validation step ensures malformed or partial LLM output is surfaced explicitly as a parsing error rather than silently accepted.

## Challenges Encountered
The main implementation challenges were:

* semi-structured term-sheet wording varies significantly by issuer and document style
* some fields are implied rather than explicitly stated in one exact format
* monetary fields required robust handling of Indian numbering conventions
* converting mixed pandas object columns into Spark-safe structures required explicit coercion
* some mismatches were not true business mismatches but representation differences, which required a second automated adjudication stage

A practical example is the IDBI document, where several fields differ from the booking rows because the booking sample intentionally contains altered or simplified values. Another example is when the booking system expresses a field as a concrete date while the term sheet describes it narratively, such as anniversary-based payment wording.

## Assumptions
The following assumptions were applied in the current implementation:

* if the Genel settlement date is absent in the first extraction pass, it can default to the issue date when supported by document wording
* if the IDBI currency is not explicitly returned, it can be normalized to INR from document context
* if the IDBI amortization type is missing but the document is perpetual, the normalized amortization type is set to `Perpetual`
* evidence is stored exactly as returned by the extraction pass, even when the normalized field value differs from the raw wording
* all mismatch adjudication remains automated and uses only the booking value, extracted value, and evidence

## Production Automation and DevOps Suggestions
For a production version, I would extend this notebook into a modular package with separate scripts for parsing, extraction, normalization, reconciliation, and output publishing. I would also add:

* parameterized configuration files for paths, models, and environments
* automated tests for normalization and reconciliation functions
* CI checks for dependency installation and smoke-test execution on mock data
* structured logging and run metadata capture
* secret-based credential management instead of inline configuration
* job orchestration with scheduled runs and artifact versioning
* quality thresholds and alerting when parsing-error rates or true-mismatch rates spike

A production deployment could run as a scheduled Databricks job with output files written to a governed storage location and with summary metrics exposed to dashboards or downstream monitoring.

## End-to-End Execution
The workflow is executed top to bottom in the Databricks notebook:

1. install dependencies
2. define paths and helper functions
3. parse PDFs and export Excel files
4. run first-pass extraction and schema validation
5. ingest booking CSV and JSON files
6. reconcile extracted fields to booking data
7. adjudicate mismatches with the second LLM pass
8. save validated extraction, detailed results, and summary outputs

This setup satisfies the reproducibility requirement because a reviewer can clone the repository, configure the environment as described in the README, run the workflow once, and inspect the generated artifacts without manual intervention.
