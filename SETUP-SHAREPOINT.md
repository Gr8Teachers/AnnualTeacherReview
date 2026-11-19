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
    "designation": { "type": "string" },
    "location":    { "type": "string" },
    "stage":       { "type": "string" },
    "submittedOn": { "type": "string" },
    "goal1":       { "type": "string" },
    "goal2":       { "type": "string" },
    "goal3":       { "type": "string" },
    "goal4":       { "type": "string" },
    "otherGoals":  { "type": "string" },
    "planJson":    { "type": "string" }
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

- Submit checks that Teacher Name, Location, and the area + action strategies for Goals 1–4 are filled in.
- After submitting, teachers can keep editing their Goals and Action plans and click **Resubmit Goals and Action plans**. Their row in the list is updated (no duplicate).
- Until `SUBMIT_URL` is set, the button tells teachers to use **Save a PDF copy for me** and email it instead.
