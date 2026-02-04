# Custom Slot Layout Guide
## For custom column/range mappings in Google Sheets

This guide explains how to use **your own Sheet layout** (custom rows/columns)
for slot management. It supports setups like:

- **Gatekeeper** slots in `Column A`, rows `3–5`
- **Runner** slots in `Column B`, rows `2–8`

The system will still **fill the first available slot** and prevent overbooking.

---

## Overview

Instead of counting registrations in `Sheet1` and totals in `Sheet2`,
you will store **slot assignments directly** in your custom range.

When a user submits:
1. The script finds the first empty cell in the role’s range.
2. It writes the registrant’s name (or ID).
3. The UI reads how many slots are still empty.

---

## Step 1: Prepare Your Custom Sheet

Example layout (you can use any layout):

- **Gatekeeper** slots in `A3:A5`
- **Runner** slots in `B2:B8`

These cells should be **blank initially** (empty = available).

---

## Step 2: Add Slot Range Configuration

Open Apps Script and add this configuration at the top:

```javascript
// Slot ranges (custom layout)
const SLOT_RANGES = {
  gatekeeper: {
    sheet: 'Sheet1',
    range: 'A3:A5'
  },
  runner: {
    sheet: 'Sheet1',
    range: 'B2:B8'
  }
};
```

---

## Step 3: Helper Functions (Find Empty Slots)

Add these helper functions:

```javascript
function getRangeValues(rangeConfig) {
  const sheet = SpreadsheetApp.openById(SPREADSHEET_ID).getSheetByName(rangeConfig.sheet);
  return sheet.getRange(rangeConfig.range).getValues();
}

function findFirstEmptyIndex(values) {
  for (let i = 0; i < values.length; i++) {
    if (!values[i][0]) return i; // empty cell
  }
  return -1;
}

function getAvailableSlots(rangeConfig) {
  const values = getRangeValues(rangeConfig);
  return values.filter(row => !row[0]).length;
}
```

---

## Step 4: Update doGet() (Slot Counts)

Replace your slot count logic with this:

```javascript
function doGet() {
  const gatekeeperAvailable = getAvailableSlots(SLOT_RANGES.gatekeeper);
  const runnerAvailable = getAvailableSlots(SLOT_RANGES.runner);

  return ContentService
    .createTextOutput(JSON.stringify({
      ok: true,
      slots: {
        gatekeeper: { available: gatekeeperAvailable },
        runner: { available: runnerAvailable }
      }
    }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

---

## Step 5: Update doPost() (Fill First Empty Slot)

Replace the slot save section with:

```javascript
function doPost(e) {
  const data = JSON.parse(e.postData.contents);
  const selectedRoles = data.role.split(',').map(r => r.trim().toLowerCase());

  for (const role of selectedRoles) {
    const rangeConfig = SLOT_RANGES[role];
    if (!rangeConfig) continue;

    const sheet = SpreadsheetApp.openById(SPREADSHEET_ID).getSheetByName(rangeConfig.sheet);
    const range = sheet.getRange(rangeConfig.range);
    const values = range.getValues();
    const emptyIndex = findFirstEmptyIndex(values);

    if (emptyIndex === -1) {
      return ContentService.createTextOutput(JSON.stringify({
        ok: false,
        error: `${role} slots are full`
      })).setMimeType(ContentService.MimeType.JSON);
    }

    // Fill the first empty slot
    range.getCell(emptyIndex + 1, 1).setValue(data.name);
  }

  return doGet(); // return updated slot counts
}
```

---

## Step 6: Update form.html slot display

Because `doGet()` now returns only `available`, update slot display in `form.html`
to use only `available` instead of total/taken. Example:

```javascript
if (data.ok && data.slots) {
  SLOT_CONFIG.runner.total = data.slots.runner.available;
  SLOT_CONFIG.runner.taken = 0;
  SLOT_CONFIG.gatekeeper.total = data.slots.gatekeeper.available;
  SLOT_CONFIG.gatekeeper.taken = 0;
  updateSlotDisplay();
}
```

---

## Notes

- Empty cell = available slot  
- Filled cell = taken slot  
- The first empty cell is always used  
- You can expand or change ranges anytime  

---

## Example Custom Mapping

```javascript
const SLOT_RANGES = {
  gatekeeper: { sheet: 'Sheet1', range: 'A3:A5' },
  runner: { sheet: 'Sheet1', range: 'B2:B8' }
};
```

---

## Result

✅ Uses **your custom table design**  
✅ Fills the **first available slot**  
✅ Updates live in the web form  
✅ Prevents overbooking  

