# Financial Services App Template — Build Guide

This is the financial-services sibling to the Real Estate App Template — same method, same VS Code + Salesforce CLI workflow, built for Abuja investment firms and microfinance banks instead of property agencies. If you've already built the real estate one, most of this will feel familiar; the parts that differ are called out as they come up.

**Who this is for:** Relationship-manager-led teams — investment firms managing client portfolios, microfinance banks managing loan books, or firms doing both. Objects, apps, and flows get built by clicking around in Salesforce Setup, same as before, then pulled into a code project with one CLI command. Reusable, redeployable, version-controlled.

**One thing that's different from real estate:** this template has two halves that don't fully overlap — Investments and Loans behave differently enough (one matures once, the other repays in installments) that they're separate objects, not one object with a "type" field. A firm that only does one or the other only gets that half deployed to their org. More on this in Part 15.

---

## Part 1 — Install Your Tools (one-time only)

Skip this if you already did it for the real estate template — same tools, same machine.

1. **Node.js** — [nodejs.org](https://nodejs.org), LTS version. Check with `node -v`.
2. **VS Code** — [code.visualstudio.com](https://code.visualstudio.com).
3. **Salesforce CLI**:
   ```
   npm install --global @salesforce/cli
   sf --version
   ```
4. **Salesforce Extension Pack** in VS Code (Extensions sidebar → search → Install).

---

## Part 2 — Create a Developer Org

A fresh org to prototype in — same as real estate, don't build this against a live client org first.

1. [developer.salesforce.com/signup](https://developer.salesforce.com/signup)
2. Pick a unique username, confirm the activation email, set a password
3. Log in once to confirm it works, then close the tab

---

## Part 3 — Create the Project

1. Open VS Code, open a terminal (**Terminal → New Terminal**)
2. Navigate to where you keep projects, then:
   ```
   sf project generate --name apexflow-financial-services-template
   cd apexflow-financial-services-template
   code .
   ```

---

## Part 4 — Connect VS Code to Your Org

1. `Cmd/Ctrl+Shift+P` → **"SFDX: Authorize an Org"**
2. Login URL: **Production** (correct for Developer Edition)
3. Alias: `apexflow-fs-template`
4. Log in in the browser tab that opens, click **Allow**
5. Confirm the alias shows in the bottom-left status bar

---

## Part 5 — Set Up Git

```
git init
git add .
git commit -m "Initial project"
```

Then connect it to the repo:

```
git remote add origin https://github.com/FirstVaryen/Financial-services-Template-Apex-Flows-.git
git branch -M main
git push -u origin main
```

Commit after every Part from here on:
```
git add .
git commit -m "describe what you just added"
```

---

## Part 6 — Your First Object: Financial_Product__c

This is the catalog — the menu of things a firm actually offers ("12-Month Fixed Deposit," "SME Working Capital Loan"). Every Investment and Loan a client holds points back to one of these.

1. Setup → **Object Manager** → **Create → Custom Object**
2. **Label:** `Financial Product`, **Plural Label:** `Financial Products` — Object Name fills in as `Financial_Product__c`
3. Save, then add these fields under **Fields & Relationships**:

   | Field Label | Type | Notes |
   |---|---|---|
   | Product Category | Picklist | Values: Investment, Loan |
   | Product Type | Picklist | Values: Fixed Deposit, Treasury Bill, Mutual Fund, Recurring Savings, Working Capital Loan, Asset Finance Loan, Salary Advance Loan |
   | Interest Rate | Percent (5, 2) | Annualized |
   | Minimum Amount | Currency (16, 2) | |
   | Tenor (Months) | Number (3, 0) | |
   | Description | Long Text Area | |
   | Active | Checkbox | Default checked |

4. Pull it in:
   ```
   sf project retrieve start --metadata CustomObject:Financial_Product__c
   git add . && git commit -m "Add Financial_Product__c object"
   ```

---

## Part 7 — Relationship_Manager__c, and Extending Account & Contact

**Relationship_Manager__c** — the person a client actually deals with. Same steps as Part 6:

| Field Label | Type | Notes |
|---|---|---|
| Branch | Lookup Relationship → **Account** | The firm/branch they work out of |
| Staff ID | Text (20) | |
| Phone | Phone | |
| Email | Email | |
| Portfolio Target (Annual) | Currency (16, 2) | For reporting later |

**Account** (extending, not creating):

| Field Label | Type | Notes |
|---|---|---|
| Institution Type | Picklist | Values: Investment Firm, Microfinance Bank, Both |
| CBN License Number | Text (30) | |

**Contact** (extending — the client):

| Field Label | Type | Notes |
|---|---|---|
| BVN | Text (11) | Bank Verification Number |
| Risk Profile | Picklist | Values: Conservative, Moderate, Aggressive |
| Annual Income Bracket | Text (40) | |

Pull and commit:
```
sf project retrieve start --metadata CustomObject:Relationship_Manager__c CustomObject:Account CustomObject:Contact
git add . && git commit -m "Add Relationship_Manager__c and extend Account/Contact"
```

---

## Part 8 — Investment__c and Loan__c

Two objects, not one. An investment matures once; a loan repays in installments and can be renewed. Trying to force them into one object with a "type" field means a pile of fields that are blank on half your records. Keep them separate.

Both are **Master-Detail to Financial_Product__c** (same "when you delete the product, the holding goes with it" logic real estate used for Property → Viewing/Transaction).

**Investment__c**

| Field Label | Type | Notes |
|---|---|---|
| Product | Master-Detail Relationship → Financial_Product__c | |
| Client | Lookup Relationship → Contact | |
| Relationship Manager | Lookup Relationship → Relationship_Manager__c | |
| Principal Amount | Currency (16, 2) | |
| Start Date | Date | |
| Maturity Date | Date | |
| Expected Return | Currency (16, 2) | |
| Status | Picklist | Values: Active, Matured, Rolled Over, Withdrawn |

**Loan__c**

| Field Label | Type | Notes |
|---|---|---|
| Product | Master-Detail Relationship → Financial_Product__c | |
| Client | Lookup Relationship → Contact | |
| Relationship Manager | Lookup Relationship → Relationship_Manager__c | |
| Principal Amount | Currency (16, 2) | |
| Disbursement Date | Date | |
| Tenor (Months) | Number (3, 0) | |
| Outstanding Balance | Currency (16, 2) | |
| Status | Picklist | Values: Pending Disbursement, Active, Repaid, In Arrears, Defaulted |
| Guarantor | Lookup Relationship → Contact | Separate from the borrower |

Pull and commit:
```
sf project retrieve start --metadata CustomObject:Investment__c CustomObject:Loan__c
git add . && git commit -m "Add Investment__c and Loan__c"
```

---

## Part 9 — Repayment__c and Compliance_Document__c

**Repayment__c** — a loan's actual installment schedule. Master-Detail to Loan__c.

| Field Label | Type | Notes |
|---|---|---|
| Loan | Master-Detail Relationship → Loan__c | |
| Due Date | Date | |
| Amount Due | Currency (16, 2) | |
| Amount Paid | Currency (16, 2) | |
| Status | Picklist | Values: Upcoming, Paid, Overdue |

**Compliance_Document__c** — KYC paperwork. This one links to the *client* (Contact), not a product, so it's a Lookup, not Master-Detail — Salesforce won't let you master-detail to a standard object.

| Field Label | Type | Notes |
|---|---|---|
| Client | Lookup Relationship → Contact | |
| Document Type | Picklist | Values: BVN Slip, Utility Bill, Government ID, Board Resolution, Signature Mandate |
| Status | Picklist | Values: Pending, Verified, Expired |
| Verified By | Lookup Relationship → User | |

Pull and commit:
```
sf project retrieve start --metadata CustomObject:Repayment__c CustomObject:Compliance_Document__c
git add . && git commit -m "Add Repayment__c and Compliance_Document__c"
```

---

## Part 10 — Build the App Shell

Build this as a **Console app** from the start, not a Standard app — real estate started Standard and had to be rebuilt as a Console app partway through to get closable multi-tab navigation, which just wastes a step. Do it right the first time.

1. Setup → **App Manager** → **New Lightning App**
2. **App Name:** `Financial Services App Template`
3. On the app type step, choose **Console** (not the default Standard)
4. Navigation Items: Financial Products, Relationship Managers, Accounts, Contacts, Investments, Loans, Repayments, Compliance Documents
5. User Profiles: at least System Administrator
6. Save

Pull and commit:
```
sf project retrieve start --metadata CustomApplication:Financial_Services_App_Template
git add . && git commit -m "Add app shell and tabs"
```

---

## Part 11 — Home Page

1. **App Builder** → **New → Home Page**, name it `Financial Services Home`
2. Add a Rich Text header
3. Consider a Dashboard component here too, once Part 14's reports exist — you can always come back and add it later, same as real estate did
4. Save, **Activation** → set as the App Default for Financial Services App Template

Pull and commit:
```
sf project retrieve start --metadata FlexiPage
git add . && git commit -m "Add home page"
```

---

## Part 12 — A Simple Lightning Web Component

Same idea as real estate's `propertyList` — a small component showing something useful on Home, written as code from the start.

1. **SFDX: Create Apex Class**, name it `PortfolioController`:
   ```apex
   public with sharing class PortfolioController {
       @AuraEnabled(cacheable=true)
       public static List<Investment__c> getRecentInvestments() {
           return [
               SELECT Name, Client__r.Name, Principal_Amount__c, Maturity_Date__c, Status__c
               FROM Investment__c
               ORDER BY CreatedDate DESC
               LIMIT 20
           ];
       }
   }
   ```
2. **SFDX: Create Lightning Web Component**, name it `recentInvestments`. `.js`:
   ```javascript
   import { LightningElement, wire } from 'lwc';
   import getRecentInvestments from '@salesforce/apex/PortfolioController.getRecentInvestments';

   export default class RecentInvestments extends LightningElement {
       @wire(getRecentInvestments) investments;
   }
   ```
3. `.html`:
   ```html
   <template>
       <lightning-card title="Recent Investments" icon-name="standard:investment_account">
           <template for:each={investments.data} for:item="inv">
               <div key={inv.Id} class="slds-p-around_small">
                   {inv.Name} — {inv.Client__r.Name} — {inv.Status__c}
               </div>
           </template>
       </lightning-card>
   </template>
   ```
4. Deploy both (right-click each → **SFDX: Deploy Source to Org**), then drag the component onto the Home page in App Builder

Commit:
```
git add . && git commit -m "Add recentInvestments LWC and controller"
```

---

## Part 13 — Sample Data

Same two options as real estate. Simplest: **Data Import Wizard**, one CSV per object. More durable: build it as SObject Tree JSON files plus a `data-plan.json`, and load with:
```
sf data import tree --plan data/data-plan.json
```

Worth building 2–3 sample Accounts (a mix of Investment Firm / Microfinance Bank / Both), a handful of Relationship Managers, some clients, and a spread of Investments and Loans — enough to make the reports in Part 15 mean something. Abuja-appropriate names, Naira amounts, CBD/Maitama addresses.

---

## Part 14 — Two Flows: Maturity Reminder and Repayment Reminder

Real estate had one reminder flow (Viewing Reminder). This one needs two — an investment maturing and a loan repayment coming due aren't the same kind of event, so one flow trying to cover both would end up messier than just writing two:

**Maturity Reminder** — fires when an Investment's maturity date is approaching (a scheduled/time-based flow, not record-triggered, since it needs to check dates on existing records rather than react to a create):
1. Setup → **Flows** → **New Flow** → **Schedule-Triggered Flow**
2. Object: `Investment__c`, run daily, entry condition: `Maturity_Date__c` = TODAY + 30
3. **Create Records**: a Task, subject "Investment Maturing Soon," assigned to the Relationship Manager's linked user
4. Save as `Investment Maturity Reminder`, Activate

**Repayment Reminder** — same idea, checking `Repayment__c.Due_Date__c` a few days out, creating a Task for the Relationship Manager on the parent Loan.

Pull and commit:
```
sf project retrieve start --metadata Flow:Investment_Maturity_Reminder Flow:Repayment_Reminder
git add . && git commit -m "Add maturity and repayment reminder flows"
```

---

## Part 15 — Permission Set, Reports, and Wrapping It Up

**Permission Set** — same pattern as real estate's Real Estate Agent:
1. **Permission Sets** → New, label `Relationship Manager`
2. Object Settings → Read/Create/Edit on Financial_Product__c, Relationship_Manager__c, Investment__c, Loan__c, Repayment__c, Compliance_Document__c

**Reports worth building**, once there's sample data: Investments by Status, Investments Maturing This Quarter, Loans by Status, Loan Portfolio at Risk (In Arrears + Defaulted). Same custom-report-type approach as real estate — a fresh custom object doesn't get a usable report type until you build one explicitly.

**The part that makes this a template, not a one-off org**: this is one repo, but two client types will use it. When an actual investment firm signs on, deploy everything *except* Loan__c, Repayment__c, and the Repayment Reminder flow. When a microfinance bank signs on, skip Investment__c and its flow instead. Both share Financial_Product__c, Relationship_Manager__c, Compliance_Document__c, and the Account/Contact extensions. A firm doing both gets everything. Write the deploy script so this is a flag, not a manual folder-picking exercise every time.

Final commit and push:
```
git add .
git commit -m "v1.0 template complete"
git push
```

Before handing this to an actual client: strip the Abuja sample data. The objects, app, and flows are what carry over — the data doesn't.

---

## Getting Unstuck

Same troubleshooting table as the real estate guide applies here — same CLI, same failure modes. If something's specific to this template (a `$` vs `.` field-reference issue in a report, a dashboard that needs `LoggedInUser` instead of a hardcoded running user, FLS not auto-granting on metadata-created fields), check `PROGRESS.md` in the real estate repo first — those exact issues already got solved once there.
