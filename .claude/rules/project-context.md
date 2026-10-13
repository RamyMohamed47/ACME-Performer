<!-- discovery-metadata: cs=0 xaml=26 deps=5 -->
# Project context: ACME_ClientHash_Performer

Performer half of a Dispatcher/Performer solution against ACME System 1 (UiPath Academy "Calculate Client Security Hash"). The Dispatcher is a separate project (`D:\UiPath\ACME_Dispatcher\ACME_ClientHash_Dispatcher`).

## Project

- Process built on the REFramework template, XAML only, VisualBasic expressions, `targetFramework: Windows`, Studio 26.0.203.
- Dependencies: `UiPath.System.Activities` 26.8.2, `UiPath.UIAutomation.Activities` 26.10.4, `UiPath.Mail.Activities` 2.12.10, `UiPath.Excel.Activities` 3.6.1, `UiPath.Testing.Activities` 25.10.3.
- Orchestrator: queue `ACME_ClientSecurityHash` (unique reference, auto retry, max 2), credential asset `ACME_Credential`, text asset `ACME_URL`, optional credential asset `SMTP_Credential`.

## Layout

| Path | Purpose |
|---|---|
| `Main.xaml`, `Framework\*` | REFramework. Customised: `InitAllApplications`, `Process`, `CloseAllApplications`, `KillAllProcesses`, `GetTransactionData` (ID = reference), `SetTransactionStatus` and `Main` (email hooks, Config passed to Close/Kill). `SendExceptionEmail` is new. |
| `ACME\*` | UI only: login, get client details, update work item, close |
| `SHA1\*` | UI only: get hash, close |
| `Process\*` | Rules only, no UI: `ParseWorkItemDetails`, `BuildClientHashInput` |
| `Tests\*` | `ClientHashInputTestCase` (no side effects), `ProcessTestCase` (completes the work item given in `in_TestWIID`), plus unchanged templates |
| `Data\Config.xlsx` | All settings |

## Conventions

- Arguments: `in_` / `out_` / `io_`, letters only with digits allowed at the end (Analyzer pattern; `in_Sha1Url` is rejected). Variables: camelCase.
- UI workflows hold no business rules; `Process\` workflows never touch the UI.
- Each UI workflow has its own Use Application/Browser card with `InteractionMode="Simulate"`. Login and `SHA1_GetHash` open if not open; the others attach only; the Close workflows close.
- Card addresses are bound to arguments (`[in_AcmeUrl]`, `[in_HashSiteUrl]`). Linking a screen rewrites the address to a literal; rebind it afterwards.
- No hard-coded credentials. Get Credential runs in the Sequence where the password is used; SecureString is never an argument (ST-SEC-007/008/009).
- Bad data raises `BusinessRuleException`; everything else is a system exception. `Process.xaml` has no catch-all.
- No literal `[` or `]` inside a VB `[expression]` (regexes use `(?: |\t)*`, not character classes).
- Logging: Info for workflow start/end and outcomes, Warn for skipped or unsent actions, Error for failures. Client names are not logged.

## ACME and SHA1 Online facts

- Details page: `<ACME_URL>/work-items/<WIID>`. Update form: `<ACME_URL>/work-items/update/<WIID>` (in the UI it is a JavaScript pop-up; a simulated click on the button does nothing).
- Client ID, Name and Country are bare text in one paragraph; read the paragraph and parse by label.
- Submitting the update form posts in the background and then shows a browser alert; the page does not refresh. `ACME_UpdateWorkItem` uses Browser Dialog Scope and reloads the form to verify.
- An unknown WIID gives a page titled `Not Found` (own Object Repository screen).
- A rejected login stays on `/login` and shows an alert span under the Email box.
- The update form has no navigation bar, so no Log Out link; `ACME_Close` goes to the dashboard first.
- SHA1 Online keeps the previous result on the page; `SHA1_GetHash` loads a fresh page first.

## Selectors

Object Repository applications `ACME` (screens `Login`, `Any Page`, `Work Items`, `Not Found`) and `SHA1 Online` (screen `Hash Generator`), captured with the UIAutomation `uia-configure-target` flow against live Chrome. Never hand-write selector XML. `.screenshots/` and `.objects/**/.images/` are git-ignored because thumbnails can show the login email.

## Tooling notes

- Config.xlsx is edited by patching its XML in place so formatting is preserved; do not re-save it through a spreadsheet library.
- `uip rpa build` deletes the tracked `entry-points.json`; restore it with `git checkout -- entry-points.json` before committing.
- Run UI workflows with `uip rpa run` through the open Studio, which supplies the Orchestrator connection.
