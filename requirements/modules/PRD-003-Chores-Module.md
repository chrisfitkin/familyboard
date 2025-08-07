# PRD-003: Chores Module
*FamilyBoard Chores Module Requirements*

## Document Information
- **Document ID:** PRD-003
- **Parent Document:** [PRD-001](../PRD-001-FamilyBoard-Overview.md)
- **Module Type:** Core MVP Module
- **Status:** Active

## 1. Module Overview

### 1.1 Purpose
Comprehensive task management system for daily, weekly, and monthly household responsibilities that promotes accountability, builds good habits, and fairly distributes household work among family members.

### 1.2 Key Value Propositions
- **Habit Formation:** Daily must-dos build consistent routines
- **Fair Distribution:** Evenly spread tasks across family
- **Visual Progress:** See completion status at a glance
- **Motivation:** Streak tracking encourages consistency

## 2. Functional Requirements

### 2.1 Task Types & Frequency

#### 2.1.1 Daily Tasks (Every Day)
- **Definition:** Tasks that must be completed daily
- **Reset Time:** Configurable (default: midnight)
- **Examples:**
  - Read for 30 minutes
  - Exercise/fitness activities
  - Make bed
  - Clear dishes

#### 2.1.2 Weekly Recurring Tasks
- **Definition:** Tasks on specific day(s) each week
- **Scheduling:** Assign to specific weekdays
- **Examples:**
  - Take out trash (Sunday nights)
  - Water plants (Wednesdays)
  - Laundry (specific days)

#### 2.1.3 Monthly Tasks
- **Definition:** Tasks that occur monthly
- **Scheduling:** Specific date or "last Sunday"
- **Examples:**
  - Change air filters
  - Deep clean bathroom
  - Pay allowances

#### 2.1.4 One-Time Tasks
- **Definition:** Non-recurring tasks
- **Assignment:** To individual or household
- **Examples:**
  - Organize garage
  - Buy school supplies
  - Schedule appointments

### 2.2 Task Categories

#### 2.2.1 Kids' Daily Must-Dos
**Academic:**
- Read for 30 minutes
- Practice instrument
- Complete homework

**Physical:**
- Pull-ups/push-ups
- Squats/exercises
- Outdoor time

**Household:**
- Wipe table
- Set table
- Clear counters
- Feed pets

#### 2.2.2 Parent Tasks
**Household Management:**
- Review mail
- Pay bills
- Meal prep
- Schedule appointments

**Maintenance:**
- Check locks
- Water plants
- Charge devices
- Security check

**Specific Day Tasks:**
- Trash night prep
- Recycling sort
- Lawn care scheduling

#### 2.2.3 Weekly Recurring
**Cleaning Rotation:**
- Bathrooms (assigned by week)
- Kitchen deep clean
- Vacuum main areas
- Mop floors

**Laundry Schedule:**
- Sheets (Fridays)
- Towels (Wednesdays)
- Clothes (by person)

**Outdoor:**
- Lawn mowing
- Garden watering
- Pool maintenance

### 2.3 Task Management Features

#### 2.3.1 Task Persistence
- **Incomplete Handling:** Tasks remain visible until done
- **Visual Indicators:** 
  - Red: Overdue
  - Yellow: Due today
  - Green: Completed
  - Gray: Future
- **Non-Blocking:** New tasks appear regardless of old

#### 2.3.2 Task Assignment
- **Individual:** Assigned to specific person
- **Rotating:** Automatically rotates between people
- **Household:** Anyone can complete
- **Age-Appropriate:** Filter by age group

#### 2.3.3 Completion Tracking
- **One-Tap:** Large checkbox for easy completion
- **Timestamp:** Record when completed
- **Who Completed:** Track if different from assigned
- **Undo Option:** 5-minute undo window

#### 2.3.4 Streak Tracking
- **Visual Streaks:** Fire icon with day count
- **Streak Rules:**
  - Daily tasks: Consecutive days
  - Weekly tasks: Consecutive weeks
  - Grace period: Until next due date
- **Milestones:** Celebrate 7, 30, 100 days
- **Recovery:** Show "best streak" after break

### 2.4 Visual Design

#### 2.4.1 Layout Structure
- **Card-Based:** One card per family member
- **Header:** Name, avatar, today's progress
- **Task List:** Scrollable task list
- **Footer:** Streak info, weekly summary

#### 2.4.2 Progress Indicators
- **Daily Progress Bar:** X of Y tasks complete
- **Circular Progress:** For individual cards
- **Weekly Heat Map:** Shows completion patterns
- **Streak Flames:** Visual streak counter

#### 2.4.3 Interactive Elements
- **Checkbox:** Large, satisfying animation
- **Swipe Actions:** Mark done, skip, reassign
- **Long Press:** View task details
- **Pull to Refresh:** Update task list

## 3. Technical Requirements

### 3.1 Firebase Implementation

#### 3.1.1 Firestore Schema
```javascript
/families/{familyId}/tasks/{taskId}
{
  title: string,
  description: string,
  category: enum,
  frequency: 'daily' | 'weekly' | 'monthly' | 'once',
  assignedTo: string[], // user IDs
  dueDate: timestamp,
  recurringConfig: {
    daysOfWeek: number[], // 0-6 for weekly
    dayOfMonth: number, // for monthly
    endDate: timestamp // optional
  },
  points: number,
  requiresPhoto: boolean,
  created: timestamp,
  createdBy: string
}

/families/{familyId}/taskCompletions/{completionId}
{
  taskId: string,
  completedBy: string,
  completedAt: timestamp,
  notes: string,
  photoUrl: string
}

/families/{familyId}/streaks/{userId}
{
  currentStreak: number,
  bestStreak: number,
  lastCompleted: timestamp,
  streakTasks: {
    [taskId]: {
      current: number,
      best: number,
      lastCompleted: timestamp
    }
  }
}
```

#### 3.1.2 Cloud Functions
- **generateRecurringTasks:** Daily job to create tasks
- **calculateStreaks:** Update streak counts
- **assignRotatingTasks:** Handle task rotation
- **sendReminders:** Task reminder notifications

#### 3.1.3 Realtime Updates
- **Path:** `/families/{familyId}/taskStatus`
- **Updates:** Instant completion sync
- **Optimistic UI:** Update before server confirms

### 3.2 Business Logic

#### 3.2.1 Task Generation
- **Timing:** Run at family's configured time
- **Logic:** 
  - Check recurring configs
  - Generate tasks for next period
  - Assign based on rotation rules
  - Skip holidays if configured

#### 3.2.2 Rotation Algorithm
- **Fair Distribution:** Equal tasks per person
- **Age Appropriate:** Filter by age limits
- **History Aware:** Avoid recent repeats
- **Manual Override:** Allow reassignment

#### 3.2.3 Streak Calculation
- **Daily Streaks:** Must complete by end of day
- **Weekly Streaks:** Must complete by due date
- **Grace Period:** Next day until 10 AM
- **Partial Credit:** 80% completion counts

## 4. Gamification Features

### 4.1 Points System
- **Task Values:** 1-10 points based on effort
- **Bonus Points:** Streak milestones
- **Leaderboards:** Weekly/monthly family ranks
- **Rewards:** Configurable by parents

### 4.2 Achievements
- **Streak Badges:** 7, 30, 100 day streaks
- **Consistency Awards:** Weekly completion
- **Helper Badges:** Completing others' tasks
- **Special Recognition:** Above and beyond

### 4.3 Motivational Elements
- **Encouraging Messages:** Personalized feedback
- **Progress Celebrations:** Animations on milestones
- **Family Goals:** Collective targets
- **Visual Rewards:** Unlock themes/avatars

## 5. Parent Controls

### 5.1 Task Management
- Create/edit/delete tasks
- Set point values
- Configure recurrence
- Approve completions

### 5.2 Reward Settings
- Point redemption rules
- Allowance integration
- Special privileges
- Screen time rewards

### 5.3 Analytics Dashboard
- Completion rates by child
- Task distribution fairness
- Trending problem areas
- Historical patterns

## 6. Mobile App Features

### 6.1 Quick Actions
- Rapid task creation
- Bulk task assignment
- Quick reassignment
- Photo verification

### 6.2 Parent View
- Override completions
- Adjust streaks
- Send motivations
- Review analytics

## 7. Success Metrics

### 7.1 Engagement Metrics
- Daily active users
- Task completion rate
- Streak maintenance rate
- Feature adoption

### 7.2 Behavioral Metrics
- Habit formation (30+ day streaks)
- Self-initiated completions
- Reduced parent reminders
- Task completion timing

### 7.3 Family Metrics
- Household task coverage
- Distribution fairness index
- Parent satisfaction scores
- Child engagement levels

## 8. Future Enhancements

### 8.1 Phase 2 Features
- Chore trading marketplace
- Allowance automation
- Voice completion ("Alexa, I finished dishes")
- AR verification

### 8.2 Advanced Features
- AI task suggestions
- Seasonal task templates
- Integration with smart home
- Collaborative tasks

### 8.3 Social Features
- Family competitions
- Share achievements
- Community templates
- Coaching tips

## 9. Dependencies
- Firebase Cloud Functions
- Firebase Realtime Database
- Cloud Scheduler
- Push Notification service
- Image storage for photos

## 10. Related Documents
- [Sprint 3: Chores Module Core](../sprints/PRD-103-Sprint-3.md)
- [Sprint 4: Family Member Management](../sprints/PRD-104-Sprint-4.md)