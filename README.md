# ACME Client Security Hash: Performer

A UiPath REFramework process that takes work items of type WI5 from an Orchestrator queue, computes each client's security hash on SHA1 Online, and completes the work item in ACME System 1 with the hash as its comment. It is the Performer half of a Dispatcher/Performer solution based on the UiPath Academy exercise "Calculate Client Security Hash". A separate Dispatcher project fills the queue.

## What it does

For every queue item:

1. Opens the work item's details page in ACME and reads Client ID, Client Name and Client Country.
2. Builds the text `ClientID-ClientName-ClientCountry`, each value trimmed.
3. Enters that text on SHA1 Online and reads the resulting hash.
4. Opens the work item's update form, adds the hash as a comment and sets the status to `Completed`.
5. Confirms ACME's "updated" alert, reloads the form and checks that the saved status and comment are the expected ones.

The run ends when the queue is empty. ACME is logged out and both browser tabs are closed, also when a step fails.

## Project structure

| Workflow | Purpose |
|---|---|
| `Main.xaml` | REFramework state machine (template, with one added email step) |
| `Framework\InitAllApplications.xaml` | Logs in to ACME; a rejected login is emailed and ends the run |
| `Framework\Process.xaml` | The steps above for one queue item, and the business rules |
| `Framework\CloseAllApplications.xaml` | Closes SHA1 Online and ACME |
| `Framework\KillAllProcesses.xaml` | Kills Chrome, only if `KillChromeOnRecovery` is `TRUE` |
| `Framework\SendExceptionEmail.xaml` | Sends an exception email over SMTP, or logs it when email is not configured |
| `ACME\ACME_Login.xaml` | Open the browser, read the credential, log in, confirm by the Log Out link |
| `ACME\ACME_GetClientDetails.xaml` | Open a work item by ID and read its two text blocks |
| `ACME\ACME_UpdateWorkItem.xaml` | Fill the update form, submit, handle the alert, read the saved values |
| `ACME\ACME_Close.xaml` | Log out and close the ACME tab |
| `SHA1\SHA1_GetHash.xaml` | Compute a hash on SHA1 Online |
| `SHA1\SHA1_Close.xaml` | Close the SHA1 Online tab |
| `Process\ParseWorkItemDetails.xaml` | Turn the text read from ACME into fields |
| `Process\BuildClientHashInput.xaml` | Validate the three client fields and join them |

Workflows under `ACME\` and `SHA1\` only drive the user interface. Workflows under `Process\` hold rules and never touch the UI, so they can be tested without a browser. `Framework\Process.xaml` connects the two.

## Requirements

- UiPath Studio 2026 (Windows project, VB expressions), Chrome with the UiPath extension
- Packages: `UiPath.System.Activities` 26.8.2, `UiPath.UIAutomation.Activities` 26.10.4, `UiPath.Mail.Activities` 2.12.10, `UiPath.Excel.Activities` 3.6.1, `UiPath.Testing.Activities` 25.10.3
- An ACME System 1 account at https://acme-test.uipath.com/

Orchestrator objects, all in the same folder:

| Type | Name | Notes |
|---|---|---|
| Queue | `ACME_ClientSecurityHash` | Filled by the Dispatcher. Unique Reference = Yes, Auto Retry = Yes, max 2 retries |
| Credential asset | `ACME_Credential` | ACME email and password |
| Text asset | `ACME_URL` | `https://acme-test.uipath.com/` |
| Credential asset | `SMTP_Credential` | Sending mailbox and its password. Needed only when exception emails are switched on |

## Configuration

All settings are in `Data\Config.xlsx`. The ones this process adds to the template:

| Setting | Value | Meaning |
|---|---|---|
| `OrchestratorQueueName` | `ACME_ClientSecurityHash` | Queue to read |
| `ACME_CredentialAssetName` | `ACME_Credential` | Asset holding the ACME login |
| `ACME_WorkItemType` | `WI5` | Only this type is processed |
| `ACME_CompletedStatus` | `Completed` | Status set after the hash is added |
| `ACME_UpdateSuccessText` | `Work Item was updated accordingly` | Alert text that confirms an update |
| `SHA1_URL` | `http://www.sha1-online.com/` | Hash generator address |
| `SHA1_Algorithm` | `sha-1` | Item selected in the algorithm dropdown |
| `ExceptionEmail_To` | empty | Recipient of exception emails |
| `ExceptionEmail_LoginFailedBody` | text from the PDD | Body of the rejected-login email |
| `SMTP_Server`, `SMTP_Port` | empty, `587` | Mail server |
| `SMTP_CredentialAssetName` | `SMTP_Credential` | Asset holding the sending account |
| `KillChromeOnRecovery` | `FALSE` | Whether Chrome may be force-killed |
| `MaxRetryNumber` (Constants) | `0` | Retries are left to the queue |

**Exception emails are off until configured.** While `ExceptionEmail_To` or `SMTP_Server` is empty, the message that would have been sent is written to the log as a warning. Fill both and create the `SMTP_Credential` asset to switch sending on.

## Queue item contract

Defined by the Dispatcher. All values are text.

| Field | Used for |
|---|---|
| Reference | Work item ID; shown as the transaction ID in logs |
| `WIID` | Work item ID; the Performer opens `…/work-items/<WIID>` with it |
| `Type` | Must equal `ACME_WorkItemType` |
| `Description`, `Status`, `Date` | Not used |

## Exception handling

| Case | Kind | Result |
|---|---|---|
| ACME rejects the login | Business, at start | Email with the PDD text; the run ends without taking a queue item |
| Queue is empty | none | The run ends normally |
| Queue item has no WIID or the wrong type | Business | Item marked Business Exception |
| Work item does not exist (ACME shows Not Found) | Business | Item marked Business Exception |
| A client field is empty | Business | Item marked Business Exception |
| Work item is already Completed, first attempt | Business | Item marked Business Exception; the work item is not changed |
| Work item is already Completed, on a retry | none | Item marked Successful: an earlier attempt saved the update |
| Page does not load, element not found, result is not 40 hexadecimal characters, update not confirmed or not saved | System | Screenshot, email with the screenshot, applications closed and reopened, queue retries the item up to 2 times |
| The run stops on a system exception | System | Email |

## Design decisions

**Work item pages are opened by address.** The details page is `…/work-items/<WIID>` and the update form is `…/work-items/update/<WIID>`. In ACME the Update Work Item button opens the form in a JavaScript pop-up window; opening the address in the same tab avoids pop-up blocking and a second window.

**An update is verified, not assumed.** ACME saves the update in the background and reports it only through a browser alert. The workflow reads the alert text, then reloads the form and compares the saved status and comment with what it sent.

**Safe to retry.** A retried item whose work item is already Completed is accepted, so a failure after the save does not produce a second comment or a false error.

**A stale hash cannot be reused.** SHA1 Online keeps the previous result on the page. The workflow loads a fresh page before every calculation and checks that the result is 40 hexadecimal characters.

**The client fields are parsed from text.** ACME shows the three client values inside one paragraph, not in separate fields, with padding around each value. `ParseWorkItemDetails` extracts them by label and trims them.

**Retries belong to the queue.** `MaxRetryNumber` is 0 and the queue's Auto Retry does the retrying, so the retry count is visible in Orchestrator and survives a restart of the robot.

**Chrome is not killed by default.** `KillChromeOnRecovery` is `FALSE` so that a run on a developer machine never closes the person's own browser. Set it to `TRUE` on an unattended robot. With `FALSE`, a tab that could not be closed is only logged.

**Selectors live in the Object Repository** (applications `ACME` and `SHA1 Online`). All cards use simulated input and take their address from an argument.

## Tests

| Test case | What it covers | Side effects |
|---|---|---|
| `Tests\ClientHashInputTestCase.xaml` | Parsing and hash-input rules, with made-up data | None; can be run any time |
| `Tests\ProcessTestCase.xaml` | `Process.xaml` end to end for one work item, passed as `in_TestWIID` | Completes that work item in ACME. It does not take an item from the queue |

The other test cases are the unchanged REFramework templates.

## Limits

- The process depends on a public website (SHA1 Online). If it is unavailable, every item fails as a system exception.
- `MaxConsecutiveSystemExceptions` is 0 (disabled), as in the template. With the website down, the run would try every queue item and send one email per failed attempt. Set a limit such as 3 to stop early.
- Each successful run completes work items for good. To run it again on the same data: in ACME choose User options → Reset test data, delete the old queue items, and run the Dispatcher.
