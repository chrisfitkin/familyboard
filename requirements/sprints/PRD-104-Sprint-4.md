# PRD-104: Sprint 4 - Family Member Management
*FamilyBoard MVP Sprint 4 Requirements*

## Sprint Information
- **Sprint ID:** PRD-104
- **Sprint Name:** Family Member Management
- **Duration:** 2 weeks (Weeks 7-8)
- **Goal:** Complete profiles and personalization across all modules
- **Parent Document:** [PRD-001](../PRD-001-FamilyBoard-Overview.md)
- **Dependencies:** Sprints 1-3 (Auth, Calendar, Chores)

## Sprint Objectives

### Primary Goals
1. Enhance family member profiles with full personalization
2. Implement role-based permissions across modules
3. Apply consistent color coding throughout app
4. Build presence and activity tracking
5. Create family settings management

### Success Criteria
- Each family member has distinct visual identity
- Permissions properly restrict module actions
- Activity tracking shows who did what
- Family settings centrally managed

## Technical Requirements

### 1. Enhanced Profile System

#### 1.1 Extended User Profile Schema
```javascript
/families/{familyId}/members/{userId}
{
  // Basic Info (from Sprint 1)
  userId: string,
  email: string,
  displayName: string,
  
  // Enhanced Profile
  profile: {
    firstName: string,
    lastName: string,
    nickname: string, // Display preference
    dateOfBirth: timestamp,
    age: number, // Calculated field
    
    // Visual Identity
    color: string, // Hex color for UI
    avatar: {
      type: 'emoji' | 'initial' | 'photo' | 'character',
      value: string, // Emoji, URL, or character ID
      backgroundColor: string
    },
    
    // Preferences
    preferences: {
      theme: 'light' | 'dark' | 'auto',
      notifications: {
        calendar: boolean,
        chores: boolean,
        meals: boolean,
        reminderTime: string // "HH:MM"
      },
      privacy: {
        hideAge: boolean,
        hideEmail: boolean
      }
    }
  },
  
  // Role & Permissions
  role: 'admin' | 'parent' | 'teen' | 'child',
  permissions: {
    calendar: {
      view: boolean,
      create: boolean,
      edit: 'own' | 'all' | 'none',
      delete: 'own' | 'all' | 'none',
      settings: boolean
    },
    chores: {
      view: boolean,
      create: boolean,
      assign: boolean,
      complete: 'own' | 'all',
      editStreaks: boolean,
      settings: boolean
    },
    meals: {
      view: boolean,
      plan: boolean,
      rate: boolean,
      settings: boolean
    },
    family: {
      invite: boolean,
      remove: boolean,
      editRoles: boolean,
      billing: boolean
    }
  },
  
  // Activity
  activity: {
    lastActive: timestamp,
    lastDevice: string,
    currentDevice: string,
    status: 'online' | 'away' | 'offline'
  },
  
  // Metadata
  joinedAt: timestamp,
  invitedBy: string,
  modifiedAt: timestamp,
  modifiedBy: string
}
```

#### 1.2 Avatar System
```javascript
// Avatar types and configuration
const avatarTypes = {
  emoji: {
    options: ['😀', '😎', '🤖', '🦄', '🐶', '🐱', '🦊', '🐻'],
    customizable: true
  },
  
  initial: {
    generator: (firstName, lastName) => {
      return `${firstName[0]}${lastName[0]}`.toUpperCase();
    }
  },
  
  character: {
    sets: ['animals', 'robots', 'monsters'],
    options: 50, // Per set
    unlockable: true
  },
  
  photo: {
    maxSize: 5 * 1024 * 1024, // 5MB
    formats: ['jpg', 'png', 'webp'],
    processing: 'crop-square'
  }
};
```

### 2. Role-Based Access Control (RBAC)

#### 2.1 Role Definitions
```javascript
const roleDefinitions = {
  admin: {
    displayName: "Family Admin",
    description: "Full access to all features",
    ageRequirement: 18,
    maxPerFamily: 2,
    inheritsFrom: null,
    defaultPermissions: {
      // All permissions set to maximum
    }
  },
  
  parent: {
    displayName: "Parent",
    description: "Manage family activities",
    ageRequirement: 18,
    inheritsFrom: null,
    defaultPermissions: {
      calendar: { view: true, create: true, edit: 'all', delete: 'all' },
      chores: { view: true, create: true, assign: true, complete: 'all' },
      meals: { view: true, plan: true, rate: true },
      family: { invite: true, remove: false, editRoles: false }
    }
  },
  
  teen: {
    displayName: "Teen",
    description: "Participate and manage own activities",
    ageRequirement: 13,
    inheritsFrom: 'child',
    defaultPermissions: {
      calendar: { view: true, create: true, edit: 'own', delete: 'own' },
      chores: { view: true, complete: 'own' },
      meals: { view: true, rate: true }
    }
  },
  
  child: {
    displayName: "Child",
    description: "View and complete assigned tasks",
    ageRequirement: 0,
    inheritsFrom: null,
    defaultPermissions: {
      calendar: { view: true },
      chores: { view: true, complete: 'own' },
      meals: { view: true }
    }
  }
};
```

#### 2.2 Permission Checking
```javascript
// Firebase Security Rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Helper functions
    function isSignedIn() {
      return request.auth != null;
    }
    
    function getUserRole(familyId) {
      return get(/databases/$(database)/documents/families/$(familyId)/members/$(request.auth.uid)).data.role;
    }
    
    function hasPermission(familyId, module, action) {
      let member = get(/databases/$(database)/documents/families/$(familyId)/members/$(request.auth.uid)).data;
      return member.permissions[module][action] == true ||
             member.permissions[module][action] == 'all';
    }
    
    // Calendar events
    match /families/{familyId}/events/{eventId} {
      allow read: if isSignedIn() && 
                     hasPermission(familyId, 'calendar', 'view');
                     
      allow create: if isSignedIn() && 
                       hasPermission(familyId, 'calendar', 'create');
                       
      allow update: if isSignedIn() && 
                       (hasPermission(familyId, 'calendar', 'edit') ||
                        (resource.data.owner == request.auth.uid && 
                         hasPermission(familyId, 'calendar', 'edit') == 'own'));
    }
  }
}
```

### 3. Visual Identity System

#### 3.1 Color Management
```javascript
// Predefined color palette
const familyColorPalette = [
  { name: 'Ruby', hex: '#E91E63', contrast: '#FFFFFF' },
  { name: 'Emerald', hex: '#4CAF50', contrast: '#FFFFFF' },
  { name: 'Sapphire', hex: '#2196F3', contrast: '#FFFFFF' },
  { name: 'Amber', hex: '#FFC107', contrast: '#000000' },
  { name: 'Violet', hex: '#9C27B0', contrast: '#FFFFFF' },
  { name: 'Teal', hex: '#009688', contrast: '#FFFFFF' },
  { name: 'Orange', hex: '#FF5722', contrast: '#FFFFFF' },
  { name: 'Indigo', hex: '#3F51B5', contrast: '#FFFFFF' }
];

// Auto-assign colors to new members
const assignMemberColor = (existingColors) => {
  const usedColors = new Set(existingColors);
  const available = familyColorPalette.filter(c => !usedColors.has(c.hex));
  
  if (available.length > 0) {
    return available[0];
  }
  
  // If all colors used, generate variation
  return generateColorVariation(existingColors);
};
```

#### 3.2 Color Application
```typescript
// Color usage throughout app
interface ThemedComponent {
  memberColor: string;
  contrastColor: string;
}

// Calendar events
const EventComponent = ({ event, member }) => (
  <View style={{
    backgroundColor: member.color,
    borderLeftColor: member.color,
    borderLeftWidth: 4
  }}>
    <Text style={{ color: member.contrastColor }}>
      {event.title}
    </Text>
  </View>
);

// Chore cards
const ChoreCard = ({ member, tasks }) => (
  <Card style={{
    borderTopColor: member.color,
    borderTopWidth: 3
  }}>
    <Avatar
      backgroundColor={member.color}
      content={member.avatar}
    />
    {/* Task list */}
  </Card>
);
```

### 4. Presence & Activity Tracking

#### 4.1 Presence System
```javascript
// Realtime Database presence
const setupPresence = (userId, familyId) => {
  const userStatusRef = realtimeDb.ref(`families/${familyId}/presence/${userId}`);
  const isOfflineForDatabase = {
    online: false,
    lastSeen: serverTimestamp()
  };
  
  const isOnlineForDatabase = {
    online: true,
    lastSeen: serverTimestamp(),
    device: getDeviceInfo()
  };
  
  // Set online when connected
  realtimeDb.ref('.info/connected').on('value', (snapshot) => {
    if (snapshot.val() === false) return;
    
    userStatusRef.onDisconnect()
      .set(isOfflineForDatabase)
      .then(() => {
        userStatusRef.set(isOnlineForDatabase);
      });
  });
};
```

#### 4.2 Activity Logging
```javascript
// Activity tracking schema
/families/{familyId}/activityLog/{logId}
{
  userId: string,
  userName: string,
  action: string, // 'created', 'updated', 'completed', etc.
  module: 'calendar' | 'chores' | 'meals' | 'settings',
  
  // Context-specific data
  details: {
    entityType: string, // 'event', 'task', 'recipe'
    entityId: string,
    entityName: string,
    
    // For updates
    changes: [{
      field: string,
      oldValue: any,
      newValue: any
    }]
  },
  
  timestamp: timestamp,
  deviceInfo: {
    platform: string,
    version: string
  }
}

// Cloud Function to log activities
exports.logActivity = functions.firestore
  .document('{collection}/{docId}')
  .onWrite(async (change, context) => {
    const { collection } = context.params;
    
    // Determine action type
    const action = !change.before.exists ? 'created' 
                 : !change.after.exists ? 'deleted'
                 : 'updated';
    
    // Extract user info from auth context
    const userId = context.auth?.uid;
    if (!userId) return;
    
    // Create activity log entry
    await createActivityLog({
      userId,
      action,
      module: getModuleFromCollection(collection),
      details: extractDetails(change),
      timestamp: serverTimestamp()
    });
  });
```

### 5. Family Settings Management

#### 5.1 Family Settings Schema
```javascript
/families/{familyId}/settings
{
  // General
  general: {
    familyName: string,
    timezone: string,
    country: string,
    language: string,
    dateFormat: 'MM/DD/YYYY' | 'DD/MM/YYYY',
    timeFormat: '12h' | '24h',
    weekStartsOn: 0-6 // Sunday-Saturday
  },
  
  // Module Configuration
  modules: {
    calendar: {
      enabled: boolean,
      defaultView: 'month' | 'week',
      workingHours: {
        start: string, // "09:00"
        end: string    // "17:00"
      },
      syncFrequency: number, // minutes
      enabledCategories: string[]
    },
    
    chores: {
      enabled: boolean,
      resetTime: string, // "00:00"
      streakGracePeriod: number, // hours
      pointsEnabled: boolean,
      requirePhotoProof: boolean,
      defaultTaskDuration: number // minutes
    },
    
    meals: {
      enabled: boolean,
      planningDay: 0-6, // Day to plan week
      defaultServings: number,
      dietaryRestrictions: string[],
      budgetTracking: boolean
    }
  },
  
  // Notifications
  notifications: {
    channels: {
      push: boolean,
      email: boolean,
      sms: boolean
    },
    
    // Per-module settings
    calendar: {
      eventReminders: number[], // minutes before
      dailySummary: string, // time "07:00"
    },
    
    chores: {
      taskReminders: boolean,
      completionNotifications: boolean,
      streakNotifications: boolean
    }
  },
  
  // Privacy
  privacy: {
    shareAnalytics: boolean,
    allowSupport: boolean,
    dataRetention: number // days
  },
  
  // Billing (future)
  subscription: {
    plan: 'free' | 'family' | 'premium',
    status: 'active' | 'trial' | 'expired',
    nextBilling: timestamp,
    paymentMethod: string
  },
  
  lastModified: timestamp,
  modifiedBy: string
}
```

#### 5.2 Settings UI
```typescript
// Settings screen structure
interface SettingsScreen {
  sections: [
    {
      title: "Family",
      items: [
        "Family Name",
        "Time Zone",
        "Invite Members",
        "Manage Members"
      ]
    },
    {
      title: "Modules",
      items: [
        "Calendar Settings",
        "Chores Settings",
        "Meals Settings"
      ]
    },
    {
      title: "Notifications",
      items: [
        "Push Notifications",
        "Email Preferences",
        "Reminder Times"
      ]
    },
    {
      title: "Account",
      items: [
        "Subscription",
        "Privacy",
        "Data Export",
        "Delete Family"
      ]
    }
  ]
}
```

### 6. Profile Management UI

#### 6.1 Member List View
```typescript
interface MemberListProps {
  familyId: string;
  members: FamilyMember[];
  currentUser: User;
  onMemberPress: (member: FamilyMember) => void;
  onInvitePress: () => void;
}

// Features:
// - Grid view with avatars
// - Online/offline indicators
// - Role badges
// - Last active time
// - Quick actions menu
```

#### 6.2 Profile Editor
```typescript
interface ProfileEditorProps {
  member: FamilyMember;
  isOwnProfile: boolean;
  canEdit: boolean;
  onSave: (updates: Partial<FamilyMember>) => void;
}

// Sections:
// - Avatar selector
// - Name and nickname
// - Color picker
// - Role selector (if permitted)
// - Permission toggles
// - Notification preferences
```

## Deliverables

### 1. Profile System
- [ ] Extended profile schema
- [ ] Avatar system with all types
- [ ] Color assignment and management
- [ ] Profile editing UI
- [ ] Age calculation from DOB

### 2. Permissions System
- [ ] Role definitions implemented
- [ ] Security rules for all modules
- [ ] Permission checking utilities
- [ ] Role assignment UI
- [ ] Permission inheritance

### 3. Visual Identity
- [ ] Color system throughout app
- [ ] Avatar display components
- [ ] Consistent member identification
- [ ] Theme support (light/dark)

### 4. Activity Tracking
- [ ] Presence system online/offline
- [ ] Activity logging functions
- [ ] Activity feed UI
- [ ] Device tracking

### 5. Settings Management
- [ ] Family settings schema
- [ ] Settings UI screens
- [ ] Module configuration
- [ ] Notification preferences

## Testing Requirements

### 1. Permission Tests
- Role-based access for each module
- Permission inheritance
- Edge cases (removed members, etc.)
- Security rule validation

### 2. Profile Tests
- Avatar upload and display
- Color assignment conflicts
- Profile update permissions
- Age calculation accuracy

### 3. Integration Tests
- Presence system reliability
- Activity logging completeness
- Settings propagation
- Cross-module identity

## Performance Requirements

### 1. Profile Operations
- Profile load: < 500ms
- Avatar upload: < 3 seconds
- Settings save: < 1 second
- Member list: < 1 second

### 2. Real-time Features
- Presence update: < 2 seconds
- Activity log: Near instant
- Permission checks: < 100ms

## Definition of Done

### Functionality
- [ ] All deliverables complete
- [ ] Permissions enforced across modules
- [ ] Visual identity consistent
- [ ] Settings fully functional

### Quality
- [ ] Security rules tested
- [ ] No permission bypasses
- [ ] UI responsive and smooth
- [ ] Offline handling works

### Documentation
- [ ] Permission matrix documented
- [ ] Settings descriptions
- [ ] Profile field validation
- [ ] API documentation

## Risks & Mitigations

### 1. Permission Complexity
- **Risk:** Confusing permission model
- **Mitigation:** Clear role presets, simple UI

### 2. Color Conflicts
- **Risk:** Similar colors for different members
- **Mitigation:** Contrast checking, variations

### 3. Privacy Concerns
- **Risk:** Children's data exposed
- **Mitigation:** Age-based privacy controls

## Dependencies
- Authentication system (Sprint 1)
- Both modules functional (Sprints 2-3)
- Image upload infrastructure
- Push notification setup

## Next Sprint Preview
[Sprint 5: External Data Integration](PRD-105-Sprint-5.md) - Email/SMS parsing and Meals placeholder