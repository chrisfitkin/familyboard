# PRD-105: Sprint 5 - External Data Integration
*FamilyBoard MVP Sprint 5 Requirements*

## Sprint Information
- **Sprint ID:** PRD-105
- **Sprint Name:** External Data Integration
- **Duration:** 2 weeks (Weeks 9-10)
- **Goal:** Add email/SMS parsing for Calendar and create Meals module placeholder
- **Parent Document:** [PRD-001](../PRD-001-FamilyBoard-Overview.md)
- **Dependencies:** Sprints 1-4 (Infrastructure, Calendar, Chores, Profiles)

## Sprint Objectives

### Primary Goals
1. Implement Gmail integration for event parsing
2. Build email parsing with ML/AI
3. Create SMS parsing capability
4. Add manual event entry flows
5. Create "Coming Soon" Meals module tab

### Success Criteria
- Coach emails automatically create calendar events
- Service appointment texts parsed correctly
- Manual entry provides quick event creation
- Meals tab displays coming soon message

## Technical Requirements

### 1. Gmail Integration

#### 1.1 Gmail API Setup
```javascript
// Cloud Function: Setup Gmail watch
exports.setupGmailWatch = functions.https.onCall(async (data, context) => {
  const { userId } = context.auth;
  const { familyId } = data;
  
  // Get user's Gmail credentials
  const oauth2Client = await getOAuth2Client(userId);
  
  const gmail = google.gmail({ version: 'v1', auth: oauth2Client });
  
  // Set up push notifications for new emails
  const watchResponse = await gmail.users.watch({
    userId: 'me',
    requestBody: {
      labelIds: ['INBOX'],
      topicName: `projects/${PROJECT_ID}/topics/gmail-push`
    }
  });
  
  // Store watch details
  await firestore.collection(`families/${familyId}/emailWatches`).doc(userId).set({
    historyId: watchResponse.data.historyId,
    expiration: watchResponse.data.expiration,
    created: serverTimestamp()
  });
  
  return { success: true };
});
```

#### 1.2 Email Processing Pipeline
```javascript
// Cloud Function: Process Gmail push notification
exports.processGmailPush = functions.pubsub
  .topic('gmail-push')
  .onPublish(async (message) => {
    const data = JSON.parse(Buffer.from(message.data, 'base64').toString());
    const { emailAddress, historyId } = data;
    
    // Get user and family from email
    const user = await getUserByEmail(emailAddress);
    const familyId = user.familyId;
    
    // Get new messages since last historyId
    const messages = await getNewMessages(user.id, historyId);
    
    // Process each message
    for (const message of messages) {
      await processEmailForEvents(message, familyId, user.id);
    }
    
    // Update historyId
    await updateHistoryId(user.id, historyId);
  });
```

#### 1.3 Email Watch Schema
```javascript
/families/{familyId}/emailWatches/{userId}
{
  email: string,
  historyId: string,
  expiration: timestamp,
  
  // Filtering rules
  filters: {
    senders: [{
      email: string,
      name: string,
      type: 'coach' | 'school' | 'service' | 'other',
      autoProcess: boolean
    }],
    
    keywords: string[], // "practice", "game", "appointment"
    
    labels: string[] // Gmail labels to watch
  },
  
  // Stats
  stats: {
    totalProcessed: number,
    eventsCreated: number,
    lastProcessed: timestamp
  },
  
  created: timestamp,
  lastModified: timestamp
}
```

### 2. Email Parsing System

#### 2.1 ML-Based Event Extraction
```javascript
// Cloud Function: Parse email for events
exports.parseEmailForEvents = functions.https.onCall(async (data, context) => {
  const { emailContent, familyId, userId } = data;
  
  // Step 1: Clean and prepare email text
  const cleanedText = cleanEmailContent(emailContent);
  
  // Step 2: Use ML to extract potential events
  const extractedData = await mlExtractEvents(cleanedText);
  
  // Step 3: Validate and enhance extracted data
  const events = await processExtractedEvents(extractedData, {
    timezone: await getFamilyTimezone(familyId),
    familyMembers: await getFamilyMembers(familyId)
  });
  
  // Step 4: Return parsed events for user confirmation
  return {
    events,
    confidence: calculateConfidence(events),
    source: {
      type: 'email',
      preview: cleanedText.substring(0, 200)
    }
  };
});

// ML extraction using Firebase ML or external service
const mlExtractEvents = async (text) => {
  // Option 1: Firebase ML Kit
  const language = await mlKit.naturalLanguage().processText(text);
  const entities = language.entities.filter(e => 
    e.type === 'DATE_TIME' || e.type === 'LOCATION'
  );
  
  // Option 2: Custom model for sports/school events
  const customExtraction = await callCustomModel(text);
  
  return mergeExtractions(entities, customExtraction);
};
```

#### 2.2 Event Pattern Recognition
```javascript
// Pattern definitions for common event types
const eventPatterns = {
  sportsEvent: {
    patterns: [
      /(?:game|match|meet|tournament)\s+(?:on|at)?\s*(\w+day)?,?\s*(\w+\s+\d{1,2})/i,
      /(?:practice|training)\s+(?:every)?\s*(\w+day)s?\s+(?:at|from)?\s*(\d{1,2}:\d{2})/i
    ],
    
    extractor: (match, text) => ({
      type: 'sports',
      title: extractSportType(text) + ' ' + (match[0].includes('practice') ? 'Practice' : 'Game'),
      date: parseFlexibleDate(match[1], match[2]),
      recurring: match[0].includes('every')
    })
  },
  
  schoolEvent: {
    patterns: [
      /(?:parent.teacher|back.to.school|conference)\s+(?:on)?\s*(\w+\s+\d{1,2})/i,
      /(?:field trip|assembly|program)\s+(?:scheduled for)?\s*(\w+,?\s+\w+\s+\d{1,2})/i
    ],
    
    extractor: (match, text) => ({
      type: 'school',
      title: extractEventTitle(match[0]),
      date: parseSchoolDate(match[1]),
      requiresResponse: text.includes('RSVP') || text.includes('confirm')
    })
  },
  
  appointment: {
    patterns: [
      /appointment\s+(?:scheduled|confirmed)?\s+(?:for|on)?\s*(\w+,?\s+\w+\s+\d{1,2})\s+at\s+(\d{1,2}:\d{2})/i,
      /(?:cleaning|service|maintenance)\s+(?:scheduled)?\s*(\w+\s+\d{1,2})\s+(?:between)?\s*(\d{1,2}(?::\d{2})?)/i
    ],
    
    extractor: (match, text) => ({
      type: 'service',
      title: extractServiceType(text) + ' Appointment',
      date: parseAppointmentDate(match[1]),
      time: parseTime(match[2]),
      location: extractLocation(text)
    })
  }
};
```

#### 2.3 Email Source Management
```javascript
/families/{familyId}/emailSources/{sourceId}
{
  email: string,
  name: string,
  type: 'coach' | 'school' | 'service' | 'other',
  
  // Learning data
  patterns: {
    commonPhrases: string[],
    dateFormats: string[],
    timeFormats: string[],
    locationFormats: string[]
  },
  
  // Auto-processing rules
  autoProcess: {
    enabled: boolean,
    confidence: number, // Required confidence level
    requiresApproval: boolean,
    
    // Mapping rules
    defaultCategory: string,
    defaultFamilyMembers: string[],
    defaultReminders: number[]
  },
  
  // Stats
  stats: {
    totalEmails: number,
    eventsCreated: number,
    accuracy: number, // User-confirmed accuracy
    lastProcessed: timestamp
  },
  
  created: timestamp,
  createdBy: string
}
```

### 3. SMS Integration

#### 3.1 SMS Processing
```javascript
// Cloud Function: Process SMS forward
exports.processSmsForward = functions.https.onCall(async (data, context) => {
  const { smsText, fromNumber, familyId, userId } = data;
  
  // Identify sender type
  const senderInfo = await identifySender(fromNumber);
  
  // Extract event information
  const extractedEvent = await extractSmsEvent(smsText, senderInfo);
  
  if (extractedEvent) {
    // Create event preview for confirmation
    return {
      success: true,
      event: {
        ...extractedEvent,
        source: {
          type: 'sms',
          number: fromNumber,
          name: senderInfo?.name
        }
      },
      requiresConfirmation: true
    };
  }
  
  return { success: false, reason: 'No event detected' };
});

// SMS-specific patterns (typically shorter and more structured)
const smsPatterns = {
  appointment: {
    pattern: /appointment\s+(\w+)\s+(\d{1,2}\/\d{1,2})\s+(?:at\s+)?(\d{1,2}:\d{2})/i,
    parser: (match) => ({
      title: 'Appointment',
      date: parseSmsDate(match[2]),
      time: match[3]
    })
  },
  
  reminder: {
    pattern: /reminder:?\s+(.+)\s+(?:on|tomorrow|today)\s+(?:at\s+)?(\d{1,2}:\d{2})/i,
    parser: (match) => ({
      title: match[1],
      date: parseRelativeDate(match[0]),
      time: match[2]
    })
  }
};
```

#### 3.2 SMS Source Tracking
```javascript
/families/{familyId}/smsSources/{phoneNumber}
{
  number: string,
  name: string,
  type: 'medical' | 'school' | 'service' | 'other',
  
  // Known formats from this sender
  formats: [{
    pattern: string,
    example: string,
    lastSeen: timestamp
  }],
  
  // Processing rules
  autoProcess: boolean,
  defaultCategory: string,
  
  stats: {
    totalMessages: number,
    eventsCreated: number
  },
  
  created: timestamp
}
```

### 4. Manual Event Entry

#### 4.1 Quick Add Interface
```typescript
interface QuickAddEvent {
  // Natural language input
  naturalInput: string;
  
  // Parsed results
  parsed: {
    title: string;
    date?: Date;
    time?: string;
    duration?: number;
    location?: string;
    participants?: string[];
  };
  
  // Suggestions
  suggestions: {
    dates: Date[];
    times: string[];
    participants: FamilyMember[];
    categories: EventCategory[];
  };
}

// Quick add component
const QuickAddComponent = () => {
  const [input, setInput] = useState('');
  const [parsed, setParsed] = useState<ParsedEvent>(null);
  
  const handleInputChange = debounce(async (text: string) => {
    const result = await parseNaturalLanguage(text);
    setParsed(result);
  }, 500);
  
  // Examples shown to user:
  // "Soccer practice every Tuesday at 4pm"
  // "Dentist appointment March 15 at 2:30"
  // "School play on Friday at 7pm"
};
```

#### 4.2 Natural Language Processing
```javascript
// Local natural language parsing
const parseNaturalLanguage = (input) => {
  const lower = input.toLowerCase();
  
  // Extract time
  const timeMatch = lower.match(/at\s+(\d{1,2}:?\d{0,2}\s*(?:am|pm)?)/);
  const time = timeMatch ? parseTimeString(timeMatch[1]) : null;
  
  // Extract date
  const datePatterns = [
    /(?:on|this|next)?\s*(monday|tuesday|wednesday|thursday|friday|saturday|sunday)/i,
    /(?:on)?\s*(\d{1,2}\/\d{1,2}(?:\/\d{2,4})?)/,
    /(?:on)?\s*(jan|feb|mar|apr|may|jun|jul|aug|sep|oct|nov|dec)\w*\s+(\d{1,2})/i,
    /(today|tomorrow|next week)/i
  ];
  
  let date = null;
  for (const pattern of datePatterns) {
    const match = lower.match(pattern);
    if (match) {
      date = parseDateFromMatch(match);
      break;
    }
  }
  
  // Extract recurring
  const recurringMatch = lower.match(/every\s+(day|week|month|\w+day)/i);
  const recurring = recurringMatch ? parseRecurring(recurringMatch[1]) : null;
  
  // Extract title (remove date/time parts)
  let title = input;
  [timeMatch, dateMatch, recurringMatch].forEach(match => {
    if (match) title = title.replace(match[0], '');
  });
  title = title.trim();
  
  return { title, date, time, recurring };
};
```

### 5. Meals Module Placeholder

#### 5.1 Coming Soon Tab
```typescript
interface MealsPlaceholderProps {
  onNotifyMe: () => void;
}

const MealsPlaceholder: React.FC<MealsPlaceholderProps> = ({ onNotifyMe }) => {
  return (
    <View style={styles.container}>
      <View style={styles.content}>
        {/* Animated icon */}
        <LottieView
          source={require('./animations/meal-planning.json')}
          autoPlay
          loop
          style={styles.animation}
        />
        
        <Text style={styles.title}>Meals Module</Text>
        <Text style={styles.subtitle}>Coming Soon!</Text>
        
        <Text style={styles.description}>
          Plan weekly meals, get AI-powered suggestions, 
          and generate shopping lists - all based on your 
          family's preferences.
        </Text>
        
        {/* Feature preview */}
        <View style={styles.features}>
          <FeatureItem icon="🍽️" text="Weekly meal planning" />
          <FeatureItem icon="🤖" text="AI recommendations" />
          <FeatureItem icon="⭐" text="Family ratings" />
          <FeatureItem icon="📝" text="Shopping lists" />
          <FeatureItem icon="🥗" text="Nutrition tracking" />
        </View>
        
        {/* Call to action */}
        <TouchableOpacity 
          style={styles.notifyButton}
          onPress={onNotifyMe}
        >
          <Text style={styles.buttonText}>
            Notify me when available
          </Text>
        </TouchableOpacity>
        
        <Text style={styles.eta}>Expected: Q3 2024</Text>
      </View>
    </View>
  );
};
```

#### 5.2 Email Collection
```javascript
// Store interest for meals module
/families/{familyId}/moduleInterest/meals
{
  interestedUsers: [{
    userId: string,
    email: string,
    signupDate: timestamp
  }],
  
  // Feature votes
  requestedFeatures: {
    [feature: string]: number // vote count
  },
  
  // Feedback
  feedback: [{
    userId: string,
    message: string,
    date: timestamp
  }]
}
```

### 6. Event Confirmation UI

#### 6.1 Confirmation Dialog
```typescript
interface EventConfirmationProps {
  suggestedEvent: ParsedEvent;
  source: EventSource;
  onConfirm: (event: Event) => void;
  onEdit: (event: Event) => void;
  onReject: () => void;
}

const EventConfirmation: React.FC<EventConfirmationProps> = ({
  suggestedEvent,
  source,
  onConfirm,
  onEdit,
  onReject
}) => {
  return (
    <Modal>
      <View style={styles.header}>
        <Text>New Event Detected</Text>
        <Text style={styles.source}>
          From: {source.name || source.identifier}
        </Text>
      </View>
      
      {/* Event preview */}
      <EventPreview event={suggestedEvent} />
      
      {/* Editable fields */}
      <View style={styles.fields}>
        <EditableField 
          label="Title" 
          value={suggestedEvent.title}
          onChange={(value) => updateField('title', value)}
        />
        <EditableField 
          label="Date" 
          value={suggestedEvent.date}
          type="date"
        />
        <EditableField 
          label="Time" 
          value={suggestedEvent.time}
          type="time"
        />
        <FamilyMemberPicker
          selected={suggestedEvent.familyMembers}
          onChange={(members) => updateField('familyMembers', members)}
        />
      </View>
      
      {/* Actions */}
      <View style={styles.actions}>
        <Button title="Add to Calendar" onPress={() => onConfirm(event)} />
        <Button title="Edit More" onPress={() => onEdit(event)} />
        <Button title="Ignore" onPress={onReject} />
      </View>
      
      {/* Learning option */}
      <CheckBox
        label="Always auto-add from this sender"
        checked={autoAdd}
        onChange={setAutoAdd}
      />
    </Modal>
  );
};
```

## Deliverables

### 1. Gmail Integration
- [ ] OAuth setup for Gmail
- [ ] Push notification subscription
- [ ] Email fetching and processing
- [ ] Sender management UI
- [ ] Filtering rules configuration

### 2. Email Parsing
- [ ] ML event extraction
- [ ] Pattern recognition system
- [ ] Multi-format date parsing
- [ ] Location extraction
- [ ] Participant detection

### 3. SMS Processing
- [ ] SMS forward handler
- [ ] Pattern matching for SMS
- [ ] Sender identification
- [ ] Confirmation workflow

### 4. Manual Entry
- [ ] Natural language input
- [ ] Smart parsing
- [ ] Suggestion system
- [ ] Quick add UI
- [ ] Recurring event support

### 5. Meals Placeholder
- [ ] Coming soon design
- [ ] Feature preview
- [ ] Email collection
- [ ] Interest tracking
- [ ] Animation assets

## Testing Requirements

### 1. Parsing Accuracy
- Various email formats
- Different date/time formats
- Multiple languages
- Edge cases
- Performance testing

### 2. Integration Tests
- Gmail API limits
- OAuth token refresh
- Push notification delivery
- Error handling

### 3. UI/UX Tests
- Confirmation flow
- Quick add usability
- Parser feedback
- Error states

## Performance Requirements

### 1. Processing Speed
- Email parsing: < 3 seconds
- SMS parsing: < 1 second
- Natural language: < 500ms
- Confirmation UI: Instant

### 2. Accuracy Targets
- Date extraction: > 90%
- Time extraction: > 85%
- Event type: > 80%
- Overall useful: > 70%

## Definition of Done

### Functionality
- [ ] Email parsing working
- [ ] SMS parsing functional
- [ ] Manual entry smooth
- [ ] Meals tab implemented
- [ ] Confirmation flow complete

### Quality
- [ ] Parsing accuracy verified
- [ ] Error handling robust
- [ ] Performance targets met
- [ ] Security reviewed

### Documentation
- [ ] Parsing patterns documented
- [ ] API integration guide
- [ ] User help content
- [ ] Privacy policy updated

## Risks & Mitigations

### 1. Gmail API Limits
- **Risk:** Rate limiting affects functionality
- **Mitigation:** Implement caching, batch processing

### 2. Parsing Accuracy
- **Risk:** Poor extraction frustrates users
- **Mitigation:** Confirmation step, continuous learning

### 3. Privacy Concerns
- **Risk:** Email content privacy
- **Mitigation:** Clear consent, local processing option

## Dependencies
- Gmail API access approval
- ML/NLP service selection
- Calendar module complete
- Push notification infrastructure

## Next Sprint Preview
[Sprint 6: Polish & MVP Launch](PRD-106-Sprint-6.md) - Final polish and deployment preparation