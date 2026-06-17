# BatchData Property Search — Google Sheets Case Study

## Project Summary

This repository is a production-ready Google Apps Script integration that converts the BatchData Property Search API into a self-serve lead-generation workflow inside Google Sheets.

The script provides a complete user experience from filter configuration and payload preview to result ingestion, deduplication, and runtime logging. It is designed for teams that need a low-code property data pipeline without building a separate web application.

## Why this solution exists

BatchData provides rich property, mortgage, foreclosure, valuation, and contact data via API. This project solves a common need:

- turn API output into structured spreadsheet output
- avoid duplicate leads across repeated runs
- preserve a persistent search offset for continuous collection
- let non-technical users control searches through a sheet UI

## Architecture

The implementation is a single Google Apps Script project attached to a Google Sheet. It uses the following architectural components:

- `onOpen()` to inject a custom menu into Google Sheets
- `setupSheets()` to create and format three worksheets:
  - `🔎 Filters`
  - `📋 Results`
  - `📝 Run Log`
- a filter-driven payload generator for BatchData property search
- a single-request fetcher to minimize API billing
- deduplication using APN, BatchData Property ID, and address hash
- runtime state stored in `PropertiesService` for skip-offset tracking
- separate debug helpers to inspect payloads, individual record fields, and raw API responses

## Project Files

- `Code_v2_final.gs` — recommended current implementation (v2.1)
- `Code_v2.gs` — alternate v2.0 implementation with a different column layout and filter model
- `Code.gs` — legacy v1 implementation for reference
- `SETUP_GUIDE.md` — deployment and user onboarding notes
- `README.md` — this case study

## Technical Scope

This script supports:

- location filtering by State / County / City / Zip Code(s)
- property classification filters such as property use and property value ranges
- loan and mortgage filters including loan amount, loan origin date, loan type, and interest rate
- optional skip-trace contact enrichment via BatchData
- results normalization into a sheet-friendly layout
- deduplication and smart offset handling so the same record is not fetched twice
- previewing request payloads before spending API credits

## Implementation Details

### Filter to Payload Mapping

The script transforms sheet controls into BatchData search criteria with path-based operators. Key mappings include:

- `address` filters → structured `searchCriteria.address` operators
- `valuation.estimatedValue` → property value range filters
- `sale.lastSalePrice` → purchase price range filters
- `openLien.totalOpenLienBalance` → loan amount filters
- `openLien.originationDate` → loan origination date filters
- `openLien.firstLoanInterestRate` → interest rate thresholds
- `openLien.firstLoanType` → loan type equality checks
- `options.skipTrace` → optional contact enrichment

### Deduplication

Duplicate prevention is implemented both server-side and client-side:

- existing `Property ID` values are sent in `ids.propertyId.notInList`
- known APN values and address hashes are also tracked for client-side dedup
- `Results` rows store dedup keys alongside each record so the exclusion set is persistent

This reduces the chance of billing the API for the same lead more than once.

### Skip Offset

The script stores a persistent offset value in `PropertiesService` under the key `SKIP_TOTAL`.

- On first run, the offset initializes from the current `Results` row count
- After each API fetch, the offset advances by the count of fetched records
- `resetSearchOffset()` can resynchronize the offset with current results

This allows repeated runs to sequentially consume new records from the API.

### Result Mapping

The `Results` sheet is intentionally formatted to match a client-specific data layout.

`Code_v2_final.gs` writes 38 columns in this order:

- Owner details: first/last name, mailing address, up to 3 emails and phones
- Property details: address, APN, property type/use/zoning
- Valuation: estimated value, equity
- Loan details: lender, lender type, amount, interest rate, loan type, recording/maturity dates
- Mortgage release and foreclosure fields
- Transfer history, previous owners, and pull metadata
- hidden dedup columns: `Property ID`, `Address Hash`

### Mock Mode and Testing

The project includes self-contained mock responses to let developers verify behavior without API charges.

- `MOCK_MODE = true` returns sample records
- `previewPayload()` renders the exact JSON payload in the `🔧 Debug` sheet
- `inspectFields()` / `inspectDedupFields()` fetch a single record to validate API field paths

### Debug Tools

The script provides utility actions via the custom menu:

- `Preview Payload (FREE)` — shows the request body without calling the API
- `Inspect Fields` — fetches one record and lists important field paths
- `Debug: Show Raw API Response` — writes the raw HTTP response to the debug sheet
- `Clear Results Only` — removes results while preserving sheet structure
- `Reset Search Offset` — recalculates or resets the skip offset

## Setup Instructions

### 1. Create the Google Sheet

- Open [Google Sheets](https://sheets.google.com)
- Create a blank spreadsheet
- Open `Extensions → Apps Script`
- Replace the default script with the chosen `.gs` file contents
- Save and run `onOpen()` once to authorize the script
- Refresh the sheet to expose the `BatchData Search` menu

### 2. Configure the API Token

- Create a BatchData Server-Side token for `property-search`
- Enable required datasets:
  - Core Property Data
  - Valuation
  - Mortgage / Lien
  - Deed / Sale History
  - Contact Enrichment (if using Skip Trace)
- Paste the key into `API_KEY` in the script
- Use `MOCK_MODE = true` until you are ready to run production searches

### 3. Run Setup

- Choose `BatchData Search → ⚙ Setup Sheets`
- This creates the filter, results, and log worksheets
- The filter sheet includes prebuilt dropdowns and notes for each search parameter

### 4. Execute a Search

- Configure filters on `🔎 Filters`
- Use `Preview Payload (FREE)` to verify the request content
- Run `▶ Run Search`
- Inspect results on `📋 Results`
- Review run history on `📝 Run Log`

## Result Columns

The recommended `Code_v2_final.gs` results structure includes the following fields:

- Owner First Name
- Owner Last Name
- Owner Mailing Address
- Email 1 / Email 2 / Email 3
- Phone 1 / Phone 2 / Phone 3
- Property Address / City / County / State / Zip / APN
- Property Type / Property Use / Property Zoning
- Est. Property Value / Equity %
- Loan Lender Name / Lender Type / Loan Amount / Loan Interest Rate
- Loan Type / Recording Date / Loan Maturity Date
- Mortgage Release Date / Mortgage Release Amount
- Notice of Default Date / Notice of Sale Date / Notice of Lis Pendens Date / Notice of Rescission Date
- Transfer of Ownership Date(s) / Previous Owner Name(s)
- Date Pulled
- Property ID / Address Hash

## Troubleshooting

### Common issues

- `API key not set`: verify `API_KEY` is defined and not left as the placeholder value
- `Setup Sheets first`: run `BatchData Search → ⚙ Setup Sheets`
- `0 records returned`: broaden filters or verify token dataset privileges
- `HTTP 401`: invalid or expired API key
- `HTTP 403`: missing BatchData access rights or insufficient account balance
- dedup behavior not working: confirm the `Property ID` and `Address Hash` columns are populated in `📋 Results`

### Data quality notes

- The script sanitizes numeric fields to reject date-like values in amount fields
- Interest rate values above 50% are treated as invalid and discarded
- Records without an `_id` or `address.street` are filtered out as junk

## Production considerations

- Do not commit `API_KEY` values to source control
- Use App Script project properties or another secure secret store for secrets in collaborative environments
- Confirm the token's enabled datasets before switching from `MOCK_MODE`
- Run a small pilot search first to verify filters and avoid unnecessary credit usage

## Future enhancements

Potential next steps for the project:

- add CRM export/trigger integration (e.g. Zapier, Make, or direct CRM API)
- support additional BatchData `property-search` filters
- centralize configuration into a dedicated script properties panel
- support incremental delta sync with timestamp-based updates
- add sheet-level validation for required filter fields

## Why this is a strong profile project

- demonstrates full-stack scripting inside Google Sheets
- shows real API integration and business-data modeling
- includes production-grade features like deduplication, logging, and mock testing
- is readable, maintainable, and easy to deploy for non-technical users

## Notes

- This case study is based on the current workspace contents and the most up-to-date implementation in `Code_v2_final.gs`.
- Confidential API credentials were intentionally excluded from this document.
