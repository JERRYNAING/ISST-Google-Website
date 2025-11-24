# Events Form - Google Sheets Backend Setup Guide

## Overview
This form (`form.html`) uses **Google Sheets as the backend database** via **Google Apps Script**. This is a free, serverless solution that doesn't require any custom backend server.

## Quick Setup Guide

### Step 1: Create a Google Sheet

1. Go to [Google Sheets](https://sheets.google.com)
2. Create a new spreadsheet
3. Name it (e.g., "Event Registrations")
4. Set up the header row in Row 1:
   - **Column A**: `Name`
   - **Column B**: `Role`
   - **Column C**: `Event`
   - **Column D**: `Timestamp`

### Step 2: Create Google Apps Script

1. In your Google Sheet, go to **Extensions** > **Apps Script**
2. Delete any default code
3. Copy and paste the Google Apps Script code below (see "Google Apps Script Code" section)
4. Update the `SPREADSHEET_ID` in the script (found in your Google Sheet URL)

### Step 3: Deploy as Web App

1. Click **Deploy** > **New deployment**
2. Click the gear icon ⚙️ next to "Select type" and choose **Web app**
3. Configure:
   - **Description**: "Event Registration Form Handler"
   - **Execute as**: **Me** (your account)
   - **Who has access**: **Anyone** (important for public forms)
4. Click **Deploy**
5. **Copy the Web App URL** (it will look like: `https://script.google.com/macros/s/.../exec`)
6. Click **Done**

### Step 4: Update the Form

1. Open `form.html`
2. Find the line: `const GOOGLE_APPS_SCRIPT_URL = "..."`
3. Replace the placeholder URL with your Web App URL from Step 3

## Google Apps Script Code

Copy this code into your Google Apps Script editor:

```javascript
/**
 * Event Registration Form Handler
 * Saves form submissions to Google Sheets
 */

// Replace with your Google Sheet ID (found in the Sheet URL)
const SPREADSHEET_ID = 'YOUR_SPREADSHEET_ID_HERE';

// Sheet name (usually 'Sheet1' for new sheets)
const SHEET_NAME = 'Sheet1';

/**
 * Handle POST request from the form
 */
function doPost(e) {
  try {
    // Parse the JSON payload
    const data = JSON.parse(e.postData.contents);
    
    // Validate required fields
    if (!data.name || !data.role || !data.event) {
      return ContentService
        .createTextOutput(JSON.stringify({
          ok: false,
          error: 'Missing required fields'
        }))
        .setMimeType(ContentService.MimeType.JSON);
    }
    
    // Get the spreadsheet
    const spreadsheet = SpreadsheetApp.openById(SPREADSHEET_ID);
    const sheet = spreadsheet.getSheetByName(SHEET_NAME);
    
    // If sheet doesn't exist, create it with headers
    if (!sheet) {
      const newSheet = spreadsheet.insertSheet(SHEET_NAME);
      newSheet.getRange(1, 1, 1, 4).setValues([['Name', 'Role', 'Event', 'Timestamp']]);
      newSheet.getRange(1, 1, 1, 4).setFontWeight('bold');
      return doPost(e); // Retry with new sheet
    }
    
    // Add the data to the sheet
    const timestamp = new Date();
    sheet.appendRow([
      data.name,
      data.role,
      data.event,
      timestamp
    ]);
    
    // Return success response
    return ContentService
      .createTextOutput(JSON.stringify({
        ok: true,
        message: 'Registration successful'
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

/**
 * Handle GET request (optional - for testing)
 */
function doGet(e) {
  return ContentService
    .createTextOutput(JSON.stringify({
      message: 'Event Registration API is running',
      status: 'ok'
    }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

### How to Find Your Spreadsheet ID

The Spreadsheet ID is in your Google Sheet URL:
```
https://docs.google.com/spreadsheets/d/SPREADSHEET_ID_HERE/edit
```

Copy the part between `/d/` and `/edit` - that's your Spreadsheet ID.

## Advanced: Slot Management

If you want to track available slots dynamically from Google Sheets, you can enhance the script:

```javascript
/**
 * Get available slots for each role
 */
function getSlots() {
  const spreadsheet = SpreadsheetApp.openById(SPREADSHEET_ID);
  const sheet = spreadsheet.getSheetByName(SHEET_NAME);
  
  if (!sheet) {
    return {
      runner: { total: 20, taken: 0 },
      gatekeeper: { total: 15, taken: 0 }
    };
  }
  
  const data = sheet.getDataRange().getValues();
  let runnerCount = 0;
  let gatekeeperCount = 0;
  
  // Count registrations (skip header row)
  for (let i = 1; i < data.length; i++) {
    const role = data[i][1]; // Role is in column B
    if (role && role.includes('Runner')) runnerCount++;
    if (role && role.includes('Gatekeeper')) gatekeeperCount++;
  }
  
  return {
    runner: { total: 20, taken: runnerCount },
    gatekeeper: { total: 15, taken: gatekeeperCount }
  };
}

/**
 * Enhanced doPost with slot checking
 */
function doPost(e) {
  try {
    const data = JSON.parse(e.postData.contents);
    
    // Check slots before saving
    const slots = getSlots();
    const selectedRoles = data.role.split(',').map(r => r.trim());
    
    for (const role of selectedRoles) {
      const roleKey = role.toLowerCase();
      if (roleKey.includes('runner')) {
        const available = slots.runner.total - slots.runner.taken;
        if (available <= 0) {
          return ContentService
            .createTextOutput(JSON.stringify({
              ok: false,
              error: 'Runner slots are full'
            }))
            .setMimeType(ContentService.MimeType.JSON);
        }
      }
      if (roleKey.includes('gatekeeper')) {
        const available = slots.gatekeeper.total - slots.gatekeeper.taken;
        if (available <= 0) {
          return ContentService
            .createTextOutput(JSON.stringify({
              ok: false,
              error: 'Gatekeeper slots are full'
            }))
            .setMimeType(ContentService.MimeType.JSON);
        }
      }
    }
    
    // Continue with normal save logic...
    // (use the code from the basic version above)
    
  } catch (error) {
    return ContentService
      .createTextOutput(JSON.stringify({
        ok: false,
        error: error.toString()
      }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}
```

## Form Data Structure

### Request Payload
The form sends a POST request with the following JSON structure:

```json
{
  "name": "John Doe",
  "role": "Runner, Gatekeeper",  // Comma-separated if multiple roles selected
  "event": "Open Day 2024"
}
```

### Expected Response

**Success Response:**
```json
{
  "ok": true,
  "message": "Registration successful"
}
```

**Error Response:**
```json
{
  "ok": false,
  "error": "Error message here"
}
```

## Testing Your Setup

### Test the Google Apps Script

1. In Apps Script editor, click the **Run** button (▶️)
2. Select `doGet` function
3. Click **Run**
4. You should see: `{"message":"Event Registration API is running","status":"ok"}`

### Test the Form

1. Open `form.html` in a browser
2. Fill out the form
3. Submit
4. Check your Google Sheet - the data should appear

### Test with curl (Optional)

```bash
curl -X POST "https://script.google.com/macros/s/YOUR_SCRIPT_ID/exec" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test User",
    "role": "Runner",
    "event": "Test Event"
  }'
```

## Troubleshooting

### Common Issues

1. **"Script function not found"**
   - Make sure the function is named `doPost` (case-sensitive)
   - Check that you saved the script

2. **"Access denied" or CORS errors**
   - Make sure you deployed with "Who has access: **Anyone**"
   - Redeploy the Web App if you changed access settings

3. **"Spreadsheet not found"**
   - Check that `SPREADSHEET_ID` is correct
   - Make sure the spreadsheet is accessible (not in a restricted folder)

4. **Data not appearing in Sheet**
   - Check Apps Script execution logs: **View** > **Execution log**
   - Verify the sheet name matches `SHEET_NAME` in the script
   - Check that the script has permission to access the sheet

5. **"Missing required fields" error**
   - Check browser console for the actual payload being sent
   - Verify form field names match: `name`, `role`, `event`

### Permission Issues

When you first run the script, Google will ask for permissions:
1. Click **Review Permissions**
2. Choose your Google account
3. Click **Advanced** > **Go to [Project Name] (unsafe)**
4. Click **Allow**

This is safe - it's just Google's way of asking permission to access your Sheet.

## Security Considerations

1. **Web App Access**: Setting "Anyone" allows public access. This is necessary for public forms but means anyone with the URL can submit data.

2. **Rate Limiting**: Google Apps Script has daily quotas:
   - 20,000 requests per day (free tier)
   - 6 minutes execution time per request

3. **Input Validation**: The script validates required fields, but you can add more validation as needed.

4. **Sheet Protection**: Consider protecting certain columns or rows in your Sheet to prevent accidental edits.

## Updating the Script

If you need to update the script:
1. Make your changes in Apps Script editor
2. Click **Deploy** > **Manage deployments**
3. Click the pencil icon ✏️ next to your deployment
4. Click **New version**
5. Click **Deploy**

The URL stays the same - no need to update the form!

## Alternative: Custom Backend

If you prefer a custom backend instead of Google Apps Script, see the alternative backend examples in the original README sections below. However, Google Apps Script is recommended for simplicity and cost-effectiveness.

---

## Additional Resources

- [Google Apps Script Documentation](https://developers.google.com/apps-script)
- [Google Sheets API](https://developers.google.com/sheets/api)
- [Apps Script Web Apps Guide](https://developers.google.com/apps-script/guides/web)

## Support

For issues:
1. Check the troubleshooting section above
2. Review Apps Script execution logs
3. Test the Web App URL directly in a browser
4. Verify the form is sending the correct data (check browser console)
