# PRD-102: Sprint 2 - Calendar Module Core
*FamilyBoard MVP Sprint 2 Requirements*

## Sprint Information
- **Sprint ID:** PRD-102
- **Sprint Name:** Calendar Module Core
- **Duration:** 2 weeks (Weeks 3-4)
- **Goal:** Complete Calendar module with Google integration
- **Parent Documents:** 
  - [PRD-001](../PRD-001-FamilyBoard-Overview.md)
  - [PRD-002](../modules/PRD-002-Calendar-Module.md)
- **Dependencies:** Sprint 1 completion

## Sprint Objectives

### Primary Goals
1. Implement Google Calendar OAuth integration
2. Build calendar sync Cloud Functions
3. Create monthly calendar view UI
4. Implement day/week zoom interactions
5. Set up real-time calendar updates

### Success Criteria
- Google Calendar bi-directional sync working
- Monthly view displays all family events
- Real-time updates across devices
- Calendar module fully functional

## Technical Requirements

### 1. Google Calendar Integration

#### 1.1 OAuth Setup
```javascript
// Cloud Function: initializeGoogleAuth
export const initializeGoogleAuth = functions.https.onCall(async (data, context) => {
  const { familyId, userId } = data;
  
  // Generate OAuth URL
  const oauth2Client = new google.auth.OAuth2(
    CLIENT_ID,
    CLIENT_SECRET,
    REDIRECT_URI
  );
  
  const scopes = [
    'https://www.googleapis.com/auth/calendar',
    'https://www.googleapis.com/auth/calendar.events'
  ];
  
  const url = oauth2Client.generateAuthUrl({
    access_type: 'offline',
    scope: scopes,
    state: `${familyId}:${userId}`
  });
  
  return { authUrl: url };
});
```

#### 1.2 Token Management
```javascript
// Firestore: Calendar credentials
/families/{familyId}/calendarCredentials/{userId}
{
  googleTokens: {
    access_token: string,
    refresh_token: string,
    expiry_date: number
  },
  calendars: [{
    id: string,
    summary: string,
    primary: boolean,
    selected: boolean,
    color: string
  }],
  lastSync: timestamp
}
```

#### 1.3 Calendar Sync Function
```javascript
// Cloud Function: syncGoogleCalendars
export const syncGoogleCalendars = functions.pubsub
  .schedule('every 15 minutes')
  .onRun(async (context) => {
    // Get all families with calendar credentials
    // For each family member with credentials:
    //   1. Refresh token if needed
    //   2. Fetch calendar events
    //   3. Compare with stored events
    //   4. Update Firestore
    //   5. Trigger realtime updates
});
```

### 2. Calendar Data Management

#### 2.1 Event Schema
```javascript
/families/{familyId}/events/{eventId}
{
  // Core fields
  id: string,
  title: string,
  description: string,
  startTime: timestamp,
  endTime: timestamp,
  allDay: boolean,
  
  // Location
  location: {
    name: string,
    address: string,
    coordinates: geopoint
  },
  
  // Metadata
  category: 'school' | 'sports' | 'travel' | 'service' | 'personal',
  subcategory: string, // e.g., 'practice', 'game', 'meeting'
  
  // Assignment
  familyMembers: string[], // User IDs involved
  owner: string, // Who created/owns event
  
  // Source
  source: {
    type: 'google' | 'manual' | 'email' | 'sms',
    id: string, // Google event ID if applicable
    calendarId: string, // Source calendar
    lastSync: timestamp
  },
  
  // Visual
  color: string, // Hex color
  icon: string, // Icon identifier
  
  // Recurrence
  recurring: {
    rule: string, // RRULE format
    exceptions: timestamp[], // Cancelled instances
    parentId: string // Original event ID
  },
  
  // Status
  status: 'confirmed' | 'tentative' | 'cancelled',
  visibility: 'public' | 'private',
  
  // Timestamps
  created: timestamp,
  updated: timestamp,
  lastModifiedBy: string
}
```

#### 2.2 Event Sync Logic
- **Conflict Resolution:**
  - Google Calendar is source of truth for Google events
  - Last-write-wins for manual events
  - Preserve local changes until next sync
  
- **Sync Strategy:**
  ```javascript
  // Incremental sync using syncToken
  const syncCalendarEvents = async (calendarId, syncToken) => {
    const events = await calendar.events.list({
      calendarId,
      syncToken: syncToken || undefined,
      maxResults: 250
    });
    
    // Process changes
    for (const event of events.items) {
      if (event.status === 'cancelled') {
        await deleteEvent(event.id);
      } else {
        await upsertEvent(event);
      }
    }
    
    return events.nextSyncToken;
  };
  ```

### 3. Calendar UI Implementation

#### 3.1 Monthly View Component
```typescript
interface MonthlyCalendarProps {
  familyId: string;
  currentMonth: Date;
  events: Event[];
  onDayPress: (date: Date) => void;
  onEventPress: (event: Event) => void;
}

// Key features:
// - 6-week grid (42 days)
// - Previous/next month dates grayed
// - Today highlighted
// - Event dots colored by family member
// - Swipe navigation between months
```

#### 3.2 Day View Component
```typescript
interface DayViewProps {
  date: Date;
  events: Event[];
  onClose: () => void;
  onEventPress: (event: Event) => void;
}

// Key features:
// - Chronological event list
// - All-day events at top
// - Time-based events below
// - Family member color coding
// - Quick add button
```

#### 3.3 Calendar Navigation
- **Gestures:**
  - Horizontal swipe: Change month
  - Tap day: Open day view
  - Pinch: Show week view (optional)
  - Pull down: Refresh sync
  
- **Visual Feedback:**
  - Loading states during sync
  - Sync status indicator
  - Error states for failed syncs

### 4. Real-time Updates

#### 4.1 Realtime Database Structure
```javascript
/families/{familyId}/calendarUpdates/
{
  lastUpdate: timestamp,
  updatedEvents: {
    [eventId]: {
      action: 'created' | 'updated' | 'deleted',
      timestamp: timestamp,
      userId: string
    }
  }
}
```

#### 4.2 Update Flow
1. User makes change (add/edit/delete event)
2. Update Firestore immediately
3. Write to Realtime DB updates queue
4. Other devices receive update notification
5. Refresh affected events from Firestore

### 5. Calendar Features

#### 5.1 Event Management
- **Add Event:**
  - Quick add with minimal fields
  - Full form with all options
  - Default to current user's calendar
  
- **Edit Event:**
  - Only owner can edit Google events
  - Anyone can edit manual events
  - Conflict warnings
  
- **Delete Event:**
  - Soft delete with undo
  - Remove from Google if applicable
  - Notify affected family members

#### 5.2 Calendar Settings
```javascript
/families/{familyId}/calendarSettings
{
  defaultView: 'month' | 'week',
  weekStartsOn: 0-6, // Sunday-Saturday
  workingHours: {
    start: string, // "09:00"
    end: string    // "17:00"
  },
  categories: [{
    id: string,
    name: string,
    color: string,
    icon: string
  }],
  notifications: {
    defaultReminder: number, // minutes before
    enablePush: boolean
  }
}
```

### 6. Cloud Functions

#### 6.1 Calendar Functions
```javascript
// OAuth and sync
exports.handleGoogleAuthCallback
exports.refreshGoogleTokens
exports.syncGoogleCalendars
exports.syncSingleCalendar

// Event management
exports.createEvent
exports.updateEvent
exports.deleteEvent
exports.detectConflicts

// Notifications
exports.sendEventReminders
exports.notifyEventChanges
```

#### 6.2 Scheduled Jobs
- **Every 15 minutes:** Sync all Google calendars
- **Every hour:** Clean up old sync tokens
- **Daily:** Generate next day's reminders
- **Weekly:** Calendar analytics summary

## Deliverables

### 1. Google Calendar Integration
- [ ] OAuth flow implementation
- [ ] Token storage and refresh
- [ ] Calendar list retrieval
- [ ] Event sync (both directions)
- [ ] Incremental sync with tokens

### 2. Calendar UI
- [ ] Monthly view grid
- [ ] Event dots and previews
- [ ] Day view with all events
- [ ] Smooth navigation
- [ ] Loading and error states

### 3. Event Management
- [ ] Create events (manual)
- [ ] Edit events (with permissions)
- [ ] Delete events
- [ ] Recurring event support
- [ ] Conflict detection

### 4. Real-time Features
- [ ] Live event updates
- [ ] Sync status indicator
- [ ] Pull-to-refresh
- [ ] Offline event queue

### 5. Calendar Settings
- [ ] Calendar selection
- [ ] Color customization
- [ ] Category management
- [ ] Notification preferences

## Testing Requirements

### 1. Integration Tests
- Google OAuth flow
- Calendar sync accuracy
- Bi-directional sync
- Token refresh
- Error handling

### 2. UI Tests
- Monthly view rendering
- Navigation gestures
- Event interactions
- Responsive layouts
- Performance with many events

### 3. E2E Tests
- Complete OAuth setup
- Add event flow
- Sync verification
- Multi-device updates
- Offline/online transitions

## Performance Requirements

### 1. Load Times
- Monthly view: < 1 second
- Day view: < 500ms
- Event creation: < 2 seconds
- Sync completion: < 5 seconds

### 2. Sync Efficiency
- Incremental sync only
- Batch API requests
- Cache calendar data
- Minimize bandwidth

### 3. UI Performance
- 60 FPS scrolling
- Smooth transitions
- Responsive to touch
- Memory efficient

## Definition of Done

### Functionality
- [ ] All deliverables complete
- [ ] Google sync working reliably
- [ ] Real-time updates verified
- [ ] Offline mode functional

### Quality
- [ ] Unit test coverage > 80%
- [ ] Integration tests passing
- [ ] UI tests automated
- [ ] Performance targets met

### Documentation
- [ ] API documentation
- [ ] User guide for calendar
- [ ] Troubleshooting guide
- [ ] Sync architecture documented

## Risks & Mitigations

### 1. Google API Limits
- **Risk:** Rate limiting on API calls
- **Mitigation:** Implement exponential backoff, batch requests

### 2. Sync Conflicts
- **Risk:** Data conflicts between local and Google
- **Mitigation:** Clear conflict resolution rules, user notifications

### 3. Performance with Many Events
- **Risk:** Slow rendering with hundreds of events
- **Mitigation:** Virtualization, pagination, smart loading

## Dependencies
- Google Calendar API access
- OAuth client credentials
- Sprint 1 authentication system
- Firebase Functions deployment

## Next Sprint Preview
[Sprint 3: Chores Module Core](PRD-103-Sprint-3.md) - Task management system implementation