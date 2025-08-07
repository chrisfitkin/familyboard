# FamilyBoard

A cross-platform family management app with dedicated phone and tablet interfaces.

## Structure

This is a monorepo containing:
- `apps/mobile` - React Native phone app for family management
- `apps/tablet` - React Native tablet app with interactive dashboard view
- `packages/firebase` - Firebase integration with local emulator support
- `packages/shared` - Shared types, constants, and utilities

## Getting Started

### Prerequisites
- Node.js 18+
- iOS: Xcode 15+ and CocoaPods
- Android: Android Studio and Java 17+
- Firebase CLI (for local emulators)

### Installation

```bash
# Install dependencies
npm install

# iOS setup (from each app directory)
cd apps/mobile/ios && pod install
cd apps/tablet/ios && pod install
```

### Running the Apps

#### Phone App
```bash
# Start Metro bundler
npm run dev:phone

# Run on iOS
npm run ios

# Run on Android
npm run android
```

#### Tablet App
```bash
# Start Metro bundler
npm run dev:tablet

# Run on iOS
npm run ios:tablet

# Run on Android
npm run android:tablet
```

### Firebase Local Development

```bash
# Start Firebase emulators
npm run dev:firebase

# Access Firebase UI at http://localhost:4000
```

## Features

### Phone App
- User authentication
- Family management
- Board categories (Events, Tasks, Notes, Photos)
- Settings and profile management

### Tablet App
- Interactive dashboard view
- Split-screen layout with sidebar navigation
- Real-time family activity overview
- Category-specific views optimized for larger screens

## Development

The apps use:
- React Native 0.80.2
- TypeScript
- React Navigation for mobile navigation
- Firebase for backend (with local emulator support)
- Shared design system with consistent colors and spacing