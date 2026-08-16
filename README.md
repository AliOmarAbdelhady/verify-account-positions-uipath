# Verify Account Positions — UiPath RPA Solution

An end-to-end UiPath automation that reconciles account positions between **ACME System 1** (web) and **ACME System 3** (desktop). For every *Verify Account Position* (WI1) work item, the solution sums all transactions of the referenced account in System 3, compares the total against the amount recorded in System 1, and completes the work item with the verified result.

The solution is split into two REFramework projects:

| Project | Role |
|---|---|
| `RAYA_UiPath_VerifyAccountPositions_Dispatcher` | Logs into ACME System 1, extracts every open WI1 work item (all pages), and uploads them to an Orchestrator queue. |
| `RAYA_UiPath_VerifyAccountPositions_Performer` | Consumes the queue, searches the client and account in ACME System 3, sums the account transactions, compares the values, and updates the work item in ACME System 1. |

Both projects target **Windows / C#** with modern UI Automation activities and build cleanly in UiPath Studio 2026.

---

## Business Rules

For each WI1 work item carrying a Client ID, Account Number and Account Amount:

1. Locate the client in ACME System 3 (Clients → Search For Client → By Client ID, including inactive clients).
2. Open the account whose number exactly matches the work item.
3. Read all account transactions and sum their amounts using decimal arithmetic.
4. Compare the two values:

   ```
   difference    = system_1_amount − system_3_sum   (signed)
   amounts_match = (system_1_amount == system_3_sum)
   ```

5. Update the work item in ACME System 1 with status **Completed** and the corresponding comment:

   | Result | Status | Comment |
   |---|---|---|
   | Values match | `Completed` | `Account value matches` |
   | Values differ | `Completed` | `Account has difference of <signed difference>` |

A mismatch is a valid business outcome, not an exception — both branches complete the work item.

---

## Solution Architecture

```
ACME System 1 (web)                      Orchestrator                     ACME System 3 (desktop)
┌─────────────────────┐   extract WI1   ┌──────────────────┐   consume   ┌─────────────────────────┐
│  Work Items (WI1)   │ ──────────────► │ VerifyAccount    │ ─────────► │  Client / Account /     │
│  Client ID          │                 │ Positions_WI1    │            │  Transactions           │
│  Account Number     │                 │ (unique refs)    │            │  (sum of movements)     │
│  Account Amount     │ ◄────────────── │                  │ ◄───────── │                         │
└─────────────────────┘   update item   └──────────────────┘  result    └─────────────────────────┘
        ▲                                  (queue)                        Performer: verify + update
        └── Dispatcher                     retry / audit log             status Completed + comment
```

**Dispatcher** — tabular-data REFramework. On the first transaction it navigates the work-item list (all pages), keeps the rows whose type matches the configured `WI_Type`, opens each detail page to read Client ID, Account Number, Account Amount and Currency, writes `Data/Output/DispatchedWorkItems.csv`, and then dispatches one queue item per row using the WIID as the unique reference (re-runs skip already-queued references idempotently).

**Performer** — queue-driven REFramework. For every queue item it validates the required fields, drives ACME System 3 (login, client search by ID, open the matching account, show all transactions), extracts and sums the amounts, compares against the System 1 amount, and updates the work item. Each result is appended to `Data/Output/TransactionResults.jsonl` as an audit record.

---

## Prerequisites

- UiPath Studio / Robot 2026+ (Windows, modern UI Automation)
- Access to ACME System 1 (`https://acme-test.uipath.com`) and ACME System 3 Desktop
- An Orchestrator tenant with a folder for the automation

## Orchestrator Setup

1. **Queue** — create `VerifyAccountPositions_WI1` with *unique references* enabled.
2. **Assets** — create the following text assets in the automation folder:

   | Asset | Value |
   |---|---|
   | `ACME_Username` | ACME account email |
   | `ACME_Password` | ACME account password |
   | `System1_URL` | `https://acme-test.uipath.com` |
   | `DataFolderPath` | `Data` |
   | `ScreenshotsFolderPath` | `Screenshots` |
   | `Mail_Server` / `Mail_PortNumber` | SMTP server / port for exception e-mails |
   | `SenderMail_Email` / `SenderMail_Password` | SMTP sender credentials |
   | `ExceptionMail_Reciever` / `ExceptionMail_Subject` | exception mailbox / subject |

   Credentials must be stored as assets (or Credential Manager) — never in the repository.

## Configuration

Each project reads `Data/Config.xlsx`:

| Setting | Value |
|---|---|
| `OrchestratorQueueName` | `VerifyAccountPositions_WI1` |
| `OrchestratorQueueFolder` | your Orchestrator folder |
| `WI_Type` | `WI1` |
| `BrowserType` | `Edge` |
| `ACMEDesktopExecutablePath` *(Performer)* | full path to `ACME-System3.exe` |
| `MaxRetryNumber` | transaction retry count (Performer ships with `0` for development — raise for production) |

## Running

**From Studio** — open either project and run `Main.xaml` (Dispatcher first, then Performer).

**From the CLI** —

```bash
uip rpa run --file-path Main.xaml --project-dir RAYA_UiPath_VerifyAccountPositions_Dispatcher
uip rpa run --file-path Main.xaml --project-dir RAYA_UiPath_VerifyAccountPositions_Performer
```

**From Orchestrator** — publish both projects, create a process for each in the automation folder, and start the Dispatcher job followed by the Performer job.

---

## Output

| File | Produced by | Content |
|---|---|---|
| `Dispatcher/Data/Output/DispatchedWorkItems.csv` | Dispatcher | One row per extracted WI1 item: WIID, Type, Description, Status, Date, ClientID, AccountNumber, AccountAmount, Currency. |
| `Performer/Data/Output/TransactionResults.jsonl` | Performer | One JSON record per processed item: work item id, client and account identifiers, both amounts, signed difference, match flag, final status, comment and timestamp. |

Both folders are committed with the expected output of a successful run against the live ACME environment, so the file formats are self-documenting. A new run overwrites the CSV and appends fresh JSON records.

## Testing

Each project ships with workflow test cases runnable from Studio or the CLI:

```bash
uip rpa run --file-path Tests/InitAllSettingsTestCase.xaml --project-dir RAYA_UiPath_VerifyAccountPositions_Dispatcher
uip rpa run --file-path Tests/InitAllSettingsTestCase.xaml --project-dir RAYA_UiPath_VerifyAccountPositions_Performer
uip rpa run --file-path Tests/CheckAccountPositionTestCase.xaml --project-dir RAYA_UiPath_VerifyAccountPositions_Performer
```

- **InitAllSettingsTestCase** verifies the configuration loads and the queue name / work-item type resolve to `VerifyAccountPositions_WI1` / `WI1`.
- **CheckAccountPositionTestCase** covers the comparison logic: exact-equality match, positive signed difference and negative signed difference.

## Project Structure

```
├── README.md
├── RAYA_UiPath_VerifyAccountPositions_Dispatcher/
│   ├── Main.xaml                     # REFramework state machine
│   ├── Framework/                    # InitAllSettings, GetTransactionData, Process, …
│   ├── Workflows/
│   │   ├── ACMEWeb_ExtractWorkItems.xaml
│   │   └── Dispatcher_DispatchWorkItems.xaml
│   ├── Data/Config.xlsx              # settings, constants, asset map
│   ├── Data/Output/                  # dispatcher run output (CSV)
│   ├── Tests/                        # workflow test cases
│   └── .objects/                     # captured UI descriptors (Object Repository)
└── RAYA_UiPath_VerifyAccountPositions_Performer/
    ├── Main.xaml
    ├── Framework/
    ├── Workflows/
    │   ├── ACMEDesktop_Login.xaml
    │   ├── ACMEDesktop_NavigateToClientSearch.xaml
    │   ├── ACMEDesktop_SumAccountTransactions.xaml
    │   ├── ACMEWeb_UpdateWorkItem.xaml
    │   ├── Control_CheckAccountPosition.xaml
    │   └── Email_SendExceptionMail.xaml
    ├── Data/Config.xlsx
    ├── Data/Output/                  # performer run output (JSONL)
    ├── Tests/
    └── .objects/
```
