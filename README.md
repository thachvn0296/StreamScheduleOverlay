# Baguetty Stream Schedule Overlay

## Features

- Current week + next week, Monday → Sunday
- Day name and date on the same top line
- Stream card anchored at the bottom of each day
- Hover a stream to see time, title, description and Twitch link
- Theme choice saved in the browser
- Google Calendar sync through Google Apps Script
- No Google password/API key stored in the overlay
- Designed for OBS Browser Source / GitHub Pages

## 1. Google Calendar connection

The recommended architecture is:

Google Calendar
      ↓
Google Apps Script Web App
      ↓
GitHub Pages overlay
      ↓
OBS Browser Source

The Apps Script reads ONE specific calendar by its Calendar ID and returns only the events needed by the overlay.

### Find the Calendar ID

1. Open Google Calendar.
2. On the left under "My calendars", find the calendar you want to use.
3. Click the three dots next to that calendar.
4. Choose "Settings and sharing".
5. Scroll to "Integrate calendar".
6. Copy the "Calendar ID".

It can look like:
yourname@gmail.com

or:
abcdef123456@group.calendar.google.com

### Create the Apps Script

1. Go to https://script.google.com/
2. Create a new project.
3. Delete the default code.
4. Paste the following:

```javascript
const CALENDAR_ID = "PASTE_YOUR_CALENDAR_ID_HERE";

function doGet(e) {
  const callback = e && e.parameter && e.parameter.callback;

  const calendar = CalendarApp.getCalendarById(CALENDAR_ID);

  if (!calendar) {
    return output(callback, {error: "Calendar not found"});
  }

  const now = new Date();
  const monday = getMonday(now);

  // Two weeks: this Monday through next Sunday.
  const end = new Date(monday);
  end.setDate(end.getDate() + 14);

  const events = calendar.getEvents(monday, end).map(event => ({
    start: event.getStartTime().toISOString(),
    end: event.getEndTime().toISOString(),
    title: event.getTitle(),
    description: event.getDescription() || "",
    url: getTwitchUrl(event.getDescription())
  }));

  return output(callback, events);
}

function getMonday(date) {
  const d = new Date(date);
  const day = d.getDay();
  const diff = day === 0 ? -6 : 1 - day;
  d.setHours(0, 0, 0, 0);
  d.setDate(d.getDate() + diff);
  return d;
}

function getTwitchUrl(description) {
  const text = description || "";
  const match = text.match(/https?:\/\/(?:www\.)?twitch\.tv\/[A-Za-z0-9_]+/i);
  return match ? match[0] : "https://www.twitch.tv/baguetty";
}

function output(callback, data) {
  if (!callback) {
    return ContentService
      .createTextOutput(JSON.stringify(data))
      .setMimeType(ContentService.MimeType.JSON);
  }
  return ContentService
    .createTextOutput(callback + "(" + JSON.stringify(data) + ")")
    .setMimeType(ContentService.MimeType.JAVASCRIPT);
}
```

5. Replace:
   `PASTE_YOUR_CALENDAR_ID_HERE`
   with the Calendar ID you copied.

6. Click Save.

### How to put the Twitch link into a calendar event

For maximum flexibility, put the Twitch URL in the event description.

Example event:

Title:
`Once Human`

Description:
`Boss farming stream
https://www.twitch.tv/baguetty`

The script will automatically find the Twitch URL.

If there is no Twitch URL in the description, it uses the fallback URL configured in `index.html`.

## 2. Deploy the Google Apps Script

In Apps Script:

1. Click "Deploy".
2. Click "New deployment".
3. For "Select type", choose "Web app".
4. Set "Execute as" to yourself / Me.
5. Set "Who has access" to anyone who should be able to access the feed.
6. Click "Deploy".
7. Authorize the script when Google asks.
8. Copy the Web app URL ending in `/exec`.

Google's official web-app deployment documentation:
https://developers.google.com/apps-script/guides/web

## 3. Connect the overlay

Open `index.html`.

Find:

CALENDAR_API_URL: "",

Change it to:

CALENDAR_API_URL: "YOUR_APPS_SCRIPT_EXEC_URL",

Then change:

DEMO_MODE: true,

to:

DEMO_MODE: false,

Save the file.

## 4. Test locally

Open `index.html` in a browser.

If your Apps Script URL is correct, your real calendar events should appear.

If you change the Google Calendar, the overlay checks for new data every 5 minutes.

## 5. Publish on GitHub Pages

1. Sign in to GitHub.
2. Create a new repository.
3. A simple name is:
   `stream-schedule`
4. Upload `index.html`.
5. Commit the file.
6. Go to the repository's Settings.
7. Select Pages.
8. Under Build and deployment, choose:
   Source: Deploy from a branch
9. Select:
   Branch: `main`
   Folder: `/ (root)`
10. Save.

GitHub will provide a URL similar to:

https://YOUR-USERNAME.github.io/stream-schedule/

GitHub says Pages changes can take several minutes to become available.

## 6. Add it to OBS

1. Open OBS.
2. Add a new "Browser" source.
3. Name it:
   `Stream Schedule`
4. Enter your GitHub Pages URL.
5. Suggested size:
   Width: 1200
   Height: 520
6. Click OK.

The overlay will load directly from GitHub Pages.

### Important OBS setting

If you want the schedule to be visible over your game/camera, the page background is transparent outside the schedule card.

## Security note

Do NOT put your Google password, OAuth refresh token, Google API private key, or other credentials in `index.html`.

The Apps Script acts as the private bridge to the selected calendar. Only the event information returned by the script is exposed to the public overlay.

Also remember that GitHub Pages sites are publicly accessible. Do not put private calendar credentials or sensitive information in the repository.

## Future improvements
- Multiple streams on the same day?
- LIVE NOW indicator
