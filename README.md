# E-Commerce Purchase Request Processing (RPA)

An end-to-end UiPath automation that reads employee purchase requests from Excel, validates them against live product and exchange-rate data, completes approved test purchases on SauceDemo, and reports the results by Excel and email.

The solution follows the **Dispatcher / Performer / Reporter** pattern using UiPath Orchestrator queues.

---

## Table of Contents

- [Overview](#overview)
- [Target Applications](#target-applications)
- [Architecture](#architecture)
- [Business Process](#business-process)
- [Input File Specification](#input-file-specification)
- [Output Report Specification](#output-report-specification)
- [Configuration](#configuration)
- [Business Rules & Exception Handling](#business-rules--exception-handling)
- [Idempotency & Rerun Safety](#idempotency--rerun-safety)
- [Getting Started](#getting-started)
- [Project Structure](#project-structure)
- [Testing Scenarios](#testing-scenarios)

---

## Overview

| Item | Detail |
|---|---|
| Process name | E-Commerce Purchase Request Processing |
| Platform | UiPath Studio / Orchestrator |
| Queue | `PurchaseRequestsQueue` |
| Input | `PurchaseRequests.xlsx` |
| Output | Excel processing report + email notifications |
| Final statuses | `Completed`, `Rejected`, `Failed` |

## Target Applications

| Application | Purpose | Link |
|---|---|---|
| SauceDemo | E-commerce test application (product lookup, cart, checkout) | https://www.saucedemo.com/ |
| Frankfurter API | Currency exchange rates | https://frankfurter.dev/ |
| Microsoft Excel | Input requests and output report | – |
| Outlook / SMTP | Employee notifications and summary email | – |

**SauceDemo test credentials:** `standard_user` / `secret_sauce`
(Public test credentials. In the automation, store them as an Orchestrator Credential asset, not in code.)

**Example API call (USD → EUR):** `https://api.frankfurter.dev/v2/rate/usd/eur`

## Architecture

```
┌────────────────┐     ┌─────────────────────┐      ┌─────────────────┐
│  Dispatcher    │───▶│ PurchaseRequestsQueue│───▶ │    Performer    │
│ (read, filter, │     │   (Orchestrator)    │      │ (SauceDemo, API,│
│  sort, upload) │     └─────────────────────┘      │  checkout)      │
└────────────────┘                                  └────────┬────────┘
                                                             │
                                                             ▼
                                                  ┌──────────────────┐
                                                  │    Reporter      │
                                                  │ (Excel report +  │
                                                  │  summary email)  │
                                                  └──────────────────┘
```

- **Dispatcher** – Reads and filters the input file, sorts by priority, and uploads transactions to the queue.
- **Performer** – Processes each queue item as an independent transaction.
- **Reporter** – Finalizes the Excel report and sends the summary email to Procurement Operations.

## Business Process

### 1. Receive Purchase Requests (Dispatcher)

1. Read `PurchaseRequests.xlsx` from the configured location.
2. Keep only rows where **Request Status = Pending** (the column is named `Status` in the supplied file).
3. Sort by **Priority**: High → Medium → Low. Rows with the same priority keep their original input order (stable sort).
4. Upload each request individually to `PurchaseRequestsQueue`:
   - **Reference** = `Request ID` (unique, prevents duplicate queue items)
   - **Priority** = High / Medium / Low mapped to the corresponding queue priority
   - **Specific data** = Request ID plus every field the Performer needs
5. Log duplicate or skipped requests in a pre-processing log/report.
6. Skip any Request ID that has already been completed.

The Performer retrieves items in queue priority order.

### 2. Collect Product Information (Performer)

1. Open SauceDemo and log in with the test credentials.
2. Check whether the requested product exists in the SauceDemo catalog.
3. If it does not exist, mark the request **Rejected** and record the reason.

### 3. Validate Price and Purchase Limit (Performer)

1. Read the product's listed price (USD) from SauceDemo.
2. Call the Frankfurter REST API (HTTP request) for the USD → Preferred Currency rate (USD, EUR, or GBP). No conversion is needed for USD.
3. Parse the JSON response and compute the converted price.
4. Compare the converted price with the **Purchase Limit** (expressed in the Preferred Currency).
5. **Reject** with a clear reason if the price exceeds the limit or any required business data is invalid.
6. Proceed to checkout only if validation succeeds.

> The budget check uses the product's **listed price** only. Shipping/tax shown at checkout is captured separately where applicable.

### 4. Complete Approved Test Purchases (Performer)

1. Add the matched product to the cart.
2. Open the cart and verify the selected item.
3. Proceed to checkout and enter the employee's **First Name**, **Last Name**, and **Postal Code**.
4. Review the order and finish checkout.
5. Capture the confirmation message as proof of success.
6. Reset the cart/session so the next transaction starts clean.

A completed checkout is **never repeated**, even if a later reporting or email step fails.

### 5. Update the Excel Processing Report

One row per Request ID, updated after each transaction. See [Output Report Specification](#output-report-specification).

### 6. Send Email Notifications

| Email | Recipient | Content |
|---|---|---|
| Confirmation | Employee | Request ID, product name, converted price, currency, final status |
| Rejection | Employee | Request ID and rejection reason |
| Summary | Procurement Operations mailbox | Totals for eligible, completed, rejected, and failed requests, with the Excel report attached |

All recipients and mail settings are configurable.

## Input File Specification

File: `PurchaseRequests.xlsx`

| Column | Description |
|---|---|
| Request ID | Unique identifier of the request |
| Employee Name | Full name of the requester |
| Employee Email | Notification recipient |
| First Name | Used at checkout |
| Last Name | Used at checkout |
| Postal Code | Used at checkout |
| Product Name | Product to look up in SauceDemo |
| Preferred Currency | `USD`, `EUR`, or `GBP` |
| Purchase Limit | Budget, in the Preferred Currency |
| Status | Request status; only `Pending` is processed |
| Priority | `High`, `Medium`, or `Low` |

## Output Report Specification

| Column | Description |
|---|---|
| Request ID | Link back to the original request |
| Employee Name | |
| Employee Email | |
| Product Name | |
| Original Price (USD) | Listed SauceDemo price |
| Preferred Currency | |
| Exchange Rate | USD → Preferred Currency |
| Converted Price | Price in the preferred currency |
| Purchase Limit | |
| Final Status | `Completed`, `Rejected`, or `Failed` |
| Status Reason | Meaningful reason for rejection or failure |
| Confirmation Details | SauceDemo checkout confirmation |
| Processing Timestamp | When the transaction finished |

## Configuration

All environment-specific values live in a configuration file (e.g. `Config.xlsx`) and/or Orchestrator assets, with nothing hard-coded.

| Setting | Description |
|---|---|
| Input file path | Location of `PurchaseRequests.xlsx` |
| Report file path | Location of the output processing report |
| Queue name | `PurchaseRequestsQueue` |
| SauceDemo URL | `https://www.saucedemo.com/` |
| SauceDemo credentials | Orchestrator Credential asset |
| Exchange-rate API base URL | `https://api.frankfurter.dev` |
| Mail server / SMTP settings | Host, port, sender (if SMTP is used) |
| Procurement Operations mailbox | Summary email recipient |
| Email templates | Subjects and bodies for confirmation / rejection / summary |

## Business Rules & Exception Handling

| Scenario | Outcome |
|---|---|
| Product not found in SauceDemo | **Rejected** – product unavailable |
| Converted price exceeds Purchase Limit | **Rejected** – limit exceeded |
| Invalid business data (e.g. unsupported currency, non-numeric or negative limit, missing name or postal code) | **Rejected** – invalid data, with the specific reason |
| API unreachable or malformed response, application error, or timeout | **Failed** – system exception, with retry per queue settings |
| Checkout succeeds but a later email or report step fails | Status stays **Completed**; only the failed step is retried |

Business exceptions (Rejected) are not retried. System exceptions (Failed) are retried according to the queue's retry setting, and the browser and application are re-initialized between attempts.

## Idempotency & Rerun Safety

- `Request ID` is the queue **Reference**, so Orchestrator rejects duplicate items.
- Before queueing, the Dispatcher checks the report for Request IDs already `Completed` and skips them.
- The report is keyed by Request ID, so rerunning updates rows rather than appending duplicates.
- Checkout is not retried once a confirmation has been captured.

## Getting Started

### Prerequisites

- UiPath Studio and access to UiPath Orchestrator
- An unattended robot/machine assigned to the folder containing `PurchaseRequestsQueue`
- Microsoft Excel, plus Outlook or SMTP access
- A browser supported by UiPath (e.g. Chrome or Edge)
- Internet access to `saucedemo.com` and `api.frankfurter.dev`

### Setup

1. Clone the repository:
   ```bash
   git clone <your-repo-url>
   cd <your-repo-name>
   ```
2. Create the `PurchaseRequestsQueue` queue in Orchestrator (enable **Unique Reference**, set the retry count).
3. Create the SauceDemo credential asset in Orchestrator.
4. Update the configuration file with your paths and mail settings.
5. Place `PurchaseRequests.xlsx` in the configured input location.
6. Open the project in UiPath Studio and publish to Orchestrator.

### Running

1. Run the **Dispatcher** to filter, sort, and queue pending requests.
2. Run the **Performer** to process the queue.
3. Run the **Reporter** (or the final stage of the Performer) to finalize the Excel report and send the summary email.

## Project Structure

> Adjust to match your actual repository layout.

```
├── Dispatcher/            # Read, filter, sort, upload to queue
├── Performer/             # SauceDemo, API calls, checkout
├── Reporter/              # Report finalization, summary email
├── Framework/             # Shared workflows (init, config, logging, close apps)
├── Data/
│   ├── Config.xlsx
│   ├── PurchaseRequests.xlsx
│   └── ProcessingReport.xlsx
├── Documentation/
└── README.md
```

## Testing Scenarios

| # | Scenario | Expected result |
|---|---|---|
| 1 | Valid request, price within limit (USD) | Completed, confirmation email sent |
| 2 | Valid request, EUR or GBP, within limit | Completed, converted price and rate recorded |
| 3 | Price exceeds limit | Rejected, rejection email sent |
| 4 | Product not in SauceDemo | Rejected, reason recorded |
| 5 | Invalid data (bad currency, missing field) | Rejected, reason recorded |
| 6 | Mixed priorities (Low listed before High) | Processed High → Medium → Low, ties in input order |
| 7 | Status not Pending | Ignored |
| 8 | Duplicate Request ID in input | Logged as duplicate and skipped |
| 9 | Rerun after a completed run | No duplicates in queue, report, or checkouts |
| 10 | API unavailable | Failed, retried, reason recorded |
| 11 | Email step fails after checkout | Checkout not repeated |
| 12 | Back-to-back transactions | Each starts with an empty cart and a clean session |

## Notes

- SauceDemo is a public test site, so no real purchases are made.
- Purchase Limit is always compared in the employee's Preferred Currency.
