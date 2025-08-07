# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

FamilyBoard is a cross-platform family management application built as a monorepo. It consists of React Native apps for mobile and tablet devices, with Firebase as the backend. The project aims to create a centralized digital hub for families to stay organized through an always-on display.

## Essential Development Commands

### Starting Development

```bash
# Install dependencies (from root)
npm install

# Start mobile app
npm run dev:phone     # or npm run dev (default)

# Start tablet app
npm run dev:tablet

# Start Firebase emulators (required for backend)
npm run dev:firebase
```

### Platform-specific Development

```bash
# iOS development
npm run ios          # Mobile iOS
npm run ios:tablet   # Tablet iOS

# Android development
npm run android      # Mobile Android
npm run android:tablet # Tablet Android
```

### Testing & Quality

```bash
# Run tests (from specific app directory)
cd apps/mobile && npm test
cd apps/tablet && npm test

# Lint code (from specific app directory)
cd apps/mobile && npm run lint
cd apps/tablet && npm run lint
```

### Maintenance Commands

```bash
# Clean build caches
npm run clean            # Clean both Metro and Watchman
npm run clean:watchman   # Clear Watchman watches only
npm run clean:metro      # Clear Metro cache only
```

### Firebase Emulator Management

```bash
cd packages/firebase
npm run emulators              # Start emulators
npm run emulators:export       # Export emulator data
npm run emulators:import       # Import saved emulator data
```

## Architecture & Project Structure

### Monorepo Layout

```
familyboard/
├── apps/
│   ├── mobile/          # React Native phone app
│   └── tablet/          # React Native tablet app
├── packages/
│   ├── firebase/        # Firebase integration & config
│   └── shared/          # Shared types, utils, constants
└── requirements/        # Product requirement documents (PRDs)
```

### Key Technologies

- **React Native 0.80.2** - Cross-platform mobile framework
- **TypeScript** - Type safety across all packages
- **Firebase** - Complete backend solution (Auth, Firestore, Storage)
- **React Navigation** - Mobile app navigation
- **Jest** - Testing framework
- **ESLint** - Code quality enforcement

### Module Path Resolution

The project uses TypeScript path mapping for clean imports:

- `@familyboard/firebase` → `packages/firebase/src`
- `@familyboard/shared` → `packages/shared/src`

### Firebase Configuration

Local development uses Firebase emulators on these ports:

- Auth: 9099
- Firestore: 8080
- Storage: 9199
- Emulator UI: 4000

## Core Data Models

### User Model

```typescript
interface User {
  id: string;
  email: string;
  displayName?: string;
  photoURL?: string;
  createdAt: Date;
  updatedAt: Date;
}
```

### Family Model

```typescript
interface Family {
  id: string;
  name: string;
  members: string[]; // User IDs
  createdBy: string;
  createdAt: Date;
  updatedAt: Date;
}
```

### BoardItem Model

```typescript
interface BoardItem {
  id: string;
  familyId: string;
  type: "event" | "task" | "note" | "photo";
  title: string;
  description?: string;
  createdBy: string;
  assignedTo?: string[];
  dueDate?: Date;
  completed?: boolean;
  createdAt: Date;
  updatedAt: Date;
}
```

## Feature Areas

### Mobile App

- User authentication (email/password)
- Family creation and member management
- Board categories: Events, Tasks, Notes, Photos
- Navigation between different family contexts
- User settings and profile management

### Tablet App

- Split-screen dashboard layout
- Sidebar navigation for larger screens
- Real-time family activity overview
- Statistics cards showing family metrics
- Category-specific detailed views

## Development Workflow

### Setting Up New Features

1. Ensure Firebase emulators are running (`npm run dev:firebase`)
2. Start the appropriate app (`npm run dev:phone` or `npm run dev:tablet`)
3. Use shared types from `@familyboard/shared` for consistency
4. Test on both iOS and Android platforms

### Working with Firebase

- All Firebase logic should be in `packages/firebase`
- Use the provided Firebase context/hooks for data access
- Emulator data can be persisted using export/import commands
- Check `firestore.rules` and `storage.rules` for security configurations

### Testing Guidelines

- Unit tests use Jest with React Native preset
- Test files go in `__tests__/` directories
- Mock Firebase services when testing components
- Run platform-specific tests before commits

## Common Tasks

### Adding a New Screen

1. Create component in appropriate app's `src/screens/` directory
2. Add navigation route if using mobile app
3. Import shared types from `@familyboard/shared`
4. Connect to Firebase using hooks from `@familyboard/firebase`

### Updating Data Models

1. Modify interfaces in `packages/shared/src/types/`
2. Update Firebase security rules if needed
3. Ensure both mobile and tablet apps handle the changes

### Managing Dependencies

- Add app-specific dependencies to the individual app's package.json
- Add shared dependencies to the appropriate package
- Run `npm install` from root to install all workspace dependencies

## Platform Setup Requirements

### Android Development
To run Android apps, you need:
1. **Android Studio** - Download from https://developer.android.com/studio
2. **Android SDK** - Installed via Android Studio
3. **Android Emulator** - Create via AVD Manager in Android Studio
4. **Environment Variables**:
   ```bash
   export ANDROID_HOME=$HOME/Library/Android/sdk
   export PATH=$PATH:$ANDROID_HOME/emulator
   export PATH=$PATH:$ANDROID_HOME/platform-tools
   ```

### iOS Development
To run iOS apps, you need:
1. **Xcode** - Install from Mac App Store
2. **iOS Simulator** - Included with Xcode
3. **CocoaPods** - Install with `sudo gem install cocoapods`

## Troubleshooting

### Metro Bundler Issues

```bash
npm run clean:metro
cd apps/[mobile|tablet] && npm start -- --reset-cache
```

### Firebase Connection Issues

- Ensure emulators are running: `npm run dev:firebase`
- Check that app is configured to use local emulator URLs
- Verify firestore.rules and storage.rules are valid

### Build Failures

1. Clean all caches: `npm run clean`
2. Delete node_modules and reinstall: `rm -rf node_modules && npm install`
3. For iOS: `cd apps/[mobile|tablet]/ios && pod install`

### Android Build Issues

1. Ensure Android SDK is installed and ANDROID_HOME is set
2. Create/update `local.properties` files:
   ```bash
   echo "sdk.dir=$HOME/Library/Android/sdk" > apps/mobile/android/local.properties
   echo "sdk.dir=$HOME/Library/Android/sdk" > apps/tablet/android/local.properties
   ```
3. Start Android emulator manually via Android Studio if needed
4. For monorepo issues, ensure gradle settings point to correct node_modules path

## Product Requirements Documentation

The `requirements/` folder contains comprehensive Product Requirement Documents (PRDs) that define the project vision, features, and implementation roadmap:

### Main Documents

- **PRD-001-FamilyBoard-Overview.md** - Overall product vision, target users, and strategic goals

### Module-Specific PRDs

Located in `requirements/modules/`:

- **PRD-002-Calendar-Module.md** - Family scheduling and event management specifications
- **PRD-003-Chores-Module.md** - Task management and assignment system requirements
- **PRD-004-Meals-Module.md** - Meal planning and grocery list features (Post-MVP)

### Sprint Planning Documents

Located in `requirements/sprints/`:

- **PRD-101-Sprint-1.md** - Foundation & Infrastructure setup
- **PRD-102-Sprint-2.md** - Calendar Module Core implementation
- **PRD-103-Sprint-3.md** - Chores Module Core development
- **PRD-104-Sprint-4.md** - Family Member Management features
- **PRD-105-Sprint-5.md** - External Data Integration plans

Consult these PRDs when implementing features to ensure alignment with product vision and user requirements. Each sprint PRD contains specific acceptance criteria and implementation priorities. Mark requirements complete (- [x]) as they are completed.
