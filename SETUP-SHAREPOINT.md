# Setting up the Submit button (SharePoint list + Power Automate)

When a teacher clicks **Submit Goals and Action plans**, the form sends their plan to a Power Automate flow. The flow saves it as one row per teacher in a SharePoint list. Head Teachers (and teachers) open the list to read the plans.

You do this once. It takes about 20 minutes.

> **Licence:** the trigger used here, *When an HTTP request is received*, is a **Premium** connector in Power Automate. Check that your account (or the person building the flow) has a Power Automate Premium / Per-user licence. If not, ask your M365 / IT team.

---

## 1. Create the SharePoint list

1. Open your team's SharePoint site → **New → List → Blank list**.
2. Name it **Annual Teacher Review**.
3. Rename the built-in **Title** column to **Teacher Name**.
4. Add these columns (**+ Add column**):

| Column name   | Type                                   |
|---------------|----------------------------------------|
| Teacher       | Person (allow people only, one person) |
| Emp_ID        | Single line of text                    |
| Link Token    | Single line of text (see part 9)       |
| Designation   | Single line of text                    |
| Location      | Single line of text                    |
| Stage         | Single line of text                    |
| Submitted On  | Date and time (include time)           |
| Goal 1        | Multiple lines of text (plain text)    |
| Goal 2        | Multiple lines of text (plain text)    |
| Goal 3        | Multiple lines of text (plain text)    |
| Goal 4        | Multiple lines of text (plain text)    |
| Other Goals   | Multiple lines of text (plain text)    |
| Plan JSON     | Multiple lines of text (plain text)    |

Column labels for the four goals in the form:
Goal 1 – Teaching [1], Goal 2 – Teaching [2], Goal 3 – Section Contribution, Goal 4 – Personal Development [Work related].

*Tip:* hide **Plan JSON** from the default view (it's the full saved form, used for reloading a plan, and isn't meant for reading).

**Permissions:** give Head Teachers read access to the list. Every row is created by the flow's account, not by the teacher, so SharePoint can't automatically limit each teacher to their own row. Either give teachers no access (they keep their own copy with **Save a PDF copy for me**), or give the whole team read access if everyone may see everyone's plans.

---

## 2. Build the flow

Go to **make.powerautomate.com → Create → Instant cloud flow → Skip**, then add these steps.

### Step A: Trigger: *When an HTTP request is received*
- **Who can trigger the flow:** **Anyone**
- Leave the JSON schema **empty** (the form sends plain text; it is parsed in the next step).

### Step B: *Parse JSON*
- **Content:** `Body` (from the trigger)
- **Schema:** paste this:

```json
{
  "type": "object",
  "properties": {
    "teacherName": { "type": "string" },
    "firstName":   { "type": "string" },
    "lastName":    { "type": "string" },
    "designation": { "type": "string" },
    "location":    { "type": "string" },
    "stage":       { "type": "string" },
    "submittedOn": { "type": "string" },
    "goal1":       { "type": "string" },
    "goal2":       { "type": "string" },
    "goal3":       { "type": "string" },
    "goal4":       { "type": "string" },
    "otherGoals":  { "type": "string" },
    "planJson":    { "type": "string" },
    "token":       { "type": "string" },
    "signLink":    { "type": "string" },
    "planLink":    { "type": "string" },
    "sendEmail":   { "type": "string" }
  }
}
```

### Step C: SharePoint *Get items*
- **Site Address:** your site
- **List Name:** Annual Teacher Review
- **Filter Query:** `Title eq '@{replace(body('Parse_JSON')?['teacherName'], '''', '''''')}'`
  (finds an existing row for this teacher so a resubmission updates it instead of adding a duplicate)
- **Top Count:** `1`

### Step D: *Condition*
- `length(body('Get_items')?['value'])` **is greater than** `0`

**If yes → SharePoint *Update item***
- **Id:** `first(body('Get_items')?['value'])?['ID']`
- Map the fields (below)

**If no → SharePoint *Create item***
- Map the fields (below)

Field mapping (both actions):

| List column   | Value from *Parse JSON* |
|---------------|-------------------------|
| Teacher Name (Title) | teacherName |
| Teacher **Claims** (Person column) | see part 8 |
| Emp ID        | see part 8 |
| Link Token    | token (see part 9) |
| Designation   | designation |
| Location      | location |
| Stage         | stage |
| Submitted On  | submittedOn |
| Goal 1 … Goal 4 | goal1 … goal4 |
| Other Goals   | otherGoals |
| Plan JSON     | planJson |

### Step E: *Response* (after the Condition, outside both branches)
- **Status Code:** `200`
- **Headers:** key `Access-Control-Allow-Origin`, value `*`
- **Body:** `{"ok": true}`

The header is required: without it the form can't read the reply and will show "Could not submit" even though the row was saved.

**Save** the flow. Open Step A again and copy the **HTTP POST URL**.

---

## 3. Connect the form to the flow

1. In this repository, open `index.html` → **Edit** (pencil icon).
2. Find this line near the bottom (search for `SUBMIT_URL`):

   ```js
   const SUBMIT_URL = '';
   ```

3. Paste the flow URL between the quotes, e.g.

   ```js
   const SUBMIT_URL = 'https://prod-00.australiaeast.logic.azure.com:443/workflows/...';
   ```

4. **Commit changes.** GitHub Pages updates within a minute or two.

> The URL works like a key: anyone who has it can add rows to the list. It is visible in the page source, so treat the list as internal and don't put anything confidential in it.

---

## 4. Test

1. Open the form, fill in Teacher Name, Location and Goals 1–4 (area + action strategies).
2. Click **Submit Goals and Action plans**. You should see *"Submitted. Your Head Teacher can now read your plan."*
3. Check the SharePoint list for the new row.
4. Submit again with the same name: the existing row should update (no duplicate).

## How it behaves in the form

- Submit checks that every field in Goals and Action plans is filled in (name, designation, location, and area / action / review strategies for every goal).
- After submitting, teachers can keep editing their Goals and Action plans and click **Resubmit Goals and Action plans**. Their row in the list is updated (no duplicate).
- Until `SUBMIT_URL` is set, the button tells teachers to use **Save a PDF copy for me** and email it instead.

---

## 5. Head Teacher import: the "read" flow

Head Teachers can click **Import from MS List** (visible in Head Teacher mode) to load any teacher's submitted plan into the form. This uses a second, separate flow that reads the list.

> **Keep this flow's link private.** Anyone with it can read every plan on the list. That's why it is **not** put in `index.html` (which is public on GitHub). Each Head Teacher pastes it into the form once, and it is saved only in their own browser. Share it with Head Teachers only, e.g. in a private Teams chat.

### Build it

**make.powerautomate.com → Create → Instant cloud flow → Skip**, name it e.g. *ATR – read list*.

1. **Trigger: *When an HTTP request is received***
   - **Who can trigger the flow:** Anyone
   - Leave the schema empty.

2. **SharePoint *Get items***
   - **Site Address / List Name:** the same Annual Teacher Review list
   - **Top Count:** `500` (under *Advanced parameters* / *Show all*)

3. **Response**
   - Click **+** under Get items → **Add an action** → search **Response** (it's under *Request*).
   - **Status Code:** `200`
   - **Headers:** `Access-Control-Allow-Origin` = `*`
   - **Body:** click in the box → ⚡ lightning icon → under **Get items** choose **body/value** (shown as `value`).

That's all; the form picks out Teacher Name, Location, Submitted On and Plan JSON itself.

*Optional (smaller, faster reply):* if you see a **Select** action (search "Select", connector **Data Operation**), you can put it between Get items and Response, with **From** = `value` and rows `name` → Title, `location` → Location, `submittedOn` → Submitted On, `planJson` → Plan JSON, then use its **Output** as the Response body. It isn't required.

**Save**, then copy the **HTTP URL** from the trigger.

### Use it

1. Open the form → **Head Teacher** → enter the password.
2. Click **Import from MS List**. The first time, paste the read-flow URL and click **Save and load list**.
3. Search or scroll, click a teacher's name, and their plan fills the form.

**Change flow link** in the import window replaces the saved link (e.g. if the flow is rebuilt). Each Head Teacher's browser keeps its own copy, so a new computer or browser needs the link pasted again.

---

## 6. Head Teacher changes (v2)

In Head Teacher mode the **Save modified plan** button (under the goals) does two things:

1. Updates the teacher's row on the list with the Head Teacher's version. The **Stage** column changes to **Goals and Action plans - v2 by HT**, and **Submitted On** shows when the Head Teacher saved it. It uses the same submit flow, so there is nothing extra to set up.
2. Saves a PDF named "… v2". Send this to the teacher; they open it with **Open my PDF copy** for their Progress review.

**Approving a plan.** Under the goals there is an **Approved** tick box that only the Head Teacher can tick (the plan must be complete). Until it is ticked, teachers can submit and resubmit as often as they like. Once it is ticked:

- In the Head Teacher's view, **Resubmit Goals and Action plans** updates the list only, and **Save modified plan** updates the list and saves the v2 PDF. Either one sets **Stage** to **Goals and Action plans - Approved by HT**.
- When the teacher opens that PDF with **Open my PDF copy**, their Goals and Action plans are read-only: no Submit/Resubmit button, no ✨ Ideas, no adding or removing goals, and Clear is turned off. They can still go on to the Progress review.
- Only the Head Teacher can change an approved plan. Untick **Approved** to let the teacher edit and resubmit again.

---

## 7. Steps 4 and 5: Annual Review Form and employee signature

**Step 4 – Annual Review Form (Head Teacher only).** Employee details come from the top of the form (Last name, First name, Emp I.D.; Classification = Designation; Location). Add the Emp I.D. if the teacher left it blank, tick the Line Manager boxes, add an explanation if work performance or conduct is not ticked, then Name / Signature / Date. **Print / PDF – Annual Review Form + Development Plan** saves one PDF: the form on a portrait page (laid out like the official Word form) followed by the full plan in landscape. Send it to the teacher.

**Step 5 – Employee signature (teacher).** The teacher opens that PDF with **Open my PDF copy**, goes to step 5, reviews the form and plan, ticks satisfied / not satisfied, signs and dates, then clicks **Save final PDF** and uploads it to People@TAFE via a KIC form.

Mixed portrait + landscape pages print correctly in **Chrome and Edge**; use one of those for these PDFs.

---

## 8. Teacher as a Person column (for Quick Steps, rules and flows)

The **Title** (Teacher Name) column can't be changed into a Person column, so the list gets a separate **Teacher** Person column. Teachers don't enter an email: the flow looks them up in a **Staff** list that you keep, using the first and last name they type (the form sends `firstName` and `lastName`).

### 8a. Create the Staff lookup list (same site)

| Column | Type |
|---|---|
| Title → rename to **Last Name** | (built-in) |
| **First Name** | Single line of text |
| **Staff** | Person (one person) |
| **Emp_ID** | Single line of text |

Add one row per teacher, with the names spelled the way teachers will type them on the form.

### 8b. Add columns to Annual Teacher Review
- **+ Add column → Person** → name it **Teacher** (allow one person).
- **+ Add column → Single line of text** → name it **Emp ID**.

### 8c. Change the submit flow
1. **Parse JSON**: add to the schema, under `teacherName`:
   ```
   "firstName": { "type": "string" },
   "lastName":  { "type": "string" },
   ```
2. **New step after Parse JSON: SharePoint *Get items*** – rename it **Get staff**.
   - **List Name:** Staff
   - **Filter Query:** `Title eq '@{replace(body('Parse_JSON')?['lastName'], '''', '''''')}' and First_x0020_Name eq '@{replace(body('Parse_JSON')?['firstName'], '''', '''''')}'`
     (`Title` is the Last Name column. If First Name's internal name differs, copy it from the column's settings URL: `…Field=First_x0020_Name`.)
   - **Top Count:** `1`
3. In **Update item** and **Create item**, set **Teacher Claims** to this expression:
   `first(body('Get_staff')?['value'])?['Staff']?['Claims']`
   and set **Emp ID** to:
   `first(body('Get_staff')?['value'])?['Emp_x005f_ID']`
4. **Save.** The read flow doesn't change.

### Things to know
- Matching isn't case-sensitive, but spelling must match the Staff list (e.g. *Jen* vs *Jennifer*). If there's no match, Teacher is left empty and the row still saves; fix the Staff list or pick the person by hand. If your flow errors on an empty match instead, wrap the expression: `if(empty(body('Get_staff')?['value']), null, first(body('Get_staff')?['value'])?['Staff']?['Claims'])`.
- Rows submitted earlier stay empty until the teacher resubmits (or you pick the person by hand).
- With Teacher filled you can filter or group by teacher, use Quick Steps, or build a *When an item is created or modified* flow that emails the Teacher when Stage changes to *v2 by HT*.
- Teachers no longer enter their Emp I.D. When the Head Teacher uses **Import from MS List**, the Emp ID from the row is filled in automatically and appears on the Annual Review Form (step 4). The Head Teacher can still type or correct it in step 4.

---

## 9. "Ready for Employee signature": email the teacher a link to sign

When everything in step 4 is signed, the Head Teacher ticks **Ready for Employee signature**. The form updates the teacher's row (Stage **Ready for employee signature**) with a secret **Link Token**, and the flow emails the teacher a link like `https://gr8teachers.github.io/AnnualTeacherReview/?sign=…`.

The teacher clicks the link, the completed form and plan load, and they go straight to step 5 to tick, sign and date. When they click **Save final PDF**, the row's Stage changes to **Signed by employee**. The Head Teacher also sees the link under the tick box (**Copy link**, **Resend email**).

### 9a. List
Add a column **Link Token** (Single line of text) and hide it from the view. Check its internal name (column settings → `Field=` in the address), e.g. `Link_x0020_Token`, `Link_x005f_Token` or `LinkToken`.

### 9b. Submit flow
1. **Parse JSON**: add `token`, `signLink` and `sendEmail` to the schema (see part 2, Step B).
2. **Update item** and **Create item**: set **Link Token** to `token`.
3. After the **Condition** (before **Response**), add another **Condition**: `sendEmail` **is equal to** `yes`.
   - **If yes → Office 365 Outlook *Send an email (V2)***
     - **To:** expression `first(body('Get_staff')?['value'])?['Staff']?['Email']`
     - **Subject:** `Your Annual Review is ready to sign`
     - **Body:** e.g. *Hi* `firstName`*, your Head Teacher has completed your Annual Review Form. Open this link, review the form and your plan, then tick, sign and date at step 5:* `signLink`
   - **If no:** leave empty.

The teacher must be on the Staff list (that's where the email address comes from), otherwise the flow fails and the form shows "Could not send".

### 9c. New flow: fetch a plan for signing
**Create → Instant cloud flow → Skip**:

1. **When an HTTP request is received** – Who can trigger: **Anyone**.
2. **Parse JSON** – Content: `Body`; Schema:
   ```json
   { "type": "object", "properties": { "token": { "type": "string" } } }
   ```
3. **Condition** – expression `length(coalesce(body('Parse_JSON')?['token'], ''))` **is greater than or equal to** `30` (stops anyone fetching rows without a real token).
   - **If yes:**
     - SharePoint **Get items** – List: Annual Teacher Review; **Filter Query:** `Link_x0020_Token eq '@{body('Parse_JSON')?['token']}'` (use your column's internal name); **Top Count:** `1`.
     - **Response** – Status `200`; Header `Access-Control-Allow-Origin` = `*`; Body: `value` from Get items.
   - **If no:**
     - **Response** – Status `404`; Header `Access-Control-Allow-Origin` = `*`; Body: `{"ok":false}`.
4. **Save**, copy the trigger's **HTTP POST URL**, then in `index.html` on GitHub (pencil icon) paste it between the quotes in `const SIGN_FETCH_URL = '';` and commit.

Anyone with a teacher's link can open that teacher's form, so treat the email like any other private staff email. Each link only opens its own row.

---

## 10. "Open plan" link on each row of the list

Every submission now sends a permanent link for that teacher's plan (`planLink`), e.g. `https://gr8teachers.github.io/AnnualTeacherReview/?plan=…`. Click it from the MS List and the latest version saved on the list opens in the form; switch to **Head Teacher** to review it. (It uses the same **Sign link** flow and **Link** column as part 9, so nothing else is needed.)

### 10a. List
1. **+ Add column → Single line of text** → name it **Open plan**.
2. Make it show as a clickable "Open plan" link: column heading ▾ → **Column settings → Format this column → Advanced mode**, paste this and **Save**:
   ```json
   {
     "$schema": "https://developer.microsoft.com/json-schemas/sp/v2/column-formatting.schema.json",
     "elmType": "a",
     "txtContent": "=if(@currentField == '', '', 'Open plan')",
     "attributes": { "href": "@currentField", "target": "_blank" },
     "style": { "font-weight": "600" }
   }
   ```

### 10b. Submit flow
1. **Parse JSON**: add `"planLink": { "type": "string" },` to the schema.
2. **Update item** and **Create item**: set **Open plan** to `planLink`.

Rows submitted before this change get their link the next time the teacher submits, or when you click **Save modified plan**, **Resubmit** or **Ready for Employee signature** on that plan.

If this device has changes that haven't been sent to the list yet (for example, Final review comments you haven't saved), the form asks before replacing them with the version on the list.

---

## 11. The full-year workflow (emails at every stage)

| # | Who | What happens | Stage on the list | Email |
|---|---|---|---|---|
| 1 | Teacher | Submits Goals and Action plans | Goals and Action plans | – |
| 2a | HT | Ticks **Approved** | Goals and Action plans - Approved by HT | Teacher: link to the approved plan (view / Print / PDF) |
| 2b | HT | Changes the plan → **Save modified plan & email teacher** | Goals and Action plans - v2 by HT | Teacher: link to the updated plan |
| 3 | Teacher | Reviews, changes, **Resubmit** | Goals and Action plans - resubmitted | – (watch the list) → back to 2a |
| 4 | HT | Runs the **Mid-year progress check** flow (all approved plans at once) | Progress check sent | Teacher: link to fill in the progress review |
| 5 | Teacher | Status + comments per goal → **Submit progress review** (then read-only) | Mid-year progress submitted | – (watch the list) |
| 6 | HT | Adds **Head Teacher comment** per goal → **Send comments to teacher** | Mid-year reviewed by HT | Teacher: your comments + link to the plan |
| 7 | HT | Final review, Annual Review Form, **Ready for Employee signature** | Ready for employee signature | Teacher: link to sign + People@TAFE steps |
| 8 | Teacher | Signs → **Save final PDF** | Signed by employee | – |

The **Mid-year progress** column shows *Not yet submitted* / *Submitted (date)* / *Reviewed by HT (date)*.

The form writes every email itself (subject + HTML), so the submit flow has **one** Send email step for all of them.

### 11a. List
Add a column **Mid-year progress** (Single line of text).

### 11b. Submit flow
1. **Parse JSON** – replace the schema with:
   ```json
   {
     "type": "object",
     "properties": {
       "teacherName": { "type": "string" }, "firstName": { "type": "string" }, "lastName": { "type": "string" },
       "designation": { "type": "string" }, "location": { "type": "string" },
       "stage": { "type": "string" }, "submittedOn": { "type": "string" },
       "goal1": { "type": "string" }, "goal2": { "type": "string" }, "goal3": { "type": "string" }, "goal4": { "type": "string" },
       "otherGoals": { "type": "string" }, "planJson": { "type": "string" },
       "token": { "type": "string" }, "planLink": { "type": "string" }, "signLink": { "type": "string" },
       "midYear": { "type": "string" },
       "sendEmail": { "type": "string" }, "emailSubject": { "type": "string" }, "emailHtml": { "type": "string" }
     }
   }
   ```
2. **Update item** and **Create item**: set **Mid-year progress** = `midYear` (and **Open_Plan** = `planLink`, **Link** = `token` if not already done).
3. **Condition 1** (`sendEmail` is equal to `yes`) → **Send an email (V2)**:
   - **To:** `first(body('Get_staff')?['value'])?['Staff']?['Email']` (unchanged)
   - **Subject:** `emailSubject` (dynamic content from Parse JSON)
   - **Body:** in code view (**</>**), replace everything with `@{body('Parse_JSON')?['emailHtml']}`

### 11c. New flow: Mid-year progress check (run once at mid-year)
**Create → Instant cloud flow → Manually trigger a flow**:

1. SharePoint **Get items** – List: Annual Teacher Review; **Filter Query:** `Stage eq 'Goals and Action plans - Approved by HT'`
2. **Apply to each** – Select an output: `value` from Get items. Inside it:
   - **Send an email (V2)**
     - **To:** `items('Apply_to_each')?['Teacher']?['Email']`
     - **Subject:** `Mid-year progress review – your Annual Review Development Plan`
     - **Body** (code view):
       ```html
       <div style="font-family:Calibri,Arial,sans-serif;font-size:11pt;line-height:1.35">
       <p style="margin:0 0 8px">Hi @{first(split(items('Apply_to_each')?['Title'], ' '))},</p>
       <p style="margin:0 0 8px">It's time for your mid-year progress review.</p>
       <p style="margin:0">Please open your plan to:</p>
       <ul style="margin:0 0 8px;padding-left:20px">
       <li style="margin:0">choose a <b>Progress review</b> status for each goal</li>
       <li style="margin:0">write your progress comments</li>
       <li style="margin:0">click <b>Submit progress review</b></li>
       </ul>
       <p style="margin:6px 0 14px 20px"><a href="@{items('Apply_to_each')?['Open_Plan']}" style="display:inline-block;padding:10px 18px;background:#1f7a66;color:#ffffff;text-decoration:none;border-radius:6px"><b>Complete your mid-year progress review</b></a></p>
       <p style="margin:0">Thank you.</p>
       </div>
       ```
   - SharePoint **Update item** – List: Annual Teacher Review; **Id:** `items('Apply_to_each')?['ID']`; **Title:** `items('Apply_to_each')?['Title']`; **Stage:** `Progress check sent`; **Mid-year progress:** `Not yet submitted`.
3. **Save**. At mid-year open the flow and click **Run**.

Only rows whose Stage is exactly *Goals and Action plans - Approved by HT* are emailed. When a teacher opens the link, the form sees *Progress check sent* and opens the Progress review for them.
