---
name: call-api
description: "Use when testing local API endpoints with auto-generated data. Lists available APIs when run without args."
---

# Call API - Local Testing Tool

This skill calls local API endpoints with auto-generated test data.

## How to use

The user provides an API name as an argument: `/call-api <api-name>`

- If no argument is given, list all available APIs from the index below.
- If the argument doesn't match, tell the user and list available options.
- If it matches, read the corresponding file in `apis/` (see File column in the index) for the full API definition, then execute.
- **Read `setup.md` before making curl calls** — it has auth methods, base URL, and common setup.
- For test data IDs, seed dependencies, and seed SQL, see `test-data.md` in this directory.

### When test data is missing

If the API call fails (404, 500, or empty results), seed the required data:

1. Read `test-data.md` → find the API's **Seed Dependency Group**
2. Run the **check query** to see which entities are missing
3. For each missing entity (in listed order), invoke the `generate-seed-data` skill with the `--fk` and `--execute` flags shown
4. After seeding, retry the API call

## API Index

| Name | Method | Path | Auth | File |
|---|---|---|---|---|
| **Application** | | | | |
| `create-application-v2.1` | POST | `/v2.1/application` | api-secret | `apis/create-application.md` |
| **Document Check** | | | | |
| `doc-check-assets-v2` | GET | `/v2/underwritings/{uwId}/assignment/document/assets` | JWT | `apis/doc-check-summary.md` |
| `doc-check-summary-v2-get` | GET | `/v2/underwritings/{uwId}/assignment/document/summary` | JWT | `apis/doc-check-summary.md` |
| `doc-check-summary-v2-update` | PUT | `/v2/underwritings/{uwId}/assignment/document/summary/update` | JWT | `apis/doc-check-summary.md` |
| `return-reason-asset-list` | GET | `/v1/underwritings/{uwId}/assignment/document/return-reason/assets` | JWT | `apis/doc-check-return-reason.md` |
| `return-reason-asset-detail` | GET | `/v1/underwritings/{uwId}/assignment/document/return-reason/assets/{collateralDetailId}` | JWT | `apis/doc-check-return-reason.md` |
| `return-reason-customer` | GET | `/v1/underwritings/{uwId}/assignment/document/return-reason/customers` | JWT | `apis/doc-check-return-reason.md` |
| `return-reason-asset-save` | PUT | `/v1/underwritings/{uwId}/assignment/document/return-reason/assets/{collateralDetailId}` | JWT | `apis/doc-check-return-reason.md` |
| `return-reason-customer-save` | PUT | `/v1/underwritings/{uwId}/assignment/document/return-reason/customers/{additionalNoteId}` | JWT | `apis/doc-check-return-reason.md` |
| **Review** | | | | |
| `review-assignment-v2` | GET | `/v2/underwritings/{uwId}/assignment/review` | JWT | `apis/review-detail.md` |
| `review-detail-regular-v2` | GET | `/v2/underwritings/{uwId}/assignment/review/detail/regular` | JWT | `apis/review-detail.md` |
| `review-draft-save-v2` | PUT | `/v2/underwritings/{uwId}/assignment/review/summary/regular/draft` | JWT | `apis/review-detail.md` |
| `review-submit-v2` | PUT | `/v2/underwritings/{uwId}/assignment/review/summary/regular` | JWT | `apis/review-detail.md` |
| `review-assets-list` | GET | `/v1/underwritings/{uwId}/assignment/review/assets` | JWT | `apis/review-detail.md` |
| `review-return-reason-asset-list` | GET | `/v1/underwritings/{uwId}/assignment/review/return-reason/assets` | JWT | `apis/review-return-reason.md` |
| `review-return-reason-asset-detail` | GET | `/v1/underwritings/{uwId}/assignment/review/return-reason/assets/{reviewCollateralId}` | JWT | `apis/review-return-reason.md` |
| `review-return-reason-customer` | GET | `/v1/underwritings/{uwId}/assignment/review/return-reason/customers` | JWT | `apis/review-return-reason.md` |
| `review-return-reason-asset-save` | PUT | `/v1/underwritings/{uwId}/assignment/review/return-reason/assets/{reviewCollateralId}` | JWT | `apis/review-return-reason.md` |
| `review-return-reason-customer-save` | PUT | `/v1/underwritings/{uwId}/assignment/review/return-reason/customers/{reviewAdditionalNotesId}` | JWT | `apis/review-return-reason.md` |
| **CFCAT** | | | | |
| `cfcat-asset-list` | GET | `/v1/underwritings/{uwId}/assignment/review/detail/delimited-data/{type}/assets` | JWT | `apis/cfcat.md` |
| `cfcat-asset-detail` | GET | `/v1/underwritings/{uwId}/assignment/review/detail/delimited-data/{type}/assets/{uwReviewCollateralId}` | JWT | `apis/cfcat.md` |
| `cfcat-non-asset` | GET | `/v1/underwritings/{uwId}/assignment/review/detail/delimited-data/non-asset` | JWT | `apis/cfcat.md` |
| `cfcat-asset-save` | PUT | `/v1/underwritings/{uwId}/assignment/review/detail/delimited-data/{type}/assets/{uwReviewCollateralId}` | JWT | `apis/cfcat.md` |
| **Financial Statements** | | | | |
| `financial-statements` | GET | `/v1/underwritings/summary/applications/{applicationId}/financial-statements` | JWT | `apis/financial-statements.md` |
| **Upload** | | | | |
| `underwriting-upload` | POST | `/v1/underwritings/upload` | JWT | `apis/underwriting-upload.md` |
| **Negotiation** | | | | |
| `negotiation-get` | GET | `/v1/underwritings/{uwId}/negotiation` | JWT | `apis/negotiation.md` |
| `negotiation-submit` | POST | `/v1/underwritings/{uwId}/negotiation` | JWT | `apis/negotiation.md` |

## Adding New APIs

1. Add a row to the API Index table above (under the appropriate domain group, with the `File` column pointing to the right domain file)
2. Add the full definition in the appropriate `apis/*.md` file, or create a new domain file if needed
3. Add any test data to `test-data.md`
