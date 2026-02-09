# Custom Slot Layout Guide
## For custom column/range mappings in Google Sheets

This guide explains how to use **your own Sheet layout** (custom rows/columns)
for slot management. It supports setups like:

- **Gatekeeper** slots in `Column A`, rows `3–5`
- **Runner** slots in `Column B`, rows `2–8`

The system will still **fill the first available slot** and prevent overbooking.

---

## Date-Based Custom Layout (May 2025)

Use this version when slots are grouped by **date** and **role**.

### Required Sheet Tab Name

Make sure the script uses the exact sheet tab name.  
If your tab is `Sheet6`, set `sheet: 'Sheet6'` in all ranges.

### Full Apps Script (Date-Based)

```javascript
/**
 * Open Day Registration - Date-based Custom Slot Layout
 * Uses fixed ranges per date and role.
 */

// Replace with your Google Sheet ID
const SPREADSHEET_ID = '1A6dyqt-RoJ762D2E3jiT_RjIH9ryGUdRyAtDxRFYht0';

// Slot ranges per date
const SLOT_RANGES_BY_DATE = {
  '2025-05-10': {
    gatekeeper: { sheet: 'Sheet6', range: 'B15:B19' },
    runner: { sheet: 'Sheet6', range: 'C15:C23' }
  },
  '2025-05-11': {
    gatekeeper: { sheet: 'Sheet6', range: 'D15:D19' },
    runner: { sheet: 'Sheet6', range: 'E15:E23' }
  },
  '2025-05-17': {
    gatekeeper: { sheet: 'Sheet6', range: 'B31:B35' },
    runner: { sheet: 'Sheet6', range: 'C31:C39' }
  },
  '2025-05-18': {
    gatekeeper: { sheet: 'Sheet6', range: 'D31:D35' },
    runner: { sheet: 'Sheet6', range: 'E31:E39' }
  }
};

function getSheet(rangeConfig) {
  return SpreadsheetApp.openById(SPREADSHEET_ID).getSheetByName(rangeConfig.sheet);
}

function getRangeValues(rangeConfig) {
  return getSheet(rangeConfig).getRange(rangeConfig.range).getValues();
}

function findFirstEmptyIndex(values) {
  for (let i = 0; i < values.length; i++) {
    if (!values[i][0]) return i;
  }
  return -1;
}

function getAvailableSlots(rangeConfig) {
  const values = getRangeValues(rangeConfig);
  return values.filter(row => !row[0]).length;
}

function getRangesForDate(date) {
  return SLOT_RANGES_BY_DATE[date] || null;
}

function doGet(e) {
  try {
    const date = e && e.parameter ? e.parameter.date : null;
    if (!date) {
      return ContentService.createTextOutput(JSON.stringify({
        ok: false,
        error: 'Missing date parameter'
      })).setMimeType(ContentService.MimeType.JSON);
    }

    const ranges = getRangesForDate(date);
    if (!ranges) {
      return ContentService.createTextOutput(JSON.stringify({
        ok: false,
        error: 'Invalid date'
      })).setMimeType(ContentService.MimeType.JSON);
    }

    const gatekeeperAvailable = getAvailableSlots(ranges.gatekeeper);
    const runnerAvailable = getAvailableSlots(ranges.runner);

    return ContentService.createTextOutput(JSON.stringify({
      ok: true,
      slots: {
        gatekeeper: { available: gatekeeperAvailable },
        runner: { available: runnerAvailable }
      }
    })).setMimeType(ContentService.MimeType.JSON);

  } catch (err) {
    return ContentService.createTextOutput(JSON.stringify({
      ok: false,
      error: err.toString()
    })).setMimeType(ContentService.MimeType.JSON);
  }
}

function doPost(e) {
  try {
    const data = JSON.parse(e.postData.contents);

    if (!data.name || !data.role || !data.event || !data.date) {
      return ContentService.createTextOutput(JSON.stringify({
        ok: false,
        error: 'Missing required fields: name, role, event, date'
      })).setMimeType(ContentService.MimeType.JSON);
    }

    const ranges = getRangesForDate(data.date);
    if (!ranges) {
      return ContentService.createTextOutput(JSON.stringify({
        ok: false,
        error: 'Invalid date'
      })).setMimeType(ContentService.MimeType.JSON);
    }

    const selectedRoles = data.role.split(',').map(r => r.trim().toLowerCase());

    for (const role of selectedRoles) {
      const rangeConfig = ranges[role];
      if (!rangeConfig) continue;

      const sheet = getSheet(rangeConfig);
      const range = sheet.getRange(rangeConfig.range);
      const values = range.getValues();
      const emptyIndex = findFirstEmptyIndex(values);

      if (emptyIndex === -1) {
        return ContentService.createTextOutput(JSON.stringify({
          ok: false,
          error: `${role} slots are full`
        })).setMimeType(ContentService.MimeType.JSON);
      }

      range.getCell(emptyIndex + 1, 1).setValue(data.name);
    }

    return doGet({ parameter: { date: data.date } });

  } catch (err) {
    return ContentService.createTextOutput(JSON.stringify({
      ok: false,
      error: err.toString()
    })).setMimeType(ContentService.MimeType.JSON);
  }
}
```

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

