# PRD-103: Sprint 3 - Chores Module Core
*FamilyBoard MVP Sprint 3 Requirements*

## Sprint Information
- **Sprint ID:** PRD-103
- **Sprint Name:** Chores Module Core
- **Duration:** 2 weeks (Weeks 5-6)
- **Goal:** Build complete Chores module with task management and streak tracking
- **Parent Documents:**
  - [PRD-001](../PRD-001-FamilyBoard-Overview.md)
  - [PRD-003](../modules/PRD-003-Chores-Module.md)
- **Dependencies:** Sprint 1 completion (auth & navigation)

## Sprint Objectives

### Primary Goals
1. Design and implement task data model
2. Build recurring task generation system
3. Create task management UI for tablet
4. Implement streak tracking logic
5. Enable real-time task updates

### Success Criteria
- Complete task creation and management on mobile
- Real-time task updates on tablet
- Working streak tracking with visual indicators
- Automated recurring task generation

## Technical Requirements

### 1. Task Data Model

#### 1.1 Core Task Schema
```javascript
/families/{familyId}/tasks/{taskId}
{
  // Basic Info
  id: string,
  title: string,
  description: string,
  
  // Scheduling
  frequency: 'daily' | 'weekly' | 'monthly' | 'once',
  dueDate: timestamp,
  dueTime: string, // "HH:MM" format, optional
  
  // Recurrence (for weekly/monthly)
  recurringConfig: {
    // Weekly: array of days (0=Sun, 6=Sat)
    daysOfWeek: number[],
    
    // Monthly: day of month or relative
    dayOfMonth: number, // 1-31
    weekOfMonth: number, // 1-4 (first, second, etc.)
    dayOfWeekInMonth: number, // 0-6 (first Monday, etc.)
    
    // End conditions
    endDate: timestamp,
    endAfterOccurrences: number
  },
  
  // Assignment
  assignmentType: 'individual' | 'rotating' | 'household',
  assignedTo: string[], // User IDs
  rotationIndex: number, // Current position in rotation
  ageRestriction: {
    minAge: number,
    maxAge: number
  },
  
  // Task Details
  category: 'daily' | 'chore' | 'academic' | 'fitness' | 'personal',
  priority: 'low' | 'medium' | 'high',
  estimatedMinutes: number,
  points: number,
  
  // Completion Requirements
  requiresPhoto: boolean,
  requiresNote: boolean,
  requiresParentApproval: boolean,
  
  // Metadata
  createdBy: string,
  created: timestamp,
  lastModified: timestamp,
  active: boolean,
  
  // Parent task reference (for instances)
  templateId: string,
  isTemplate: boolean
}
```

#### 1.2 Task Instance Schema
```javascript
/families/{familyId}/taskInstances/{instanceId}
{
  taskId: string, // Reference to parent task
  assignedTo: string,
  dueDate: timestamp,
  
  // Status
  status: 'pending' | 'completed' | 'skipped' | 'overdue',
  
  // Completion data
  completedBy: string,
  completedAt: timestamp,
  completionNote: string,
  completionPhotoUrl: string,
  approvedBy: string,
  approvedAt: timestamp,
  
  // Metadata
  created: timestamp,
  notificationsSent: number[]
}
```

#### 1.3 Streak Schema
```javascript
/families/{familyId}/streaks/{userId}
{
  userId: string,
  
  // Global streaks
  globalStreak: {
    current: number,
    best: number,
    startDate: timestamp,
    lastCompletedDate: timestamp
  },
  
  // Per-task streaks
  taskStreaks: {
    [taskId]: {
      current: number,
      best: number,
      startDate: timestamp,
      lastCompletedDate: timestamp,
      totalCompletions: number
    }
  },
  
  // Daily completion
  dailyCompletion: {
    [date: string]: { // YYYY-MM-DD format
      totalTasks: number,
      completedTasks: number,
      completionRate: number
    }
  },
  
  // Achievements
  achievements: [{
    type: string,
    earnedAt: timestamp,
    taskId: string
  }],
  
  lastUpdated: timestamp
}
```

### 2. Task Generation System

#### 2.1 Cloud Functions
```javascript
// Scheduled function to generate daily task instances
exports.generateDailyTasks = functions.pubsub
  .schedule('0 0 * * *') // Midnight UTC
  .timeZone('America/New_York') // Configurable per family
  .onRun(async (context) => {
    const families = await getFamiliesWithChores();
    
    for (const family of families) {
      const tasks = await getActiveTasks(family.id);
      const today = getToday(family.timezone);
      
      for (const task of tasks) {
        if (shouldGenerateInstance(task, today)) {
          await createTaskInstance(task, today);
        }
      }
      
      // Update family's task generation timestamp
      await updateLastGeneration(family.id);
    }
  });

// Helper function to determine if task should be generated
const shouldGenerateInstance = (task, date) => {
  switch (task.frequency) {
    case 'daily':
      return true;
      
    case 'weekly':
      const dayOfWeek = date.getDay();
      return task.recurringConfig.daysOfWeek.includes(dayOfWeek);
      
    case 'monthly':
      return matchesMonthlyConfig(task.recurringConfig, date);
      
    case 'once':
      return isSameDay(task.dueDate, date);
  }
};
```

#### 2.2 Task Assignment Logic
```javascript
// Assign tasks based on assignment type
const assignTaskInstance = async (task, familyId) => {
  const familyMembers = await getFamilyMembers(familyId);
  
  switch (task.assignmentType) {
    case 'individual':
      return task.assignedTo[0];
      
    case 'rotating':
      const eligibleMembers = filterByAge(
        familyMembers,
        task.ageRestriction
      );
      const assignee = eligibleMembers[task.rotationIndex % eligibleMembers.length];
      
      // Update rotation index for next time
      await updateRotationIndex(task.id, task.rotationIndex + 1);
      
      return assignee.id;
      
    case 'household':
      return null; // Anyone can complete
  }
};
```

### 3. Chores UI Implementation

#### 3.1 Tablet Layout
```typescript
// Main chores view component
interface ChoresViewProps {
  familyId: string;
  familyMembers: FamilyMember[];
  currentDate: Date;
}

// Layout structure:
// - Header: Date, family stats
// - Grid/List: Member cards with tasks
// - Footer: Legend, quick actions

// Member task card
interface MemberTaskCardProps {
  member: FamilyMember;
  tasks: TaskInstance[];
  streaks: StreakData;
  onTaskComplete: (taskId: string) => void;
  onTaskDetails: (taskId: string) => void;
}
```

#### 3.2 Task List Component
```typescript
interface TaskListProps {
  tasks: TaskInstance[];
  groupBy: 'time' | 'category' | 'status';
  onTaskPress: (task: TaskInstance) => void;
  onTaskComplete: (task: TaskInstance) => void;
}

// Features:
// - Grouped sections (overdue, today, upcoming)
// - Swipe to complete
// - Long press for details
// - Visual status indicators
```

#### 3.3 Streak Visualization
```typescript
interface StreakDisplayProps {
  streak: StreakData;
  size: 'small' | 'large';
  showDetails: boolean;
}

// Visual elements:
// - Fire icon with animation
// - Number display
// - Progress to next milestone
// - Best streak indicator
```

### 4. Real-time Updates

#### 4.1 Realtime Database Structure
```javascript
/families/{familyId}/choreUpdates/
{
  lastUpdate: timestamp,
  
  // Task completions
  completions: {
    [instanceId]: {
      completedBy: string,
      completedAt: timestamp
    }
  },
  
  // Streak updates
  streakUpdates: {
    [userId]: {
      current: number,
      updatedAt: timestamp
    }
  },
  
  // New task notifications
  newTasks: {
    [instanceId]: {
      assignedTo: string,
      createdAt: timestamp
    }
  }
}
```

#### 4.2 Optimistic Updates
```javascript
// Complete task with optimistic UI update
const completeTask = async (taskInstance) => {
  // 1. Update UI immediately
  updateTaskStatus(taskInstance.id, 'completed');
  
  // 2. Update Firestore
  try {
    await firestore
      .collection(`families/${familyId}/taskInstances`)
      .doc(taskInstance.id)
      .update({
        status: 'completed',
        completedBy: currentUser.id,
        completedAt: serverTimestamp()
      });
    
    // 3. Update streaks
    await updateStreaks(currentUser.id, taskInstance.taskId);
    
    // 4. Notify other devices
    await realtimeDb
      .ref(`families/${familyId}/choreUpdates/completions/${taskInstance.id}`)
      .set({
        completedBy: currentUser.id,
        completedAt: Date.now()
      });
      
  } catch (error) {
    // Revert UI on failure
    updateTaskStatus(taskInstance.id, 'pending');
    showError('Failed to complete task');
  }
};
```

### 5. Streak Tracking

#### 5.1 Streak Calculation Logic
```javascript
// Cloud Function: Update streaks after task completion
exports.updateStreaks = functions.firestore
  .document('families/{familyId}/taskInstances/{instanceId}')
  .onUpdate(async (change, context) => {
    const before = change.before.data();
    const after = change.after.data();
    
    // Check if task was just completed
    if (before.status !== 'completed' && after.status === 'completed') {
      const { familyId } = context.params;
      const userId = after.completedBy;
      
      // Get user's streak data
      const streakRef = firestore
        .collection(`families/${familyId}/streaks`)
        .doc(userId);
      
      const streakDoc = await streakRef.get();
      const streakData = streakDoc.exists ? streakDoc.data() : createNewStreakData(userId);
      
      // Update global streak
      const lastDate = streakData.globalStreak.lastCompletedDate?.toDate();
      const today = new Date();
      const daysSinceLastCompletion = getDaysBetween(lastDate, today);
      
      if (daysSinceLastCompletion === 1) {
        // Continue streak
        streakData.globalStreak.current += 1;
      } else if (daysSinceLastCompletion > 1) {
        // Streak broken, start new
        streakData.globalStreak.current = 1;
        streakData.globalStreak.startDate = serverTimestamp();
      }
      
      // Update best streak if needed
      if (streakData.globalStreak.current > streakData.globalStreak.best) {
        streakData.globalStreak.best = streakData.globalStreak.current;
      }
      
      // Update task-specific streak
      const taskStreak = streakData.taskStreaks[after.taskId] || createNewTaskStreak();
      // ... similar logic for task streak
      
      // Save updated streaks
      await streakRef.set(streakData, { merge: true });
      
      // Check for achievements
      await checkStreakAchievements(userId, streakData);
    }
  });
```

#### 5.2 Streak Recovery Rules
```javascript
const GRACE_PERIOD_HOURS = 10; // Until 10 AM next day

const isStreakBroken = (lastCompletedDate, currentDate) => {
  const timeDiff = currentDate - lastCompletedDate;
  const hoursDiff = timeDiff / (1000 * 60 * 60);
  
  // If last completion was yesterday but it's before grace period
  if (hoursDiff < 24 + GRACE_PERIOD_HOURS) {
    return false;
  }
  
  return true;
};
```

### 6. Mobile Management Features

#### 6.1 Task Creation Flow
```typescript
// Task creation screens
interface CreateTaskFlow {
  // Step 1: Basic info
  basicInfo: {
    title: string;
    description?: string;
    category: TaskCategory;
  };
  
  // Step 2: Scheduling
  scheduling: {
    frequency: TaskFrequency;
    startDate: Date;
    recurringConfig?: RecurringConfig;
  };
  
  // Step 3: Assignment
  assignment: {
    type: AssignmentType;
    assignees: string[];
    ageRestriction?: AgeRange;
  };
  
  // Step 4: Details
  details: {
    points?: number;
    estimatedMinutes?: number;
    requiresPhoto?: boolean;
  };
}
```

#### 6.2 Task Templates
```javascript
// Predefined task templates
const taskTemplates = {
  dailyReading: {
    title: "Read for 30 minutes",
    category: "academic",
    frequency: "daily",
    points: 5,
    estimatedMinutes: 30
  },
  
  weeklyLaundry: {
    title: "Do laundry",
    category: "chore",
    frequency: "weekly",
    recurringConfig: { daysOfWeek: [6] }, // Saturday
    points: 10,
    estimatedMinutes: 60
  },
  
  // ... more templates
};
```

## Deliverables

### 1. Task Management System
- [ ] Task creation with all options
- [ ] Recurring task configuration
- [ ] Task assignment logic
- [ ] Age-based filtering
- [ ] Task templates library

### 2. Automated Generation
- [ ] Daily task generation function
- [ ] Timezone-aware scheduling
- [ ] Rotation assignment
- [ ] Error handling and retries

### 3. Tablet UI
- [ ] Family member task cards
- [ ] Task list with grouping
- [ ] Completion interactions
- [ ] Visual status indicators
- [ ] Pull-to-refresh

### 4. Streak System
- [ ] Streak calculation logic
- [ ] Visual streak display
- [ ] Achievement detection
- [ ] Best streak tracking
- [ ] Recovery grace period

### 5. Real-time Features
- [ ] Live task updates
- [ ] Completion animations
- [ ] Streak update notifications
- [ ] Offline task queue

## Testing Requirements

### 1. Unit Tests
- Task generation logic
- Streak calculations
- Assignment algorithms
- Date/timezone handling
- Recurrence patterns

### 2. Integration Tests
- Cloud Function triggers
- Firestore transactions
- Realtime sync
- Offline/online transitions

### 3. UI Tests
- Task completion flow
- Streak animations
- Navigation between views
- Error state handling

## Performance Requirements

### 1. Response Times
- Task list load: < 1 second
- Task completion: < 500ms feedback
- Streak update: < 2 seconds
- Generation job: < 5 minutes for all families

### 2. Scalability
- Support 100+ tasks per family
- Handle 10,000+ families
- Efficient batch operations
- Optimized queries

## Definition of Done

### Functionality
- [ ] All deliverables complete
- [ ] Recurring tasks generating correctly
- [ ] Streaks calculating accurately
- [ ] Real-time sync working

### Quality
- [ ] Test coverage > 80%
- [ ] No critical bugs
- [ ] Performance targets met
- [ ] Error handling complete

### Documentation
- [ ] Task schema documented
- [ ] API reference complete
- [ ] User guide written
- [ ] Architecture diagram

## Risks & Mitigations

### 1. Timezone Complexity
- **Risk:** Incorrect task generation across timezones
- **Mitigation:** Comprehensive timezone testing, UTC storage

### 2. Streak Calculation Bugs
- **Risk:** Incorrect streak counts frustrate users
- **Mitigation:** Extensive unit tests, manual verification tools

### 3. Performance at Scale
- **Risk:** Slow generation with many families
- **Mitigation:** Batch processing, queue system, caching

## Dependencies
- Sprint 1 authentication system
- Firebase Functions deployment
- Notification service setup
- Family member profiles

## Next Sprint Preview
[Sprint 4: Family Member Management](PRD-104-Sprint-4.md) - Complete profiles and personalization