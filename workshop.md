# Expense Management Application Design Instructions

## Overview
This document provides comprehensive instructions for creating a 3-screen expense management application in Power Apps with an integrated Power Automate approval workflow.

## Table of Contents
1. [Data Model & Backend Setup](#data-model--backend-setup)
2. [Screen 1: Dashboard](#screen-1-dashboard)
3. [Screen 2: Expense Creation](#screen-2-expense-creation)
4. [Screen 3: Expense Details](#screen-3-expense-details)
5. [Power Automate Approval Flow](#power-automate-approval-flow)
6. [Security & Permissions](#security--permissions)
7. [Implementation Steps](#implementation-steps)

---

## Data Model & Backend Setup

### SharePoint Lists Structure

#### 1. Expenses List
**List Name:** `Expenses`

| Column Name | Type | Required | Description |
|-------------|------|----------|-------------|
| Title | Single line of text | Yes | Expense description/title |
| Amount | Currency | Yes | Expense amount |
| Category | Choice | Yes | Expense category (Meals, Travel, Office, etc.) |
| ExpenseDate | Date | Yes | Date expense was incurred |
| SubmittedBy | Person | Yes | User who submitted the expense |
| SubmittedDate | Date/Time | Yes | When expense was submitted |
| Status | Choice | Yes | Submitted, Approved, Rejected, Paid |
| Manager | Person | No | Assigned manager for approval |
| ApprovalComments | Multiple lines of text | No | Manager's approval/rejection comments |
| ApprovedDate | Date/Time | No | Date of approval/rejection |
| Receipt | Attachment | No | Receipt image/document |
| BusinessJustification | Multiple lines of text | Yes | Business reason for expense |

**Choice Values for Status:**
- Submitted
- Approved
- Rejected
- Paid

**Choice Values for Category:**
- Meals & Entertainment
- Travel & Transportation
- Office Supplies
- Software & Subscriptions
- Training & Education
- Other

#### 2. Managers List (Optional)
**List Name:** `ExpenseManagers`

| Column Name | Type | Required | Description |
|-------------|------|----------|-------------|
| Title | Single line of text | Yes | Department/Team name |
| Employee | Person | Yes | Employee |
| Manager | Person | Yes | Their manager |

---

## Screen 1: Dashboard

### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│                    Header Section                       │
├─────────────────────────────────────────────────────────┤
│  [Submitted]  [Approved]  [Rejected]  [Paid]          │
│    $1,250      $2,100      $150      $1,950           │
├─────────────────────────────────────────────────────────┤
│  Filter: [Category ▼] [Status ▼] [Date Range]  [Sort ▼]│
├─────────────────────────────────────────────────────────┤
│  Expense List Gallery                                   │
│  ┌─────────────────────────────────────────────────┐   │
│  │ Meals & Entertainment    $125.00   Submitted    │   │
│  │ 2024-01-15              [View Details]          │   │
│  ├─────────────────────────────────────────────────┤   │
│  │ Travel - Flight          $450.00   Approved     │   │
│  │ 2024-01-12              [View Details]          │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Components Details

#### Status Summary Cards (Horizontal Container)
- **Container:** Horizontal container with 4 cards
- **Card Design:** Each card shows:
  - Status name (large text)
  - Total amount for that status (currency format)
  - Background color coding:
    - Submitted: Blue (#0078D4)
    - Approved: Green (#107C10)
    - Rejected: Red (#D13438)
    - Paid: Gray (#605E5C)

#### Filter Section
- **Category Dropdown:** Filter by expense category
- **Status Dropdown:** Filter by approval status
- **Date Range:** Start and end date pickers
- **Sort Dropdown:** Options: Date (newest first), Date (oldest first), Amount (high to low), Amount (low to high)

#### Expense List Gallery
- **Gallery Type:** Vertical gallery with custom template
- **Template Design:**
  - Left side: Category icon + expense title
  - Center: Amount (large, bold)
  - Right side: Status badge + "View Details" button
  - Bottom: Submission date

### Formulas & Logic

#### Status Summary Calculations
```powerapps
// Submitted Total
Sum(Filter(Expenses, SubmittedBy.Email = User().Email && Status.Value = "Submitted"), Amount)

// Approved Total  
Sum(Filter(Expenses, SubmittedBy.Email = User().Email && Status.Value = "Approved"), Amount)

// Rejected Total
Sum(Filter(Expenses, SubmittedBy.Email = User().Email && Status.Value = "Rejected"), Amount)

// Paid Total
Sum(Filter(Expenses, SubmittedBy.Email = User().Email && Status.Value = "Paid"), Amount)
```

**⚠️ Delegation Warning Fix:**
If you encounter delegation warnings with the above formulas, use these delegation-friendly alternatives:

```powerapps
// Alternative 1: Using With() to separate filters
Text(With({UserExpenses: Filter(Expenses, SubmittedBy.Email = User().Email)}, 
    Sum(Filter(UserExpenses, Status.Value = "Submitted"), Amount)), "$#,##0.00")

// Alternative 2: Using a collection (add to App.OnStart)
ClearCollect(UserExpenses, Filter(Expenses, SubmittedBy.Email = User().Email));

// Then use in labels:
Text(Sum(Filter(UserExpenses, Status.Value = "Submitted"), Amount), "$#,##0.00")

// Alternative 3: For very large datasets, consider using AddColumns with aggregation
Text(Sum(AddColumns(Filter(Expenses, SubmittedBy.Email = User().Email && Status.Value = "Submitted"), 
    "AmountValue", Amount), AmountValue), "$#,##0.00")
```

**Recommended Approach for Production:**
Use Alternative 2 with collections for better performance:

1. **Add to App.OnStart:**
```powerapps
// Load user's expenses into a collection on app start
ClearCollect(UserExpenses, Filter(Expenses, SubmittedBy.Email = User().Email));
```

2. **Update status card formulas to:**
```powerapps
// Submitted
Text(Sum(Filter(UserExpenses, Status.Value = "Submitted"), Amount), "$#,##0.00")

// Approved  
Text(Sum(Filter(UserExpenses, Status.Value = "Approved"), Amount), "$#,##0.00")

// Rejected
Text(Sum(Filter(UserExpenses, Status.Value = "Rejected"), Amount), "$#,##0.00")

// Paid
Text(Sum(Filter(UserExpenses, Status.Value = "Paid"), Amount), "$#,##0.00")
```

3. **Refresh collection when new expenses are added:**
Add this to the submit button after successful expense creation:
```powerapps
// Refresh the collection after adding new expense
ClearCollect(UserExpenses, Filter(Expenses, SubmittedBy.Email = User().Email));
```

#### Gallery Data Source
```powerapps
// Base filter for current user
With(
    {
        BaseFilter: Filter(
            Expenses,
            SubmittedBy.Email = User().Email
        )
    },
    // Apply additional filters based on dropdown selections
    Filter(
        BaseFilter,
        (IsBlank(CategoryFilter.Selected) || Category.Value = CategoryFilter.Selected.Value) &&
        (IsBlank(StatusFilter.Selected) || Status.Value = StatusFilter.Selected.Value) &&
        (IsBlank(StartDate.SelectedDate) || ExpenseDate >= StartDate.SelectedDate) &&
        (IsBlank(EndDate.SelectedDate) || ExpenseDate <= EndDate.SelectedDate)
    )
)
```

---

## Screen 2: Expense Creation

### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│                  Create New Expense                     │
├─────────────────────────────────────────────────────────┤
│  Expense Title: [________________________]              │
│                                                         │
│  Amount: [$_______]  Category: [Dropdown ▼]           │
│                                                         │
│  Expense Date: [Date Picker]                           │
│                                                         │
│  Business Justification:                                │
│  [_________________________________________________]    │
│  [_________________________________________________]    │
│  [_________________________________________________]    │
│                                                         │
│  Receipt (Optional):                                    │
│  [Upload File] or [Take Photo]                         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │            Receipt Preview                       │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│              [Cancel]    [Submit Expense]               │
└─────────────────────────────────────────────────────────┘
```

### Form Components

#### Input Controls
1. **Expense Title:** Text input (required)
2. **Amount:** Number input with currency formatting (required)
3. **Category:** Dropdown with predefined categories (required)
4. **Expense Date:** Date picker (required, default to today)
5. **Business Justification:** Multi-line text input (required)
6. **Receipt:** File upload control (optional)

#### Validation Rules
```powerapps
// Form validation before submission
If(
    IsBlank(TitleInput.Text) ||
    IsBlank(AmountInput.Text) ||
    AmountInput.Text <= 0 ||
    IsBlank(CategoryDropdown.Selected) ||
    IsBlank(ExpenseDatePicker.SelectedDate) ||
    IsBlank(JustificationInput.Text),
    
    // Show error message
    Notify("Please fill in all required fields", NotificationType.Error),
    
    // Proceed with submission
    SubmitExpense()
)
```

#### Submit Function
```powerapps
// SubmitExpense function
Patch(
    Expenses,
    Defaults(Expenses),
    {
        Title: TitleInput.Text,
        Amount: Value(AmountInput.Text),
        Category: CategoryDropdown.Selected,
        ExpenseDate: ExpenseDatePicker.SelectedDate,
        SubmittedBy: User(),
        SubmittedDate: Now(),
        Status: {Value: "Submitted"},
        BusinessJustification: JustificationInput.Text,
        Manager: LookUp(ExpenseManagers, Employee.Email = User().Email).Manager
    }
);

// Trigger Power Automate flow
ExpenseApprovalFlow.Run(
    Last(Expenses).ID,
    TitleInput.Text,
    Value(AmountInput.Text),
    User().FullName,
    LookUp(ExpenseManagers, Employee.Email = User().Email).Manager.Email
);

// Reset form and navigate back
Reset(ExpenseForm);
Navigate(DashboardScreen, ScreenTransition.UnCover);
Notify("Expense submitted successfully!", NotificationType.Success)
```

---

## Screen 3: Expense Details

### Layout Structure
```
┌─────────────────────────────────────────────────────────┐
│  ← Back to Dashboard                                    │
├─────────────────────────────────────────────────────────┤
│                   Expense Details                       │
│                                                         │
│  Title: Business Lunch with Client                     │
│  Amount: $125.00                                        │
│  Category: Meals & Entertainment                        │
│  Date: January 15, 2024                               │
│                                                         │
│  Business Justification:                                │
│  Client meeting to discuss Q1 project requirements     │
│  and deliverables. Lunch meeting was most convenient   │
│  for both parties.                                      │
│                                                         │
│  Receipt:                                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │            [Receipt Image]                       │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Approval Status                     │   │
│  │                                                  │   │
│  │  Status: Submitted                              │   │
│  │  Submitted: Jan 15, 2024 2:30 PM               │   │
│  │  Submitted by: John Doe                         │   │
│  │  Manager: Jane Smith                            │   │
│  │                                                  │   │
│  │  ○ Submitted ✓                                  │   │
│  │  ○ Manager Review (Pending)                     │   │
│  │  ○ Approved                                     │   │
│  │  ○ Paid                                         │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│              [Edit] [Delete] [Back]                     │
└─────────────────────────────────────────────────────────┘
```

### Component Details

#### Expense Information Display
- **Display Form:** Shows all expense details in read-only format
- **Conditional Formatting:** Status-based color coding
- **Receipt Display:** Image control with zoom capability

#### Approval Timeline
- **Visual Progress:** Step-by-step approval process
- **Status Indicators:** 
  - Completed steps: Green checkmark
  - Current step: Blue circle
  - Pending steps: Gray circle
- **Timestamps:** Show when each step was completed

#### Action Buttons
- **Edit Button:** Navigate to edit screen (only if status = "Submitted")
- **Delete Button:** Delete expense (only if status = "Submitted")
- **Back Button:** Return to dashboard

### Navigation Logic
```powerapps
// Pass selected expense to detail screen
Set(SelectedExpense, ThisItem);
Navigate(ExpenseDetailScreen, ScreenTransition.Cover)

// On detail screen, display the selected expense
DisplayForm.Item = SelectedExpense
```

---

## Power Automate Approval Flow

### Flow Overview
**Flow Name:** Expense Approval Workflow

### Trigger
- **Type:** PowerApps trigger
- **Inputs:**
  - ExpenseID (number)
  - ExpenseTitle (string)
  - Amount (number)
  - SubmitterName (string)
  - ManagerEmail (string)

### Flow Steps

#### 1. Get Expense Details
```
Action: Get item
Site: [Your SharePoint Site]
List: Expenses
ID: ExpenseID (from trigger)
```

#### 2. Start Approval Process
```
Action: Start and wait for an approval
Approval type: Approve/Reject - First to respond
Title: Expense Approval Required: [ExpenseTitle]
Assigned to: [ManagerEmail]
Details: 
Employee: [SubmitterName]
Amount: $[Amount]
Description: [ExpenseTitle]
Business Justification: [BusinessJustification]
Date: [ExpenseDate]

Please review and approve/reject this expense request.
```

#### 3. Condition: Check Approval Response
```
Condition: Approval response equals "Approve"
```

#### 4A. If Approved
```
Action: Update item
Site: [Your SharePoint Site]  
List: Expenses
ID: ExpenseID
Fields:
- Status: Approved
- ApprovedDate: utcNow()
- ApprovalComments: [Approval comments]
```

```
Action: Send email notification
To: [SubmitterEmail]
Subject: Expense Approved - [ExpenseTitle]
Body: Your expense request for $[Amount] has been approved by [ManagerName].
```

#### 4B. If Rejected
```
Action: Update item
Site: [Your SharePoint Site]
List: Expenses  
ID: ExpenseID
Fields:
- Status: Rejected
- ApprovedDate: utcNow()
- ApprovalComments: [Approval comments]
```

```
Action: Send email notification
To: [SubmitterEmail]
Subject: Expense Rejected - [ExpenseTitle]  
Body: Your expense request for $[Amount] has been rejected. 
Reason: [Approval comments]
```

### Flow Diagram
```
PowerApps Trigger
       ↓
Get Expense Details
       ↓
Start Approval Process
       ↓
   Condition
   ↙       ↘
Approved   Rejected
   ↓         ↓
Update to   Update to
Approved    Rejected
   ↓         ↓
Send Email  Send Email
Notification Notification
```

---

## Security & Permissions

### SharePoint List Permissions
- **Expenses List:**
  - Users: Contribute (can create/edit their own items)
  - Managers: Full Control
  - HR/Finance: Full Control

### Power Apps Security
- **App Sharing:**
  - Share with all employees who need to submit expenses
  - Managers need access to approve expenses

### Power Automate Permissions
- **Flow Permissions:**
  - Run-only permissions for app users
  - Edit permissions for IT administrators

---

## Detailed Build Instructions

### Phase 1: Backend Setup (SharePoint)

#### Step 1.1: Create SharePoint Site
1. Navigate to SharePoint admin center or your tenant
2. Click **"Create site"**
3. Select **"Team site"**
4. Enter site details:
   - **Site name:** "Expense Management"
   - **Description:** "Employee expense tracking and approval system"
   - **Privacy:** Private (add members as needed)
5. Click **"Create"**
6. Wait for site provisioning to complete

#### Step 1.2: Create Expenses List
1. In your new SharePoint site, click **"New" > "List"**
2. Select **"Blank list"**
3. Name: **"Expenses"**
4. Click **"Create"**
5. Add the following columns (click **"Add column"** for each):

   **Column 1: Amount**
   - Type: **Currency**
   - Required: **Yes**
   - Default value: Leave blank

   **Column 2: Category**
   - Type: **Choice**
   - Required: **Yes**
   - Choices (enter each on new line):
     ```
     Meals & Entertainment
     Travel & Transportation
     Office Supplies
     Software & Subscriptions
     Training & Education
     Other
     ```

   **Column 3: ExpenseDate**
   - Type: **Date and time**
   - Required: **Yes**
   - Include time: **No**

   **Column 4: SubmittedBy**
   - Type: **Person or Group**
   - Required: **Yes**
   - Allow multiple selections: **No**

   **Column 5: SubmittedDate**
   - Type: **Date and time**
   - Required: **Yes**
   - Include time: **Yes**

   **Column 6: Status**
   - Type: **Choice**
   - Required: **Yes**
   - Default value: **Submitted**
   - Choices:
     ```
     Submitted
     Approved
     Rejected
     Paid
     ```

   **Column 7: Manager**
   - Type: **Person or Group**
   - Required: **No**
   - Allow multiple selections: **No**

   **Column 8: ApprovalComments**
   - Type: **Multiple lines of text**
   - Required: **No**
   - Type of text: **Plain text**

   **Column 9: ApprovedDate**
   - Type: **Date and time**
   - Required: **No**
   - Include time: **Yes**

   **Column 10: BusinessJustification**
   - Type: **Multiple lines of text**
   - Required: **Yes**
   - Type of text: **Plain text**

6. Modify the default **"Title"** column:
   - Click on column header > **"Column settings" > "Edit"**
   - Change description to: "Expense description/title"

#### Step 1.3: Create ExpenseManagers List (Optional but Recommended)
1. Click **"New" > "List"**
2. Select **"Blank list"**
3. Name: **"ExpenseManagers"**
4. Add columns:

   **Column 1: Employee**
   - Type: **Person or Group**
   - Required: **Yes**

   **Column 2: Manager**
   - Type: **Person or Group**
   - Required: **Yes**

5. Populate with employee-manager relationships

#### Step 1.4: Configure List Permissions
1. Go to **Expenses** list
2. Click **Settings gear > List settings**
3. Under **Permissions and Management**, click **"Permissions for this list"**
4. Click **"Stop Inheriting Permissions"**
5. Configure permissions:
   - **All Employees:** Contribute (can create and edit their own items)
   - **Managers:** Full Control
   - **HR/Finance:** Full Control

#### Step 1.5: Test Data Entry
1. Create 2-3 test expense records
2. Verify all fields save correctly
3. Test different status values
4. Confirm permissions work as expected

---

### Phase 2: Power Automate Flow Creation

#### Step 2.1: Create New Flow
1. Go to **Power Automate** (flow.microsoft.com)
2. Click **"Create" > "Instant cloud flow"**
3. Flow name: **"Expense Approval Workflow"**
4. Choose trigger: **"PowerApps"**
5. Click **"Create"**

#### Step 2.2: Configure PowerApps Trigger
1. Click on the PowerApps trigger step
2. Click **"Add an input"**
3. Add the following inputs:
   - **ExpenseID** (Number)
   - **ExpenseTitle** (Text)
   - **Amount** (Number)
   - **SubmitterName** (Text)
   - **ManagerEmail** (Text)

#### Step 2.3: Add Get Item Action
1. Click **"New step"**
2. Search for **"SharePoint"**
3. Select **"Get item"**
4. Configure:
   - **Site Address:** Select your expense management site
   - **List Name:** Expenses
   - **Id:** Select **ExpenseID** from dynamic content

#### Step 2.4: Add Start Approval Action
1. Click **"New step"**
2. Search for **"Approvals"**
3. Select **"Start and wait for an approval"**
4. Configure:
   - **Approval type:** Approve/Reject - First to respond
   - **Title:** 
     ```
     Expense Approval Required: @{triggerBody()['text']}
     ```
   - **Assigned to:** Select **ManagerEmail** from dynamic content
   - **Details:** 
     ```
     Employee: @{triggerBody()['text_1']}
     Amount: $@{triggerBody()['number']}
     Description: @{triggerBody()['text']}
     Business Justification: @{body('Get_item')?['BusinessJustification']}
     Date: @{body('Get_item')?['ExpenseDate']}
     
     Please review and approve/reject this expense request.
     ```

#### Step 2.5: Add Condition for Approval Response
1. Click **"New step"**
2. Search for **"Control"**
3. Select **"Condition"**
4. Configure condition:
   - **Choose a value:** Select **Outcome** from dynamic content (under Start and wait for approval)
   - **Condition:** is equal to
   - **Choose a value:** Approve

#### Step 2.6: Configure "If Yes" Branch (Approved)
1. In the **"If yes"** branch, click **"Add an action"**
2. Search for **"SharePoint"**
3. Select **"Update item"**
4. Configure:
   - **Site Address:** Your expense management site
   - **List Name:** Expenses
   - **Id:** Select **ExpenseID** from dynamic content
   - **Status Value:** Approved
   - **ApprovedDate:** Select **utcNow()** from expression
   - **ApprovalComments:** Select **Response summary** from dynamic content

5. Add another action in **"If yes"** branch:
6. Search for **"Office 365 Outlook"**
7. Select **"Send an email (V2)"**
8. Configure:
   - **To:** Select **SubmittedBy Email** from Get item dynamic content
   - **Subject:** 
     ```
     Expense Approved - @{triggerBody()['text']}
     ```
   - **Body:**
     ```
     Your expense request for $@{triggerBody()['number']} has been approved.
     
     Expense: @{triggerBody()['text']}
     Amount: $@{triggerBody()['number']}
     Approved by: @{body('Start_and_wait_for_an_approval')?['responses'][0]['responder']['displayName']}
     
     Your expense will be processed for payment.
     ```

#### Step 2.7: Configure "If No" Branch (Rejected)
1. In the **"If no"** branch, click **"Add an action"**
2. Search for **"SharePoint"**
3. Select **"Update item"**
4. Configure:
   - **Site Address:** Your expense management site
   - **List Name:** Expenses
   - **Id:** Select **ExpenseID** from dynamic content
   - **Status Value:** Rejected
   - **ApprovedDate:** Select **utcNow()** from expression
   - **ApprovalComments:** Select **Response summary** from dynamic content

5. Add another action in **"If no"** branch:
6. Search for **"Office 365 Outlook"**
7. Select **"Send an email (V2)"**
8. Configure:
   - **To:** Select **SubmittedBy Email** from Get item dynamic content
   - **Subject:**
     ```
     Expense Rejected - @{triggerBody()['text']}
     ```
   - **Body:**
     ```
     Your expense request for $@{triggerBody()['number']} has been rejected.
     
     Expense: @{triggerBody()['text']}
     Amount: $@{triggerBody()['number']}
     Rejected by: @{body('Start_and_wait_for_an_approval')?['responses'][0]['responder']['displayName']}
     Reason: @{body('Start_and_wait_for_an_approval')?['responses'][0]['comments']}
     
     Please contact your manager if you have questions.
     ```

#### Step 2.8: Save and Test Flow
1. Click **"Save"**
2. Click **"Test"**
3. Select **"Manually"**
4. Provide test values for all inputs
5. Click **"Run flow"**
6. Verify flow completes successfully

---

### Phase 3: Power Apps Development

#### Step 3.1: Create New Canvas App
1. Go to **Power Apps** (make.powerapps.com)
2. Click **"Create" > "Canvas app from blank"**
3. App name: **"Expense Management"**
4. Format: **Tablet** (recommended for dashboard layout)
5. Click **"Create"**

#### Step 3.2: Connect to Data Sources
1. In the app designer, click **"Data" tab** (cylinder icon)
2. Click **"Add data"**
3. Search for **"SharePoint"**
4. Select your SharePoint site
5. Select **"Expenses"** list
6. Click **"Connect"**
7. Repeat for **"ExpenseManagers"** list if created

#### Step 3.2.1: Configure App.OnStart for Performance (Recommended)
1. Select **"App"** in the tree view
2. Set the **OnStart** property to:
```powerapps
// Load user's expenses into a collection for better performance
ClearCollect(UserExpenses, Filter(Expenses, SubmittedBy.Email = User().Email));
```
3. This prevents delegation warnings and improves performance for status calculations

#### Step 3.3: Build Screen 1 (Dashboard)

##### Add Status Summary Cards
1. **Rename Screen1 to "DashboardScreen":**
   - Select Screen1 in tree view
   - Change Name property to: `DashboardScreen`

2. **Add Header Label:**
   - Insert > Label
   - Text: `"Expense Dashboard"`
   - Position: Top center
   - Font size: 24
   - Font weight: Bold

3. **Create Horizontal Container for Status Cards:**
   - Insert > Layout > Horizontal container
   - Name: `StatusContainer`
   - Position below header
   - Width: App.Width - 40
   - Height: 120

4. **Add Status Cards (4 cards in container):**

   **Card 1 - Submitted:**
   - Insert > Layout > Vertical container (inside StatusContainer)
   - Name: `SubmittedCard`
   - Fill: `RGBA(0, 120, 212, 1)` (Blue)
   - Add Label for title:
     - Text: `"SUBMITTED"`
     - Color: White
     - Font weight: Bold
   - Add Label for amount:
     - Text: `Text(Sum(Filter(UserExpenses, Status.Value = "Submitted"), Amount), "$#,##0.00")`
     - Color: White
     - Font size: 18

   **Card 2 - Approved:**
   - Insert > Layout > Vertical container (inside StatusContainer)
   - Name: `ApprovedCard`
   - Fill: `RGBA(16, 124, 16, 1)` (Green)
   - Add Labels similar to Card 1:
     - Title: `"APPROVED"`
     - Amount: `Text(Sum(Filter(UserExpenses, Status.Value = "Approved"), Amount), "$#,##0.00")`

   **Card 3 - Rejected:**
   - Insert > Layout > Vertical container (inside StatusContainer)
   - Name: `RejectedCard`
   - Fill: `RGBA(209, 52, 56, 1)` (Red)
   - Add Labels:
     - Title: `"REJECTED"`
     - Amount: `Text(Sum(Filter(UserExpenses, Status.Value = "Rejected"), Amount), "$#,##0.00")`

   **Card 4 - Paid:**
   - Insert > Layout > Vertical container (StatusContainer)
   - Name: `PaidCard`
   - Fill: `RGBA(96, 94, 92, 1)` (Gray)
   - Add Labels:
     - Title: `"PAID"`
     - Amount: `Text(Sum(Filter(UserExpenses, Status.Value = "Paid"), Amount), "$#,##0.00")`

##### Add Filter Controls
1. **Add Horizontal Container for Filters:**
   - Insert > Layout > Horizontal container
   - Name: `FilterContainer`
   - Position below status cards

2. **Add Category Filter:**
   - Insert > Input > Dropdown
   - Name: `CategoryFilter`
   - Items: `Distinct(Expenses, Category.Value)`
   - Allow empty selection: True
   - Hint text: `"All Categories"`

3. **Add Status Filter:**
   - Insert > Input > Dropdown
   - Name: `StatusFilter`
   - Items: `["Submitted", "Approved", "Rejected", "Paid"]`
   - Allow empty selection: True
   - Hint text: `"All Statuses"`

4. **Add Date Filters:**
   - Insert > Input > Date picker
   - Name: `StartDateFilter`
   - Hint text: `"Start Date"`
   - Insert > Input > Date picker
   - Name: `EndDateFilter`
   - Hint text: `"End Date"`

5. **Add Sort Dropdown:**
   - Insert > Input > Dropdown
   - Name: `SortFilter`
   - Items: `["Date (Newest)", "Date (Oldest)", "Amount (High to Low)", "Amount (Low to High)"]`
   - Default: `"Date (Newest)"`

##### Add Expense Gallery
1. **Insert Vertical Gallery:**
   - Insert > Gallery > Vertical gallery
   - Name: `ExpenseGallery`
   - Position below filters
   - Width: App.Width - 40
   - Height: Remaining screen space

2. **Configure Gallery Data Source:**
   - Items property:
   ```powerapps
   With(
       {
           FilteredExpenses: Filter(
               Expenses,
               SubmittedBy.Email = User().Email &&
               (IsBlank(CategoryFilter.Selected) || Category.Value = CategoryFilter.Selected.Value) &&
               (IsBlank(StatusFilter.Selected) || Status.Value = StatusFilter.Selected.Value) &&
               (IsBlank(StartDateFilter.SelectedDate) || ExpenseDate >= StartDateFilter.SelectedDate) &&
               (IsBlank(EndDateFilter.SelectedDate) || ExpenseDate <= EndDateFilter.SelectedDate)
           )
       },
       Switch(
           SortFilter.Selected.Value,
           "Date (Newest)", SortByColumns(FilteredExpenses, "SubmittedDate", Descending),
           "Date (Oldest)", SortByColumns(FilteredExpenses, "SubmittedDate", Ascending),
           "Amount (High to Low)", SortByColumns(FilteredExpenses, "Amount", Descending),
           "Amount (Low to High)", SortByColumns(FilteredExpenses, "Amount", Ascending),
           SortByColumns(FilteredExpenses, "SubmittedDate", Descending)
       )
   )
   ```

<!-- 3. **Customize Gallery Template:**
   - Select first item in gallery
   - Add Labels for:
     - **Title:** `ThisItem.Title`
     - **Amount:** `Text(ThisItem.Amount, "$#,##0.00")`
     - **Category:** `ThisItem.Category.Value`
     - **Date:** `Text(ThisItem.ExpenseDate, "mm/dd/yyyy")`
     - **Status:** `ThisItem.Status.Value` -->
   
3. **Add View Details Button:**
   - Insert > Button
   - Text: `"View Details"`
   - OnSelect: 
   ```powerapps
   Set(SelectedExpense, ThisItem);
   Navigate(DetailScreen, ScreenTransition.Cover)
   ```

4. **Add New Expense Button:**
   - Insert > Button (outside gallery, top right)
   - Text: `"+ New Expense"`
   - OnSelect: `Navigate(CreateScreen, ScreenTransition.Cover)`

#### Step 3.4: Build Screen 2 (Expense Creation)

1. **Add New Screen:**
   - Insert > New screen > Blank
   - Name: `CreateScreen`

2. **Add Header:**
   - Insert > Label
   - Text: `"Create New Expense"`
   - Font size: 24, Font weight: Bold

3. **Add Back Button:**
   - Insert > Button
   - Text: `"← Back"`
   - OnSelect: `Navigate(DashboardScreen, ScreenTransition.UnCover)`

4. **Add Form Controls:**

   **Expense Title:**
   - Insert > Input > Text input
   - Name: `TitleInput`
   - HintText: `"Enter expense description"`

   **Amount:**
   - Insert > Input > Text input
   - Name: `AmountInput`
   - Format: Number
   - HintText: `"0.00"`

   **Category:**
   - Insert > Input > Dropdown
   - Name: `CategoryDropdown`
   - Items: `["Meals & Entertainment", "Travel & Transportation", "Office Supplies", "Software & Subscriptions", "Training & Education", "Other"]`

   **Expense Date:**
   - Insert > Input > Date picker
   - Name: `ExpenseDatePicker`
   - DefaultDate: `Today()`

   **Business Justification:**
   - Insert > Input > Text input
   - Name: `JustificationInput`
   - Mode: Multiline
   - HintText: `"Enter business justification for this expense"`

   **Receipt Upload:**
   - Insert > Media > Add picture
   - Name: `ReceiptUpload`

5. **Add Submit Button:**
   - Insert > Button
   - Text: `"Submit Expense"`
   - OnSelect:
   ```powerapps
   If(
       IsBlank(TitleInput.Text) ||
       IsBlank(AmountInput.Text) ||
       Value(AmountInput.Text) <= 0 ||
       IsBlank(CategoryDropdown.Selected) ||
       IsBlank(ExpenseDatePicker.SelectedDate) ||
       IsBlank(JustificationInput.Text),
       
       Notify("Please fill in all required fields", NotificationType.Error),
       
       With(
           {
               NewExpense: Patch(
                   Expenses,
                   Defaults(Expenses),
                   {
                       Title: TitleInput.Text,
                       Amount: Value(AmountInput.Text),
                       Category: {Value: CategoryDropdown.Selected.Value},
                       ExpenseDate: ExpenseDatePicker.SelectedDate,
                       SubmittedBy: User(),
                       SubmittedDate: Now(),
                       Status: {Value: "Submitted"},
                       BusinessJustification: JustificationInput.Text,
                       Manager: If(
                           !IsEmpty(ExpenseManagers),
                           LookUp(ExpenseManagers, Employee.Email = User().Email).Manager,
                           User()
                       )
                   }
               )
           },
           // Trigger Power Automate Flow
           'ExpenseApprovalWorkflow'.Run(
               NewExpense.ID,
               NewExpense.Title,
               NewExpense.Amount,
               User().FullName,
               If(
                   !IsEmpty(ExpenseManagers),
                   LookUp(ExpenseManagers, Employee.Email = User().Email).Manager.Email,
                   User().Email
               )
           );
           
           // Refresh the collection to update status cards
           ClearCollect(UserExpenses, Filter(Expenses, SubmittedBy.Email = User().Email));
           
           // Reset form and navigate
           Reset(TitleInput);
           Reset(AmountInput);
           Reset(CategoryDropdown);
           Reset(ExpenseDatePicker);
           Reset(JustificationInput);
           Reset(ReceiptUpload);
           
           Navigate(DashboardScreen, ScreenTransition.UnCover);
           Notify("Expense submitted successfully!", NotificationType.Success)
       )
   )
   ```

6. **Add Cancel Button:**
   - Insert > Button
   - Text: `"Cancel"`
   - OnSelect: `Navigate(DashboardScreen, ScreenTransition.UnCover)`

#### Step 3.5: Build Screen 3 (Expense Details)

1. **Add New Screen:**
   - Insert > New screen > Blank
   - Name: `ExpenseDetailScreen`

2. **Add Header and Back Button:**
   - Insert > Button
   - Text: `"← Back to Dashboard"`
   - OnSelect: `Navigate(DashboardScreen, ScreenTransition.UnCover)`

3. **Add Detail Display Form:**
   - Insert > Forms > Display form
   - Name: `ExpenseDetailForm`
   - DataSource: Expenses
   - Item: `SelectedExpense`
   - Fields: Select all relevant fields

4. **Add Approval Status Section:**
   - Insert > Label
   - Text: `"Approval Status"`
   - Font weight: Bold

5. **Add Status Timeline:**
   - Create visual timeline using shapes and labels
   - Use conditional formatting based on `SelectedExpense.Status.Value`

6. **Add Action Buttons:**
   
   **Edit Button:**
   - Insert > Button
   - Text: `"Edit"`
   - Visible: `SelectedExpense.Status.Value = "Submitted"`
   - OnSelect: Navigation to edit screen (optional feature)

   **Delete Button:**
   - Insert > Button
   - Text: `"Delete"`
   - Visible: `SelectedExpense.Status.Value = "Submitted"`
   - OnSelect:
   ```powerapps
   Remove(Expenses, SelectedExpense);
   Navigate(DashboardScreen, ScreenTransition.UnCover);
   Notify("Expense deleted", NotificationType.Success)
   ```

#### Step 3.6: Connect Power Automate Flow
1. In Power Apps, go to **Data > Power Automate**
2. Click **"Add flow"**
3. Select your **"Expense Approval Workflow"**
4. The flow will now be available as `'ExpenseApprovalWorkflow'.Run()`

#### Step 3.7: Test the App
1. Click **"Play"** button (▶️) to preview app
2. Test all screens and functionality:
   - Dashboard displays correctly
   - Filters work
   - New expense creation works
   - Detail screen displays selected expense
   - Flow triggers on submission
3. Fix any issues found during testing

---

### Phase 4: Testing & Deployment

#### Step 4.1: Comprehensive Testing
1. **Functional Testing:**
   - Test all user scenarios
   - Verify data validation
   - Test approval workflow end-to-end
   - Test mobile responsiveness

2. **Security Testing:**
   - Verify users can only see their own expenses
   - Test manager approval permissions
   - Verify data access controls

3. **Performance Testing:**
   - Test with larger datasets
   - Verify gallery performance
   - Test flow execution times

#### Step 4.2: User Acceptance Testing
1. Share app with test users
2. Provide test scenarios
3. Collect feedback
4. Make necessary adjustments

#### Step 4.3: Production Deployment
1. **Save and Publish App:**
   - Click **"File" > "Save"**
   - Click **"Publish"**
   - Click **"Publish this version"**

2. **Share the App:**
   - Click **"Share"**
   - Add users/groups who need access
   - Set appropriate permissions
   - Send invitation

3. **Monitor and Support:**
   - Monitor app usage
   - Address user questions
   - Monitor flow runs for errors
   - Collect feedback for improvements

---

### Phase 5: Post-Deployment Activities

#### Step 5.1: User Training
1. Create user guide document
2. Conduct training sessions
3. Create video tutorials
4. Set up help desk support

#### Step 5.2: Monitoring and Maintenance
1. Monitor Power Automate flow runs
2. Review SharePoint list growth and performance
3. Monitor app usage analytics
4. Plan for future enhancements

#### Step 5.3: Future Enhancements
Based on user feedback, consider adding:
- Bulk expense upload
- Expense reporting dashboard
- Mobile app optimization
- Integration with accounting systems
- Advanced approval workflows

---

*These detailed build instructions provide step-by-step guidance for implementing the complete expense management solution.*

---

## Additional Considerations

### Mobile Optimization
- Ensure responsive design for mobile devices
- Test camera functionality for receipt capture
- Optimize touch targets for mobile use

### Accessibility
- Add proper labels for screen readers
- Ensure sufficient color contrast
- Support keyboard navigation

### Performance
- Use delegation-friendly formulas
- Limit gallery items with proper filtering
- Optimize image sizes for receipts

### Future Enhancements
- Bulk expense upload
- Expense categories management
- Reporting dashboard for managers
- Integration with accounting systems
- Mileage tracking
- Multi-currency support

---

*This design provides a comprehensive foundation for building a professional expense management application with Power Apps and Power Automate.*
