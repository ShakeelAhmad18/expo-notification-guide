# Expo Push Notifications — Complete Implementation Guide

**Author**: Senior React Native Developer  
**Based on**: OneCare AI — Real Production Implementation  
**Date**: April 2026  
**Expo SDK**: 55 | **React Native**: 0.83+

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites & Dependencies](#2-prerequisites--dependencies)
3. [Firebase & FCM Setup](#3-firebase--fcm-setup)
4. [Frontend Implementation (React Native)](#4-frontend-implementation-react-native)
5. [Backend Implementation (Node.js/Express)](#5-backend-implementation-nodejsexpress)
6. [Scheduled Notifications (Cron Jobs)](#6-scheduled-notifications-cron-jobs)
7. [Deep Linking from Notifications](#7-deep-linking-from-notifications)
8. [EAS Build & APK Configuration](#8-eas-build--apk-configuration)
9. [Testing & Debugging](#9-testing--debugging)
10. [Common Errors & Solutions](#10-common-errors--solutions)
11. [Production Checklist](#11-production-checklist)

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    PUSH NOTIFICATION FLOW                       │
│                                                                 │
│  ┌──────────┐         ┌──────────────┐         ┌────────────┐  │
│  │ Your App │──Token──▶│ Your Backend │──Push──▶│ Expo Push  │  │
│  │ (Client) │         │  (Node.js)   │  API    │  Servers   │  │
│  └──────────┘         └──────────────┘         └─────┬──────┘  │
│       ▲                                              │         │
│       │                                              ▼         │
│       │                                        ┌────────────┐  │
│       │◀───────── Notification ─────────────────│  FCM/APNs  │  │
│       │                                        │  (Google)   │  │
│       │                                        └────────────┘  │
│  ┌──────────┐                                                  │
│  │ Physical │                                                  │
│  │  Device  │                                                  │
│  └──────────┘                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Concepts
- **Expo Push Token**: A unique device identifier (e.g., `ExponentPushToken[abc123]`)
- **FCM (Firebase Cloud Messaging)**: Google's service that delivers notifications to Android
- **APNs (Apple Push Notification service)**: Apple's service for iOS
- **Expo Push API**: A unified relay that sits between your backend and FCM/APNs

---

## 2. Prerequisites & Dependencies

### Frontend (React Native / Expo)

```bash
# Install required packages
npx expo install expo-notifications expo-device expo-constants
```

| Package | Purpose |
|---------|---------|
| `expo-notifications` | Core notification API (permissions, tokens, listeners) |
| `expo-device` | Detects if running on a physical device (required for push) |
| `expo-constants` | Reads `app.json` config to get EAS Project ID |

### Backend (Node.js / Express)

```bash
npm install expo-server-sdk node-cron
```

| Package | Purpose |
|---------|---------|
| `expo-server-sdk` | Official Expo SDK to send push notifications from your server |
| `node-cron` | Scheduler for time-based reminders (medication, appointments) |

---

## 3. Firebase & FCM Setup

> [!IMPORTANT]
> This is the most critical step. Without FCM credentials, notifications will NOT be delivered to Android devices in APK/production builds.

### Step 3.1 — Create Firebase Project

1. Go to [Firebase Console](https://console.firebase.google.com)
2. Click **"Add Project"** → Name it → Continue
3. Enable/disable Google Analytics (optional) → Create Project

### Step 3.2 — Register Your Android App

1. In Firebase Console → **Project Settings** → **General** tab
2. Click **"Add App"** → Select **Android**
3. Enter your **package name** (must match `app.json`):
   ```
   Example: com.shakeeldev.userapp
   ```
4. Click **Register App**
5. **Download `google-services.json`**
6. Place it in your **project root** (same level as `app.json`):
   ```
   UserApp/
   ├── app.json
   ├── google-services.json   ← HERE
   ├── package.json
   └── src/
   ```

### Step 3.3 — Configure app.json

```json
{
  "expo": {
    "android": {
      "package": "com.yourcompany.yourapp",
      "googleServicesFile": "./google-services.json"
    },
    "plugins": [
      ["expo-notifications", {
        "icon": "./assets/icon.png",
        "color": "#E6F4FE"
      }]
    ],
    "extra": {
      "eas": {
        "projectId": "your-eas-project-id-here"
      }
    }
  }
}
```

> **How to get EAS Project ID**: Run `eas init` or find it at [expo.dev](https://expo.dev) → Your Project → Settings

### Step 3.4 — Upload FCM V1 Credentials to Expo

This connects Expo's push servers to your Firebase project:

1. Firebase Console → **Project Settings** → **Service Accounts** tab
2. Click **"Generate new private key"** → Download the JSON
3. Go to [expo.dev](https://expo.dev) → Your Project → **Credentials** → **Android**
4. Under **FCM V1 Service Account Key** → Upload the JSON

**OR via CLI:**
```bash
eas credentials
# Select: Android → Push Notifications → Upload FCM V1 Service Account Key
```

> [!CAUTION]
> Without this step, you will see this error:
> `"Unable to retrieve the FCM server key for the recipient's app"`
> The `google-services.json` is for the CLIENT. The FCM V1 key is for EXPO'S SERVERS.

---

## 4. Frontend Implementation (React Native)

### Step 4.1 — Create NotificationManager.js

Create `src/utils/NotificationManager.js`:

```javascript
import * as Notifications from 'expo-notifications';
import * as Device from 'expo-device';
import { Platform } from 'react-native';
import Constants from 'expo-constants';
import apiClient from '../api/client'; // Your axios instance

// Configure foreground notification behavior
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldPlaySound: true,
    shouldSetBadge: true,
  }),
});

class NotificationManager {
  /**
   * Register for push notifications and return the Expo Push Token
   */
  async registerForPushNotificationsAsync() {
    let token;

    // Push notifications only work on physical devices
    if (!Device.isDevice) {
      console.warn('[Notifications] Physical device required.');
      return null;
    }

    try {
      // Step 1: Check & Request Permissions
      const { status: existingStatus } = await Notifications.getPermissionsAsync();
      let finalStatus = existingStatus;

      if (existingStatus !== 'granted') {
        const { status } = await Notifications.requestPermissionsAsync();
        finalStatus = status;
      }

      if (finalStatus !== 'granted') {
        console.warn('[Notifications] Permission denied!');
        return null;
      }

      // Step 2: Setup Android Notification Channel (BEFORE getting token)
      if (Platform.OS === 'android') {
        await Notifications.setNotificationChannelAsync('default', {
          name: 'Default',
          importance: Notifications.AndroidImportance.MAX,
          vibrationPattern: [0, 250, 250, 250],
          lightColor: '#D0E9BC',
          sound: 'default',
        });
      }

      // Step 3: Resolve EAS Project ID
      const projectId =
        Constants?.expoConfig?.extra?.eas?.projectId ??
        Constants?.easConfig?.projectId;

      if (!projectId) {
        console.error('[Notifications] EAS Project ID not found!');
        return null;
      }

      // Step 4: Get the Expo Push Token
      const tokenResponse = await Notifications.getExpoPushTokenAsync({
        projectId: projectId,
      });
      token = tokenResponse.data;

      console.log('[Notifications] Token:', token);
    } catch (error) {
      console.error('[Notifications] Registration failed:', error.message);
      return null;
    }

    return token;
  }

  /**
   * Send the token to your backend for storage
   */
  async syncTokenWithBackend(token) {
    if (!token) return false;

    try {
      await apiClient.put('/auth/push-token', { pushToken: token });
      console.log('[Notifications] Token synced to backend.');
      return true;
    } catch (error) {
      console.error('[Notifications] Sync failed:', error.message);
      return false;
    }
  }

  /**
   * Setup listeners for foreground and tap events
   */
  setupListeners(navigation) {
    // Foreground: notification received while app is open
    const foregroundSub = Notifications.addNotificationReceivedListener(
      (notification) => {
        console.log('[Notifications] Foreground:', notification);
      }
    );

    // Tap: user tapped the notification
    const responseSub = Notifications.addNotificationResponseReceivedListener(
      (response) => {
        const { data } = response.notification.request.content;

        // Deep-link to the correct screen
        if (data?.screen) {
          const nav = navigation?.current || navigation;
          if (nav && (typeof nav.isReady !== 'function' || nav.isReady())) {
            nav.navigate(data.screen, data.params);
          }
        }
      }
    );

    // Return cleanup function
    return () => {
      foregroundSub.remove();
      responseSub.remove();
    };
  }
}

export default new NotificationManager();
```

### Step 4.2 — Integrate in AppNavigator.jsx

```javascript
import React, { useEffect } from 'react';
import { NavigationContainer, createNavigationContainerRef } from '@react-navigation/native';
import { useSelector } from 'react-redux';
import NotificationManager from '../utils/NotificationManager';

const navigationRef = createNavigationContainerRef();

const AppNavigator = () => {
  const { isAuthenticated } = useSelector((state) => state.auth);

  // Register for push notifications when user logs in
  useEffect(() => {
    if (isAuthenticated) {
      const setup = async () => {
        const token = await NotificationManager.registerForPushNotificationsAsync();
        if (token) {
          await NotificationManager.syncTokenWithBackend(token);
        }
      };

      setup();

      // Setup deep-link listeners
      const cleanup = NotificationManager.setupListeners(navigationRef);
      return () => cleanup();
    }
  }, [isAuthenticated]);

  return (
    <NavigationContainer ref={navigationRef}>
      {/* ... your Stack.Navigator screens ... */}
    </NavigationContainer>
  );
};
```

### Step 4.3 — Add pushToken to Redux State (authSlice.js)

In your Redux auth slice, add two things:

**Initial State:**
```javascript
const initialState = {
  // ... existing fields ...
  pushToken: null,
};
```

**No additional reducers needed** — the token is managed by `NotificationManager` and stored on the backend, not in Redux.

---

## 5. Backend Implementation (Node.js/Express)

### Step 5.1 — Add pushToken to User Model

```javascript
// models/User.js
const UserSchema = new mongoose.Schema({
  // ... existing fields ...
  pushToken: {
    type: String,
    default: null
  },
});
```

### Step 5.2 — Create Push Token Route

**Controller (controllers/auth.js):**
```javascript
exports.updatePushToken = async (req, res, next) => {
  try {
    const user = await User.findByIdAndUpdate(
      req.user.id,
      { pushToken: req.body.pushToken },
      { returnDocument: 'after', runValidators: true }
    );

    res.status(200).json({ success: true, data: user });
  } catch (err) {
    next(err);
  }
};
```

**Route (routes/auth.js):**
```javascript
const { updatePushToken } = require('../controllers/auth');
router.put('/push-token', protect, updatePushToken);
```

### Step 5.3 — Create Push Notification Service

Create `utils/pushNotifications.js`:

```javascript
const { Expo } = require('expo-server-sdk');

let expo = new Expo();

/**
 * Send a push notification to a specific user
 * @param {string} pushToken - Expo push token
 * @param {string} title - Notification title
 * @param {string} body - Notification body text
 * @param {object} data - Payload for deep linking
 */
exports.sendPushNotification = async (pushToken, title, body, data = {}) => {
  // Validate token format
  if (!Expo.isExpoPushToken(pushToken)) {
    console.error(`Invalid push token: ${pushToken}`);
    return;
  }

  const message = {
    to: pushToken,
    sound: 'default',
    title,
    body,
    data,
    priority: 'high',
    channelId: 'default',  // Must match Android channel ID
  };

  try {
    const ticket = await expo.sendPushNotificationsAsync([message]);
    console.log('[PushService] Ticket:', ticket);
    return ticket;
  } catch (error) {
    console.error('[PushService] Error:', error);
  }
};
```

### Step 5.4 — Trigger on Events (e.g., Appointment Confirmed)

```javascript
// controllers/appointments.js
const { sendPushNotification } = require('../utils/pushNotifications');

exports.updateAppointment = async (req, res, next) => {
  try {
    let appointment = await Appointment.findById(req.params.id);
    const oldStatus = appointment.status;

    appointment = await Appointment.findByIdAndUpdate(
      req.params.id, req.body,
      { returnDocument: 'after', runValidators: true }
    ).populate('doctor', 'name');

    // Send notification when status changes to "confirmed"
    if (req.body.status === 'confirmed' && oldStatus !== 'confirmed') {
      const patient = await User.findById(appointment.patient);
      if (patient?.pushToken) {
        await sendPushNotification(
          patient.pushToken,
          'Appointment Confirmed 🗓️',
          `Your appointment with Dr. ${appointment.doctor?.name} has been confirmed.`,
          { screen: 'Appointments' }  // For deep-linking
        );
      }
    }

    res.status(200).json({ success: true, data: appointment });
  } catch (err) {
    next(err);
  }
};
```

---

## 6. Scheduled Notifications (Cron Jobs)

### Create Scheduler Service

Create `services/scheduler.js`:

```javascript
const cron = require('node-cron');
const Appointment = require('../models/Appointment');
const Medication = require('../models/Medication');
const { sendPushNotification } = require('../utils/pushNotifications');

exports.initScheduler = () => {
  console.log('[Scheduler] Initialized.');

  // Run every minute
  cron.schedule('* * * * *', async () => {
    const now = new Date();
    const HH_MM = `${now.getHours().toString().padStart(2, '0')}:${now.getMinutes().toString().padStart(2, '0')}`;

    try {
      // 1. Appointment Reminders (15 min before)
      await checkAppointmentReminders(now);

      // 2. Medication Reminders (exact time match)
      await checkMedicationReminders(HH_MM);
    } catch (error) {
      console.error('[Scheduler] Error:', error);
    }
  });
};

async function checkAppointmentReminders(now) {
  const target = new Date(now.getTime() + 15 * 60000);
  const windowStart = new Date(target); windowStart.setSeconds(0, 0);
  const windowEnd = new Date(target); windowEnd.setSeconds(59, 999);

  const appointments = await Appointment.find({
    appointmentDate: { $gte: windowStart, $lte: windowEnd },
    status: 'confirmed'
  }).populate('doctor patient', 'name pushToken');

  for (const apt of appointments) {
    if (apt.patient?.pushToken) {
      await sendPushNotification(
        apt.patient.pushToken,
        'Appointment Reminder 🔔',
        `Your consultation with Dr. ${apt.doctor?.name} starts in 15 minutes.`,
        { screen: 'Appointments' }
      );
    }
  }
}

async function checkMedicationReminders(currentTime) {
  const medications = await Medication.find({
    status: 'active',
    reminderTimes: currentTime  // e.g., ["08:00", "20:00"]
  }).populate('patient', 'pushToken');

  for (const med of medications) {
    if (med.patient?.pushToken) {
      await sendPushNotification(
        med.patient.pushToken,
        'Medication Time 💊',
        `It's time to take your ${med.name} (${med.dosage}).`,
        { screen: 'Medications' }
      );
    }
  }
}
```

### Initialize in server.js

```javascript
const { initScheduler } = require('./services/scheduler');

// After connectDB()
connectDB();
initScheduler();  // Start the cron scheduler
```

---

## 7. Deep Linking from Notifications

When a user taps a notification, the `data` payload determines which screen to open:

```javascript
// Sending (backend):
await sendPushNotification(token, title, body, {
  screen: 'Appointments',       // Screen name in your navigator
  params: { id: '12345' }       // Optional navigation params
});

// Receiving (frontend - in NotificationManager.setupListeners):
const { data } = response.notification.request.content;
if (data?.screen) {
  navigation.navigate(data.screen, data.params);
}
```

> [!TIP]
> Use `createNavigationContainerRef()` at module level and pass `ref={navigationRef}` to `NavigationContainer`. This ensures deep-linking works even when the app is opened from a killed state.

---

## 8. EAS Build & APK Configuration

### eas.json

```json
{
  "build": {
    "preview": {
      "distribution": "internal"
    },
    "production": {
      "autoIncrement": true
    }
  }
}
```

### Build Commands

```bash
# Development APK (for testing)
eas build --platform android --profile preview

# Production AAB (for Play Store)
eas build --platform android --profile production
```

---

## 9. Testing & Debugging

### Test 1: Verify Token Registration
After login, check your backend logs:
```
[Notifications] Token: ExponentPushToken[abc123...]
[Notifications] Token synced to backend.
```
Verify in MongoDB that the user has `pushToken` field populated.

### Test 2: Send Test Notification
Use Expo's push notification tool:
1. Go to [expo.dev/notifications](https://expo.dev/notifications)
2. Paste your `ExponentPushToken[...]`
3. Fill in title/body → Send
4. Check your physical device

### Test 3: Verify Scheduled Reminders
1. Add a medication with `reminderTimes: ["HH:MM"]` (2 minutes from now)
2. Watch backend logs for:
   ```
   [Scheduler] Checking reminders for HH:MM...
   [PushService] Ticket: [{ status: 'ok', id: '...' }]
   ```
3. Notification should appear on device

### Test 4: Verify Deep Linking
1. Receive a notification
2. Tap it
3. App should open and navigate to the specified screen

---

## 10. Common Errors & Solutions

### Error 1: `"Unable to retrieve the FCM server key"`
```
status: 'error'
message: "Unable to retrieve the FCM server key for the recipient's app"
details: { error: 'InvalidCredentials' }
```
**Cause**: FCM V1 credentials not uploaded to Expo  
**Fix**: Firebase Console → Service Accounts → Generate Key → Upload to expo.dev → Credentials → Android → FCM V1

---

### Error 2: `"ExponentPushToken is not valid"`
**Cause**: Running on emulator/simulator  
**Fix**: Push notifications require a physical device. `Device.isDevice` returns `false` on emulators.

---

### Error 3: `"EAS Project ID not found"`
**Cause**: Missing `projectId` in `app.json`  
**Fix**: Add to `app.json`:
```json
"extra": {
  "eas": {
    "projectId": "your-project-id"
  }
}
```
Get it by running `eas init` or from expo.dev dashboard.

---

### Error 4: `"Failed to resolve plugin for expo-notifications"`
**Cause**: Plugin listed in `app.json` but package not installed  
**Fix**: `npx expo install expo-notifications`

---

### Error 5: Notifications work in Expo Go but not in APK
**Cause**: Missing `google-services.json`  
**Fix**: Download from Firebase Console → Place in project root → Reference in `app.json`:
```json
"android": {
  "googleServicesFile": "./google-services.json"
}
```

---

### Error 6: Notification received but no sound/vibration
**Cause**: Android notification channel not configured  
**Fix**: Create channel BEFORE requesting token:
```javascript
await Notifications.setNotificationChannelAsync('default', {
  name: 'Default',
  importance: Notifications.AndroidImportance.MAX,
  sound: 'default',
});
```

---

## 11. Production Checklist

Before shipping to production, verify every item:

- [ ] `expo-notifications`, `expo-device`, `expo-constants` installed
- [ ] `expo-notifications` registered as plugin in `app.json`
- [ ] `google-services.json` placed in project root
- [ ] `googleServicesFile` path set in `app.json` → `android`
- [ ] EAS `projectId` set in `app.json` → `extra.eas`
- [ ] FCM V1 Service Account Key uploaded to expo.dev
- [ ] `User` model has `pushToken: String` field
- [ ] `PUT /auth/push-token` route exists and is protected
- [ ] `expo-server-sdk` installed on backend
- [ ] `sendPushNotification()` utility created
- [ ] `channelId: 'default'` matches Android channel name
- [ ] Notification triggers added to relevant controllers
- [ ] Scheduler initialized in `server.js`
- [ ] Deep-link listeners setup with `navigationRef`
- [ ] Tested on physical device (not emulator)
- [ ] Verified `status: 'ok'` in push ticket response

---

## File Structure Reference

```
Frontend (React Native):
├── app.json                          ← Plugin config + EAS ID
├── google-services.json              ← Firebase client config
├── eas.json                          ← Build profiles
└── src/
    ├── utils/
    │   └── NotificationManager.js    ← Token + Listeners
    ├── navigation/
    │   └── AppNavigator.jsx          ← Registration trigger
    └── store/slices/
        └── authSlice.js              ← Auth state

Backend (Node.js):
├── models/
│   └── User.js                       ← pushToken field
├── controllers/
│   └── auth.js                       ← updatePushToken
├── routes/
│   └── auth.js                       ← PUT /push-token
├── utils/
│   └── pushNotifications.js          ← Expo Server SDK
├── services/
│   └── scheduler.js                  ← Cron jobs
└── server.js                         ← initScheduler()
```

---

> **Document Version**: 1.0  
> **Tested With**: Expo SDK 55, React Native 0.83, Node.js 22  
> **Project**: OneCare AI — Healthcare Platform
