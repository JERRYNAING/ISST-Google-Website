# Timer Component - Documentation

## Overview
This timer component displays a countdown timer for submission deadlines. It features a modern UI with animated status indicators, progress bars, and action buttons for downloading and submitting forms.

## How to Change the Timer Duration

### Quick Guide
To change the timer duration, you only need to modify **one line** in the JavaScript code.

### Step-by-Step Instructions

1. **Open the file**: `Finance/Timer.html`

2. **Locate the TimerManager class constructor** (around line 637-647)

3. **Find this line**:
   ```javascript
   this.COUNTDOWN_DURATION_HOURS = 24;
   ```

4. **Change the number** to your desired duration in hours:
   ```javascript
   // Examples:
   this.COUNTDOWN_DURATION_HOURS = 12;  // 12 hours
   this.COUNTDOWN_DURATION_HOURS = 6;   // 6 hours
   this.COUNTDOWN_DURATION_HOURS = 1;   // 1 hour
   this.COUNTDOWN_DURATION_HOURS = 48;  // 48 hours (2 days)
   ```

5. **Update the initial display** (optional but recommended):
   
   Also update the initial HTML display to match your timer duration. Find this section (around line 19-23):
   ```html
   <span class="time-segment hours">24</span>
   <span class="time-separator">:</span>
   <span class="time-segment minutes">00</span>
   <span class="time-separator">:</span>
   <span class="time-segment seconds">00</span>
   ```
   
   Change the hours value to match your `COUNTDOWN_DURATION_HOURS`:
   ```html
   <!-- For 12 hours -->
   <span class="time-segment hours">12</span>
   
   <!-- For 6 hours -->
   <span class="time-segment hours">06</span>
   ```

6. **Save the file** and refresh your browser to see the changes.

### Example: Changing to 12 Hours

**Before:**
```javascript
this.COUNTDOWN_DURATION_HOURS = 24;
```

**After:**
```javascript
this.COUNTDOWN_DURATION_HOURS = 12;
```

And in the HTML:
```html
<span class="time-segment hours">12</span>
```

## Code Structure

### Key Components

1. **HTML Structure** (Lines 4-111)
   - Timer display section (left side)
   - Action buttons section (right side)
   - Status indicator

2. **CSS Styling** (Lines 113-633)
   - CSS variables for colors and design tokens
   - Component styles
   - Animations and keyframes
   - Responsive design media queries

3. **JavaScript TimerManager Class** (Lines 635-814)
   - `constructor()`: Initializes timer with duration
   - `updateDisplay()`: Updates the visual timer display
   - `tick()`: Called every second to decrement timer
   - `timeUp()`: Handles timer expiration
   - `checkWarnings()`: Shows warnings when time is low

### Important Code Sections

#### Timer Duration Configuration
```javascript
// Location: TimerManager constructor (line ~638)
this.COUNTDOWN_DURATION_HOURS = 24;  // ← CHANGE THIS VALUE
this.totalSeconds = this.COUNTDOWN_DURATION_HOURS * 60 * 60;
```

#### Display Update Logic
```javascript
// Location: updateDisplay() method (line ~691)
const hours = Math.floor(this.remainingSeconds / 3600);
const minutes = Math.floor((this.remainingSeconds % 3600) / 60);
const seconds = this.remainingSeconds % 60;
```

#### Warning Thresholds
```javascript
// Location: checkWarnings() method (line ~715)
// These thresholds are based on hours remaining
// Adjust if needed for different timer durations
const hoursLeft = this.remainingSeconds / 3600;
if (hoursLeft <= 1 && hoursLeft > 0.5) {
  // Warning at 1 hour
}
```

## Features

- **Countdown Timer**: Displays hours, minutes, and seconds
- **Progress Bar**: Visual representation of time remaining
- **Status Indicator**: Shows "Active" (green) or "Expired" (red)
- **Warning System**: Visual warnings when time is running low
- **Button Locking**: Disables buttons when timer expires
- **Responsive Design**: Works on mobile, tablet, and desktop
- **Animations**: Smooth transitions and visual effects

## Customization Options

### Change Warning Thresholds
Edit the `checkWarnings()` method to adjust when warnings appear:
```javascript
// Current: Warns at 1 hour, 30 minutes, and 6 minutes
// You can customize these values
```

### Change Button Links
Update the `href` attributes in the HTML:
```html
<!-- Download Button (line ~66) -->
<a id="downloadBtn" href="YOUR_GOOGLE_SHEETS_URL" ...>

<!-- Submit Button (line ~88) -->
<a id="submitBtn" href="YOUR_GOOGLE_SHEETS_URL" ...>
```

### Change Colors
Modify CSS variables in the `:root` section (line ~114):
```css
--primary: #6366f1;      /* Main timer color */
--success: #10b981;     /* Active status color */
--danger: #ef4444;      /* Expired status color */
```

## Troubleshooting

### Timer Not Starting
- Check browser console for JavaScript errors
- Ensure all required HTML elements exist (timer, progressFill, progressPercent)
- Verify the TimerManager class is being instantiated

### Timer Shows Wrong Time
- Verify `COUNTDOWN_DURATION_HOURS` is set correctly
- Check that `totalSeconds` calculation is correct (hours × 60 × 60)
- Ensure initial HTML display matches the duration

### Buttons Not Working
- Check that button links are valid URLs
- Verify buttons aren't disabled (check for `disabled` class)
- Ensure JavaScript event listeners are bound correctly

## Browser Compatibility

- Chrome/Edge: ✅ Fully supported
- Firefox: ✅ Fully supported
- Safari: ✅ Fully supported
- Mobile browsers: ✅ Responsive design included

## Notes

- The timer runs client-side and will reset if the page is refreshed
- For persistent timers across sessions, consider adding localStorage or server-side tracking
- The timer uses `setInterval` which may drift slightly over long periods
- All time calculations are in seconds internally, converted to hours/minutes/seconds for display

## Support

For issues or questions, refer to the code comments in `Timer.html` which explain each section in detail.

