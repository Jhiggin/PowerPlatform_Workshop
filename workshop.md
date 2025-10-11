# Power Apps + Power Automate Full‑Day Hands‑On Demo — Step‑by‑Step Guide

**Format:** 1 full day (≈ 6.5–7.5 hrs of hands‑on + breaks)  
**Scenario:** Expense Request & Approval (SharePoint list + Canvas App + Approvals in Teams/Email)  
**Outcome:** A working Canvas app to submit/track expenses, an automated approval flow, a button‑triggered approval from the app, and an optional scheduled reminder flow.  

---

## Agenda at a Glance
- **Lab 0 (15–25 min):** Environment & Solution setup
- **Lab 1 (35–45 min):** Create SharePoint list (data layer) + sample data
- **Lab 2 (90–120 min):** Build Canvas app (browse, details, edit) + UX polish
- **Lab 3 (75–90 min):** Build automated approval flow (on submit)
- **Lab 4 (45–60 min):** Integrate a Power Apps–triggered flow (button)
- **Lab 5 (30–45 min, optional):** Scheduled reminders for pending approvals
- **Lab 6 (20–30 min):** Package in a Solution, share, and publish

> Tip: Keep all assets inside **one Solution** to make export/import and ALM easy.

---

## Prerequisites Checklist
- Microsoft 365 tenant with access to **Power Apps**, **Power Automate**, **SharePoint**, and **Teams**.
- Permissions to create **SharePoint lists** and **Power Automate flows**.
- A SharePoint site where you can create lists (e.g., **Training – Expense App** site).
- Power Platform **environment** with Dataverse available (Approvals use Dataverse under the hood).

> **Modern Development Note**: This demo uses SharePoint lists for simplicity. For production apps, consider using **Dataverse tables** for better performance, relationships, and business logic. Also consider **modern controls** instead of classic controls for better accessibility and performance.

---

## Data Model (SharePoint)

### Step 1: Create Lookup Lists
First, create two supporting lists for categories and statuses:

**Categories List:**
- Create a list named **ExpenseCategories**
- Add items: Travel, Meals, Office, Other

**Status List:**
- Create a list named **ExpenseStatus**
- Add items: Draft, Submitted, Approved, Rejected, Paid

### Step 2: Create Main Expenses List
Create a SharePoint **List** named **Expenses** with these columns (enable **Attachments** on the list):

| Column Name | Type | Details |
|---|---|---|
| **Title** | Single line of text | Purpose/description of the expense |
| **Amount** | Currency | 2 decimals |
| **Category** | Lookup | Lookup to ExpenseCategories list (Title column) |
| **ExpenseDate** | Date and time | Date only |
| **Status** | Lookup | Lookup to ExpenseStatus list (Title column), Default: Draft |
| **Approver** | Person or Group | Single person |
| **Comments** | Multiple lines of text | Plain text |
| **ReceiptUrl** (optional) | Hyperlink | Link to a receipt if desired |

> Add a custom view **“Approvals”** that shows: Title, Amount, Category, ExpenseDate, Status, Approver, Modified, Modified By.

**Sample rows** (Quick Edit → paste):
- Title: "Hotel Chicago", Amount: 425, Category: Travel, ExpenseDate: 2025‑08‑07, Status: Draft
- Title: "Client Lunch", Amount: 78.50, Category: Meals, ExpenseDate: 2025‑08‑12, Status: Draft

---

## Lab 0 — Environment & Solution Setup (15–25 min)
1. Go to **make.powerapps.com** → ensure you’re in the intended **Environment**.
2. **Solutions** → **New solution**:  
   - Name: `ExpenseAppTraining`  
   - Publisher: create/select one (e.g., `Contoso Training`)  
   - Version: `1.0.0.0`
3. Open the solution; you’ll add **app** and **flows** here.

---

## Lab 1 — Create the SharePoint Lists (35–45 min)
1. **Create lookup lists first:**
   - In your SharePoint site, **New → List → Blank list**. Name it **ExpenseCategories**.
   - Add items: Travel, Meals, Office, Other (using the default Title column)
   - Create another list named **ExpenseStatus**
   - Add items: Draft, Submitted, Approved, Rejected, Paid
2. **Create main Expenses list:**
   - **New → List → Blank list**. Name it **Expenses**.
   - Add columns listed in **Data Model** (above). For Category and Status, create **Lookup** columns pointing to the respective lists.
   - Set **Status** default to **Draft** (select from lookup)
3. Ensure **Attachments** are enabled: **List settings → Advanced settings → Attachments: Enabled**.
4. Add 2–3 sample rows (select lookup values from dropdowns).
5. (Optional) Create the view **Approvals** and set a filter **Status is Submitted**.

---

## Lab 2 — Build the Canvas App (90–120 min)

### A. Create app & connect data
1. In **make.powerapps.com → Solutions → ExpenseAppTraining**, click **+ New → App → Canvas app**.
2. Name: **Expense App**. **Phone** layout.
3. Inside the app, **Data** pane → **Add data** → search **SharePoint** → connect to your site → select **Expenses** list.

### B. Screens
Create 3 screens: **BrowseScreen**, **DetailScreen**, **EditScreen**.

#### 1) BrowseScreen (Gallery + Search/Filters)
1. Insert **Text input** named `txtSearch` with **Hint text**: "Search by title or category".
2. Insert **Dropdown** `drpCategory` with Items: `["All","Travel","Meals","Office","Other"]` (or use `Distinct(Expenses, Category.Title)`). Default to **All**.
3. Insert **Dropdown** `drpStatus` with Items: `["All","Draft","Submitted","Approved","Rejected","Paid"]`. Default to **All**.
4. Insert a **Vertical Gallery** `galExpenses`. **Items** formula:
   ```powerfx
   With(
       {
           q: Lower(txtSearch.Text),
           cat: drpCategory.Selected.Value,
           stat: drpStatus.Selected.Value
       },
       SortByColumns(
           Filter(
               Expenses,
               (IsBlank(q) || StartsWith(Lower(Title), q) || StartsWith(Lower(Category.Title), q)),
               (IsBlank(cat) || cat = "All" || Category.Title = cat),
               (IsBlank(stat) || stat = "All" || Status.Title = stat)
           ),
           "Created",
           Descending
       )
   )
   ```
5. In the gallery template, show **Title**, **Amount**, **Category.Title**, **Status.Title**, and **ExpenseDate**.
6. (Optional UX) Color the status label:
   ```powerfx
   ColorValue(
     Switch(ThisItem.Status.Title,
       "Approved", "#107C10",
       "Rejected", "#A4262C",
       "Submitted", "#605E5C",
       "Paid", "#004E8C",
       "Draft", "#8A8886",
       "#605E5C"
     )
   )
   ```
7. Set **OnSelect** of the gallery template to:  
   `Navigate(DetailScreen, ScreenTransition.Fade)`

#### 2) DetailScreen
1. Insert **Display form** `frmDetail` → **DataSource**: `Expenses`, **Item**: `galExpenses.Selected`.
2. Fields: Title, Amount, Category, ExpenseDate, Status, Approver, Comments, ReceiptUrl.
3. Add buttons:
   - **Edit** → `Navigate(EditScreen, ScreenTransition.Cover)`
   - **New** → `NewForm(frmEdit); Navigate(EditScreen, ScreenTransition.Cover)`
   - **Submit for Approval** (we’ll wire this in Lab 4). For now:
     ```powerfx
     If(
       frmDetail.Item.Status.Title = "Draft",
       Patch(Expenses, frmDetail.Item, { Status: { Title: "Submitted" } });
       Notify("Marked as Submitted", NotificationType.Success);
       ResetForm(frmDetail);
       Back()
     )
     ```

#### 3) EditScreen
1. Insert **Edit form** `frmEdit` → **DataSource**: `Expenses`, **Item**: `galExpenses.Selected`.
2. Set **DefaultMode** to `If(IsBlank(galExpenses.Selected.ID), FormMode.New, FormMode.Edit)`.
3. Reorder cards; mark **Title**, **Amount**, **ExpenseDate** as **Required** (in Data card properties).
4. Add **Save** and **Cancel** buttons.
   - **Save** → `SubmitForm(frmEdit)`
   - **Cancel** → `ResetForm(frmEdit); Back()`
5. On the **Edit form** (`frmEdit`) set:
   - **OnSuccess**:
     ```powerfx
     Set(varCurrentID, frmEdit.LastSubmit.ID);
     Notify("Saved successfully", NotificationType.Success);
     Back()
     ```
   - **OnFailure**: `Notify(First(frmEdit.Error).Message, NotificationType.Error)`

### C. Polishing touches
- Add **Sort** icon on Browse to toggle ascending/descending:
  - Add a variable `varSortDesc` (App **OnStart**: `Set(varSortDesc, true)`), and update the **Items** formula’s `SortByColumns` 4th arg accordingly.
- Add **Validation** on **Amount** (Data card **DisplayMode** or **Update** with `If(Value(Amount_DataCardValue.Text) > 0, Value(Amount_DataCardValue.Text), Blank())`).
- (Optional) Add an **Approver Picker**: add **Office 365 Users** connector; use a **Combo box** bound to `Office365Users.SearchUser({searchTerm: cmbApprover.SearchText})`; store selected user email into the **Approver** column.

---

## Lab 3 — Automated Approval Flow (75–90 min)
**Goal:** When an item is **Submitted**, route for approval, update Status to **Approved/Rejected**, and notify the requester.

1. In **Solutions → ExpenseAppTraining**, **+ New → Automation → Cloud flow**.
2. Choose **“When an item is created or modified”** (SharePoint) as the **trigger**.
   - **Site address**: your site  
   - **List name**: **Expenses**
3. **New step → Condition** to run only when `Status` is **Submitted**:
   - Condition: **Status Title** *equals* `Submitted`.
   - In **If no**, **Terminate** (or **Do nothing**).
4. **If yes** branch → **Get manager (V2)** (Microsoft Entra ID connector):
   - **User (UPN)**: `Created By Email` (from trigger)
   *(Note: Microsoft Entra ID was formerly Azure AD. If manager lookup isn't available, skip and use a fixed approver or the Approver column from the item.)*
5. **Start and wait for an approval** (Approvals):
   - **Approval type**: Approve/Reject – First to respond  
   - **Title**: `Expense @{triggerOutputs()?['body/Title']} for $@{triggerOutputs()?['body/Amount']}`  
   - **Assigned to**: `Mail` from **Get manager** (or another email)  
   - **Details**: include a link to the item and fields:
     ```
     Purpose: @{triggerOutputs()?['body/Title']}
     Amount: $@{triggerOutputs()?['body/Amount']}
     Category: @{triggerOutputs()?['body/Category/Title']}
     Date: @{triggerOutputs()?['body/ExpenseDate']}
     Submitted by: @{triggerOutputs()?['body/Author/DisplayName']}
     ```
6. **Condition** on the approval outcome:
   - Left value: **Outcome** from the approval  
   - Operator: **is equal to**  
   - Right value: `Approve`
7. **If yes** → **Update item** (SharePoint): set **Status = Approved**; **Comments =** `Comments` from the approval; **Approver =** approver email.
8. **If no** → **Update item** with **Status = Rejected**, **Comments** = rejection comments.
9. **Send an email (V2)** (Outlook) to the requester (`Created By Email`) with the **Outcome** and **Comments**. Include the item link.
10. **Save** the flow and **Test** by editing an item’s **Status** to **Submitted** in SharePoint.

> Note: Approvals rely on Dataverse. If prompted, let Power Automate create the Dataverse table in your environment.

---

## Lab 4 — Integrate a Power Apps–Triggered Flow (45–60 min)
**Goal:** Submit an item for approval directly from the Canvas app and get a response back.

### A. Flow (Power Apps trigger)
1. In **Solutions → ExpenseAppTraining**, **+ New → Automation → Instant cloud flow**.
2. Select **Power Apps (V2)** as the trigger. Name it: **Submit Expense (from app)**.
3. After the trigger, add **Initialize variable** steps to capture **Ask in Power Apps** parameters:
   - `varExpenseID` (Integer) → **Ask in Power Apps**  
   - (Optional) `varApproverEmail` (String) → **Ask in Power Apps**
4. **Get item** (SharePoint) → pass `varExpenseID`.
5. **Update item** → set **Status = Submitted** (and **Approver** using `varApproverEmail` if provided).
6. **Start and wait for an approval** (same config as Lab 3; use Approver from **Get manager** or `varApproverEmail`).
7. **Condition** on **Outcome = Approve** → **Update item** to **Approved** else **Rejected**; write **Comments**.
8. **Respond to a PowerApp or flow** → Add output parameters, e.g.:
   - **Result** (Text): `Submitted`, `Approved`, or `Rejected`
   - **Approver** (Text): approver display name or email
   - **Comments** (Text): approval/rejection comments
9. **Save** the flow.

### B. Wire the flow from the Canvas app
1. In the **app editor**, **Data** pane → **Add data** → **Flows** → add **Submit Expense (from app)**.
2. On **DetailScreen**, set **Submit for Approval** button **OnSelect**:
   ```powerfx
   Set(
     approvalResponse,
     'Submit Expense (from app)'.Run(frmDetail.Item.ID)
   );
   If(
     approvalResponse.Result = "Submitted",
     Notify("Submitted for approval", NotificationType.Success),
     Notify("Result: " & Text(approvalResponse.Result) & If(!IsBlank(approvalResponse.Comments), ": " & approvalResponse.Comments, ""), NotificationType.Information)
   );
   Refresh(Expenses)
   ```
   > If you added an approver picker, pass it as second parameter: `...Run(frmDetail.Item.ID, cmbApprover.Selected.Mail)`
3. **Play** the app and submit an item. Confirm the status transitions and notifications.

---

## Lab 5 — Scheduled Reminders (Optional, 30–45 min)
**Goal:** Nightly reminder to approvers for items **Submitted** more than 2 days ago.

1. **+ New → Automation → Cloud flow**; choose **Recurrence** trigger: every **1** day at **9:00 AM**.
2. **Get items** (SharePoint) from **Expenses**.
3. **Filter array**: keep items where **Status Title = Submitted** and **Created** < `addDays(utcNow(), -2)`.
4. **For each** filtered item:
   - **Get manager (V2)** (Microsoft Entra ID) for **Author Email** (or use **Approver** column).
   - **Send message** (Teams or Email) to approver with item link and age in days (`div(sub(ticks(utcNow()), ticks(items('Apply_to_each')?['Created'])), 864000000000)`).

---

## Lab 6 — Package, Share, and Publish (20–30 min)
1. In the app editor → **File → Save**, then **Publish → Publish this version**.
2. **Share** the app with end users (give **User** permission) and approvers if they’ll open the app.
3. In **Power Automate**, **Share** the flows with your co‑instructors/builders as **Co‑owners**.
4. In **Solutions**, select **ExpenseAppTraining** → **Export** (unmanaged) to move between environments.

---

## Troubleshooting & Tips
- **Connections**: If a flow or app complains about connections, open the **Connections** pane and re‑authenticate.
- **Approvals not provisioning**: Let Power Automate create the Dataverse Approval tables; wait ~1–2 min and retry.
- **SharePoint lookup fields**: In Power Fx, reference lookup values via `.Title` (e.g., `ThisItem.Status.Title`) when using lookup columns.
- **Infinite trigger loops**: If you use **When item is modified**, make sure you only proceed when **Status = Submitted** to avoid loops.
- **Testing**: Use a second test account (or colleague) as the approver to see the full flow.

---

## Stretch Goals (If Time Allows)
- **AI Builder – Receipt Processing**: Extract totals from uploaded receipts and populate **Amount** automatically. (Premium)
- **Teams Adaptive Cards**: Replace email with interactive approvals in Teams using **Post adaptive card and wait for a response**.
- **Power BI Tile**: Embed a KPI tile in the app showing monthly spend by category.
- **Component Library**: Move common controls (search bar, status tag) into a component library for reuse.
- **Modern Controls**: Upgrade to modern controls for better performance and accessibility.
- **Dataverse Migration**: Consider migrating from SharePoint lists to Dataverse tables for better scalability.

---

## End‑of‑Day Artifacts Checklist
- [ ] SharePoint List: **Expenses**
- [ ] Canvas App: **Expense App** (3 screens + polish)
- [ ] Flow: **Expense Approval – Auto** (SP item trigger)
- [ ] Flow: **Submit Expense (from app)** (Power Apps trigger)
- [ ] (Optional) Flow: **Reminder – Pending Approvals** (Recurring)
- [ ] Solution: **ExpenseAppTraining** (all assets inside)

---

## Appendix — Handy Formulas & Expressions
**Search + filter + sort (Gallery Items):**
```powerfx
With(
    {
        q: Lower(txtSearch.Text),
        cat: drpCategory.Selected.Value,
        stat: drpStatus.Selected.Value
    },
    SortByColumns(
        Filter(
            Expenses,
            (IsBlank(q) || StartsWith(Lower(Title), q) || StartsWith(Lower(Category.Value), q)),
            (IsBlank(cat) || cat = "All" || Category.Value = cat),
            (IsBlank(stat) || stat = "All" || Status.Value = stat)
        ),
        "Created",
        Descending
    )
)
```

**Success handler (Edit form):**
```powerfx
Set(varCurrentID, frmEdit.LastSubmit.ID);
Notify("Saved successfully", NotificationType.Success);
Back()
```

**Submit for approval (simple Patch):**
```powerfx
If(
  frmDetail.Item.Status.Title = "Draft",
  Patch(Expenses, frmDetail.Item, { Status: { Title: "Submitted" } });
  Notify("Marked as Submitted", NotificationType.Success);
  Refresh(Expenses)
)
```

**Compose link to item (for emails):**
```
https://<yourtenant>.sharepoint.com/sites/<SiteName>/Lists/Expenses/DispForm.aspx?ID=<ID>
```

**Filter array condition (Power Automate):** keep items where
```
@and(
  equals(item()?['Status']?['Title'], 'Submitted'),
  less(item()?['Created'], addDays(utcNow(), -2))
)
```

---

**You’re set.** This guide walks attendees from a clean tenant to a functional app and flows with room for stretch topics. Adjust the timings to fit your audience’s pace.

