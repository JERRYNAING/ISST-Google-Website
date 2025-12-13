# Slot Management System Setup Guide
## For Events/form.html

This guide explains how to implement **automatic slot management** using **Google Apps Script** and **Google Sheets** as the backend. When slots are filled, the numbers will automatically update in real-time.

---

## Overview

The system works as follows:
1. **Google Sheets** stores all registrations and calculates available slots
2. **Google Apps Script** provides API endpoints to:
   - **GET** current slot availability
   - **POST** new registrations and update slot counts
3. **form.html** fetches slot counts on page load and after each submission
4. Slots automatically update when registrations are submitted

---

## Step 1: Create Google Sheet

### 1.1 Create New Spreadsheet

1. Go to [Google Sheets](https://sheets.google.com)
2. Create a new spreadsheet
3. Name it: **"Open Day Registrations"** (or your preferred name)

### 1.2 Set Up Sheet Structure

**Sheet 1: Registrations** (for storing all registrations)
- **Row 1 (Headers):**
  - Column A: `Name`
  - Column B: `Role`
  - Column C: `Event`
  - Column D: `Timestamp`
  
**Sheet 2: Slot Configuration** (for managing slot limits)
- **Row 1 (Headers):**
  - Column A: `Role`
  - Column B: `Total Slots`
  - Column C: `Taken Slots`
  - Column D: `Available Slots`
  
- **Row 2:**
  - Column A: `Runner`
  - Column B: `20` (or your desired total)
  
- **Row 3:**
  - Column A: `Gatekeeper`
  - Column B: `15` (or your desired total)

**Note:** Column C and D in Sheet 2 will be auto-calculated by the script.

### 1.3 Get Your Spreadsheet ID

1. Look at your Google Sheet URL:
   ```
   https://docs.google.com/spreadsheets/d/SPREADSHEET_ID_HERE/edit
   ```
2. Copy the part between `/d/` and `/edit` - that's your **Spreadsheet ID**
3. Save this ID - you'll need it in the next step

---

## Step 2: Create Google Apps Script

### 2.1 Open Apps Script Editor

1. In your Google Sheet, go to **Extensions** > **Apps Script**
2. Delete any default code
3. You'll see a blank editor

### 2.2 Copy the Complete Script

Copy and paste this **complete Google Apps Script code**:

```javascript
/**
 * Open Day Registration System with Slot Management
 * Handles form submissions and real-time slot tracking
 */

// ============================================
// CONFIGURATION - UPDATE THESE VALUES
// ============================================

// Replace with your Google Sheet ID (from the Sheet URL)
const SPREADSHEET_ID = 'YOUR_SPREADSHEET_ID_HERE';

// Sheet names
const REGISTRATIONS_SHEET = 'Sheet1';  // Sheet for storing registrations
const SLOTS_SHEET = 'Sheet2';          // Sheet for slot configuration

// ============================================
// HELPER FUNCTIONS
// ============================================

/**
 * Get the spreadsheet object
 */
function getSpreadsheet() {
  return SpreadsheetApp.openById(SPREADSHEET_ID);
}

/**
 * Initialize sheets if they don't exist
 */
function initializeSheets() {
  const spreadsheet = getSpreadsheet();
  
  // Initialize Registrations Sheet
  let regSheet = spreadsheet.getSheetByName(REGISTRATIONS_SHEET);
  if (!regSheet) {
    regSheet = spreadsheet.insertSheet(REGISTRATIONS_SHEET);
    regSheet.getRange(1, 1, 1, 4).setValues([['Name', 'Role', 'Event', 'Timestamp']]);
    regSheet.getRange(1, 1, 1, 4).setFontWeight('bold');
  }
  
  // Initialize Slots Sheet
  let slotsSheet = spreadsheet.getSheetByName(SLOTS_SHEET);
  if (!slotsSheet) {
    slotsSheet = spreadsheet.insertSheet(SLOTS_SHEET);
    slotsSheet.getRange(1, 1, 1, 4).setValues([['Role', 'Total Slots', 'Taken Slots', 'Available Slots']]);
    slotsSheet.getRange(1, 1, 1, 4).setFontWeight('bold');
    
    // Set default slot values
    slotsSheet.getRange(2, 1, 1, 2).setValues([['Runner', 20]]);
    slotsSheet.getRange(3, 1, 1, 2).setValues([['Gatekeeper', 15]]);
  }
}

/**
 * Count registrations for each role
 */
function countRegistrations() {
  const spreadsheet = getSpreadsheet();
  const regSheet = spreadsheet.getSheetByName(REGISTRATIONS_SHEET);
  
  if (!regSheet) {
    return { runner: 0, gatekeeper: 0 };
  }
  
  const data = regSheet.getDataRange().getValues();
  let runnerCount = 0;
  let gatekeeperCount = 0;
  
  // Count registrations (skip header row)
  for (let i = 1; i < data.length; i++) {
    const role = data[i][1]; // Role is in column B (index 1)
    if (role) {
      const roleLower = role.toLowerCase();
      if (roleLower.includes('runner')) {
        runnerCount++;
      }
      if (roleLower.includes('gatekeeper')) {
        gatekeeperCount++;
      }
    }
  }
  
  return { runner: runnerCount, gatekeeper: gatekeeperCount };
}

/**
 * Get slot configuration from Sheet2
 */
function getSlotConfig() {
  const spreadsheet = getSpreadsheet();
  const slotsSheet = spreadsheet.getSheetByName(SLOTS_SHEET);
  
  if (!slotsSheet) {
    // Return defaults if sheet doesn't exist
    return {
      runner: { total: 20, taken: 0 },
      gatekeeper: { total: 15, taken: 0 }
    };
  }
  
  const data = slotsSheet.getDataRange().getValues();
  const config = {
    runner: { total: 20, taken: 0 },
    gatekeeper: { total: 15, taken: 0 }
  };
  
  // Read slot totals from Sheet2 (skip header row)
  for (let i = 1; i < data.length; i++) {
    const role = data[i][0]; // Role name in column A
    const total = data[i][1]; // Total slots in column B
    
    if (role && total) {
      const roleLower = role.toLowerCase();
      if (roleLower.includes('runner')) {
        config.runner.total = total;
      }
      if (roleLower.includes('gatekeeper')) {
        config.gatekeeper.total = total;
      }
    }
  }
  
  // Get actual taken counts from registrations
  const counts = countRegistrations();
  config.runner.taken = counts.runner;
  config.gatekeeper.taken = counts.gatekeeper;
  
  return config;
}

/**
 * Update slot counts in Sheet2
 */
function updateSlotSheet() {
  const spreadsheet = getSpreadsheet();
  const slotsSheet = spreadsheet.getSheetByName(SLOTS_SHEET);
  
  if (!slotsSheet) {
    initializeSheets();
    return updateSlotSheet();
  }
  
  const config = getSlotConfig();
  
  // Update Runner row (row 2)
  slotsSheet.getRange(2, 3).setValue(config.runner.taken); // Taken Slots
  slotsSheet.getRange(2, 4).setValue(config.runner.total - config.runner.taken); // Available Slots
  
  // Update Gatekeeper row (row 3)
  slotsSheet.getRange(3, 3).setValue(config.gatekeeper.taken); // Taken Slots
  slotsSheet.getRange(3, 4).setValue(config.gatekeeper.total - config.gatekeeper.taken); // Available Slots
}

// ============================================
// API ENDPOINTS
// ============================================

/**
 * Handle GET request - Returns current slot availability
 * This is called by form.html to fetch slot counts
 */
function doGet(e) {
  try {
    // Initialize sheets if needed
    initializeSheets();
    
    // Get current slot configuration
    const slots = getSlotConfig();
    
    // Return JSON response
    return ContentService
      .createTextOutput(JSON.stringify({
        ok: true,
        slots: {
          runner: {
            total: slots.runner.total,
            taken: slots.runner.taken,
            available: slots.runner.total - slots.runner.taken
          },
          gatekeeper: {
            total: slots.gatekeeper.total,
            taken: slots.gatekeeper.taken,
            available: slots.gatekeeper.total - slots.gatekeeper.taken
          }
        }
      }))
      .setMimeType(ContentService.MimeType.JSON);
      
  } catch (error) {
    return ContentService
      .createTextOutput(JSON.stringify({
        ok: false,
        error: error.toString()
      }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

/**
 * Handle POST request - Saves registration and updates slots
 * This is called when form.html submits the form
 */
function doPost(e) {
  try {
    // Initialize sheets if needed
    initializeSheets();
    
    // Parse the JSON payload
    const data = JSON.parse(e.postData.contents);
    
    // Validate required fields
    if (!data.name || !data.role || !data.event) {
      return ContentService
        .createTextOutput(JSON.stringify({
          ok: false,
          error: 'Missing required fields: name, role, and event are required'
        }))
        .setMimeType(ContentService.MimeType.JSON);
    }
    
    // Get current slot configuration
    const slots = getSlotConfig();
    
    // Check if selected roles have available slots
    const selectedRoles = data.role.split(',').map(r => r.trim());
    const errors = [];
    
    for (const role of selectedRoles) {
      const roleLower = role.toLowerCase();
      
      if (roleLower.includes('runner')) {
        const available = slots.runner.total - slots.runner.taken;
        if (available <= 0) {
          errors.push('Runner slots are full');
        }
      }
      
      if (roleLower.includes('gatekeeper')) {
        const available = slots.gatekeeper.total - slots.gatekeeper.taken;
        if (available <= 0) {
          errors.push('Gatekeeper slots are full');
        }
      }
    }
    
    // If any role is full, return error
    if (errors.length > 0) {
      return ContentService
        .createTextOutput(JSON.stringify({
          ok: false,
          error: errors.join(', ')
        }))
        .setMimeType(ContentService.MimeType.JSON);
    }
    
    // Save registration to Sheet1
    const spreadsheet = getSpreadsheet();
    const regSheet = spreadsheet.getSheetByName(REGISTRATIONS_SHEET);
    
    const timestamp = new Date();
    regSheet.appendRow([
      data.name,
      data.role,
      data.event,
      timestamp
    ]);
    
    // Update slot counts in Sheet2
    updateSlotSheet();
    
    // Get updated slot counts
    const updatedSlots = getSlotConfig();
    
    // Return success response with updated slot counts
    return ContentService
      .createTextOutput(JSON.stringify({
        ok: true,
        message: 'Registration successful',
        slots: {
          runner: {
            total: updatedSlots.runner.total,
            taken: updatedSlots.runner.taken,
            available: updatedSlots.runner.total - updatedSlots.runner.taken
          },
          gatekeeper: {
            total: updatedSlots.gatekeeper.total,
            taken: updatedSlots.gatekeeper.taken,
            available: updatedSlots.gatekeeper.total - updatedSlots.gatekeeper.taken
          }
        }
      }))
      .setMimeType(ContentService.MimeType.JSON);
      
  } catch (error) {
    // Return error response
    return ContentService
      .createTextOutput(JSON.stringify({
        ok: false,
        error: error.toString()
      }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}
```

### 2.3 Update Configuration

1. Find this line in the script:
   ```javascript
   const SPREADSHEET_ID = 'YOUR_SPREADSHEET_ID_HERE';
   ```
2. Replace `YOUR_SPREADSHEET_ID_HERE` with your actual Spreadsheet ID from Step 1.3
3. Click **Save** (💾 icon) or press `Ctrl+S` / `Cmd+S`
4. Name your project: Click on "Untitled project" at the top and rename it to "Open Day Registration System"

---

## Step 3: Deploy as Web App

### 3.1 Initial Deployment

1. Click **Deploy** > **New deployment**
2. Click the gear icon ⚙️ next to "Select type"
3. Choose **Web app**
4. Configure:
   - **Description**: "Open Day Registration with Slot Management"
   - **Execute as**: **Me** (your Google account)
   - **Who has access**: **Anyone** (⚠️ **IMPORTANT**: This allows the form to work publicly)
5. Click **Deploy**
6. **Copy the Web App URL** - it will look like:
   ```
   https://script.google.com/macros/s/AKfycby.../exec
   ```
7. Click **Done**

### 3.2 Authorize Permissions

When you first deploy, Google will ask for permissions:
1. Click **Review Permissions**
2. Choose your Google account
3. Click **Advanced** > **Go to [Project Name] (unsafe)**
4. Click **Allow**

This is safe - Google needs permission to access your Sheet.

---

## Step 4: Update form.html

### 4.1 Add the Google Apps Script URL

1. Open `Events/form.html`
2. Find this line (around line 342):
   ```javascript
   const GOOGLE_APPS_SCRIPT_URL = "https://script.google.com/macros/s/YOUR_SCRIPT_ID/exec";
   ```
3. Replace `https://script.google.com/macros/s/YOUR_SCRIPT_ID/exec` with your **actual Web App URL** from Step 3.1

### 4.2 Add Function to Fetch Slots from Backend

Find the `SLOT_CONFIG` section (around line 349) and replace it with this code that fetches slots from Google Apps Script:

```javascript
    /* ============================================
       SLOT CONFIGURATION
       ============================================ */
    // Slot configuration - will be loaded from Google Apps Script
    let SLOT_CONFIG = {
      runner: {
        total: 20,        // Will be updated from backend
        taken: 0          // Will be updated from backend
      },
      gatekeeper: {
        total: 15,        // Will be updated from backend
        taken: 0          // Will be updated from backend
      }
    };
    
    /**
     * Fetch slot counts from Google Apps Script
     * This function is called on page load and after form submission
     */
    async function fetchSlotCounts() {
      try {
        const response = await fetch(GOOGLE_APPS_SCRIPT_URL, {
          method: 'GET',
          headers: { 'Content-Type': 'application/json' }
        });
        
        const data = await response.json();
        
        if (data.ok && data.slots) {
          // Update SLOT_CONFIG with data from backend
          SLOT_CONFIG.runner.total = data.slots.runner.total;
          SLOT_CONFIG.runner.taken = data.slots.runner.taken;
          SLOT_CONFIG.gatekeeper.total = data.slots.gatekeeper.total;
          SLOT_CONFIG.gatekeeper.taken = data.slots.gatekeeper.taken;
          
          // Update the display
          updateSlotDisplay();
        } else {
          console.error('Failed to fetch slots:', data.error);
        }
      } catch (error) {
        console.error('Error fetching slots:', error);
        // Keep using default values if fetch fails
      }
    }
```

### 4.3 Update Form Submission to Use Backend Slot Data

Find the form submission handler (around line 712) and update the success handler to refresh slots:

```javascript
        if (res.ok && data.ok) {
          // Success: Show success message and reset form
          statusEl.textContent = 'Saved! Thank you for your registration.';
          statusEl.className = 'status ok';
          
          // Reset form and clear selections
          form.reset();
          clearAllErrors();
          
          // Update slot display with data from server response
          if (data.slots) {
            SLOT_CONFIG.runner.total = data.slots.runner.total;
            SLOT_CONFIG.runner.taken = data.slots.runner.taken;
            SLOT_CONFIG.gatekeeper.total = data.slots.gatekeeper.total;
            SLOT_CONFIG.gatekeeper.taken = data.slots.gatekeeper.taken;
            updateSlotDisplay();
          } else {
            // Fallback: fetch slots from backend
            fetchSlotCounts();
          }
          
          // Reset button after a delay
          setTimeout(() => {
            submitBtn.textContent = 'Submit';
          }, 2000);
        } else {
          throw new Error(data.error || 'Failed to save. Please try again.');
        }
```

### 4.4 Load Slots on Page Load

Find the initialization section (around line 638) and add:

```javascript
    /* ============================================
       INITIALIZE SLOT DISPLAY
       ============================================ */
    // Fetch slot counts from backend on page load
    fetchSlotCounts();
    
    // Also update display immediately (in case fetch is slow)
    updateSlotDisplay();
```

---

## Step 5: Test the System

### 5.1 Test Slot Fetching

1. Open `form.html` in a browser
2. Open browser Developer Tools (F12)
3. Check the Console tab
4. You should see slot counts loading automatically
5. The slot numbers should display correctly

### 5.2 Test Form Submission

1. Fill out the form:
   - Enter a name
   - Select a role (Runner or Gatekeeper)
   - Enter an event name
2. Click **Submit**
3. Check:
   - ✅ Form shows success message
   - ✅ Slot counts update automatically (e.g., "15 slots available" → "14 slots available")
   - ✅ Data appears in your Google Sheet

### 5.3 Test Slot Full Scenario

1. Fill all slots by submitting multiple registrations
2. Try to submit another registration for the same role
3. You should see an error: "Runner slots are full" (or "Gatekeeper slots are full")

---

## How It Works

### Automatic Slot Updates

1. **On Page Load:**
   - `form.html` calls `fetchSlotCounts()`
   - This sends a GET request to Google Apps Script
   - Script counts registrations in Sheet1
   - Returns current slot availability
   - Form displays updated numbers

2. **After Form Submission:**
   - Form sends POST request with registration data
   - Google Apps Script:
     - Validates slots are available
     - Saves registration to Sheet1
     - Updates slot counts in Sheet2
     - Returns updated slot counts in response
   - Form updates display with new numbers

3. **Real-Time Updates:**
   - Every time someone submits the form, slots decrease
   - When slots reach 0, the role checkbox is disabled
   - Numbers update immediately after each submission

---

## Troubleshooting

### Issue: "Script function not found"

**Solution:**
- Make sure function names are exactly `doGet` and `doPost` (case-sensitive)
- Check that you saved the script
- Redeploy the Web App

### Issue: "Access denied" or CORS errors

**Solution:**
- Make sure Web App is deployed with "Who has access: **Anyone**"
- Redeploy if you changed access settings
- Check browser console for specific error messages

### Issue: Slots not updating

**Solution:**
1. Check Google Apps Script execution logs:
   - In Apps Script editor, go to **View** > **Execution log**
   - Look for errors
2. Verify Spreadsheet ID is correct
3. Check that Sheet1 and Sheet2 exist
4. Test the Web App URL directly in browser:
   ```
   https://script.google.com/macros/s/YOUR_SCRIPT_ID/exec
   ```
   Should return JSON with slot data

### Issue: "Spreadsheet not found"

**Solution:**
- Double-check `SPREADSHEET_ID` in the script
- Make sure the spreadsheet is accessible (not in a restricted folder)
- Verify the ID is copied correctly (no extra spaces)

### Issue: Slot counts are wrong

**Solution:**
1. Check Sheet1 - count registrations manually
2. Check Sheet2 - verify "Total Slots" values are correct
3. Run `getSlotConfig()` function in Apps Script editor to debug
4. Check execution logs for errors

---

## Updating Slot Totals

To change the total number of slots:

1. Open your Google Sheet
2. Go to **Sheet2** (Slot Configuration)
3. Update the "Total Slots" column (Column B):
   - Row 2: Runner total slots
   - Row 3: Gatekeeper total slots
4. The script will automatically recalculate available slots

**OR** edit the script directly:
```javascript
// In getSlotConfig() function, change these defaults:
config.runner.total = 25;  // Change from 20 to 25
config.gatekeeper.total = 20;  // Change from 15 to 20
```

Then redeploy the Web App.

---

## Security Notes

1. **Public Access**: The Web App is set to "Anyone" to allow public form submissions. This means anyone with the URL can submit data.

2. **Rate Limits**: Google Apps Script has quotas:
   - 20,000 requests per day (free tier)
   - 6 minutes execution time per request
   - For high-traffic events, consider upgrading

3. **Data Validation**: The script validates:
   - Required fields are present
   - Slots are available before saving
   - Prevents overbooking

4. **Sheet Protection**: Consider protecting Sheet2 (Slot Configuration) to prevent accidental edits to slot totals.

---

## Advanced: Multiple Events

If you need to track slots for multiple events:

1. Add an "Event" column to Sheet2
2. Modify the script to filter by event name
3. Update `getSlotConfig()` to accept an event parameter
4. Pass event name in GET/POST requests

---

## Support

For issues:
1. Check the troubleshooting section above
2. Review Apps Script execution logs: **View** > **Execution log**
3. Test the Web App URL directly in a browser
4. Check browser console for JavaScript errors
5. Verify form is sending correct data (check Network tab in DevTools)

---

## Summary

✅ **Google Sheets** stores registrations and slot configuration  
✅ **Google Apps Script** provides API endpoints  
✅ **form.html** fetches and displays slot counts  
✅ **Slots auto-update** when registrations are submitted  
✅ **No overbooking** - system prevents full slots from being selected  

The system is now fully functional and will automatically manage slots in real-time!

