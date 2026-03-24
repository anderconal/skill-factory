# Technical Stories: Mobile

Stories técnicas específicas para proyectos Mobile. Complementan las stories de `_base.md`.

> **Decisión previa:** Validar si la app será nativa (iOS/Android separado), híbrida (React Native/Flutter) o PWA.

## Stories

```yaml
stories:
  - ref: "mobile-skeleton"
    title: "Walking skeleton - Mobile app end-to-end"
    effort: 3
    area: "walking-skeleton"
    description: |
      As a development team,
      I want a minimal mobile app deployed to test devices,
      so that we validate the architecture and deployment process before building features.
    acceptanceCriteria:
      - "App installs on test device (iOS or Android)"
      - "App calls backend API endpoint"
      - "API returns data that app displays"
      - "Works on real device, not just simulator"
    verifiableValue: "Install app on device → see 'Hello World' from API"
    priority: "High"
    notes: "First story. Includes decision: native/hybrid/PWA"

  - ref: "mobile-responsive"
    title: "Responsive layout for devices"
    effort: 2
    area: "ux"
    description: |
      As a user,
      I want the app to work on my device size,
      so that I can use it on my phone or tablet comfortably.
    acceptanceCriteria:
      - "Layout adapts to phone and tablet sizes"
      - "Portrait and landscape orientations supported (or explicitly locked)"
      - "Touch targets minimum 44x44pt (iOS) / 48x48dp (Android)"
      - "Text scales with system accessibility settings"
    verifiableValue: "Test on phone and tablet → both work well"

  - ref: "mobile-navigation"
    title: "Navigation design"
    effort: 2
    area: "ux"
    description: |
      As a user,
      I want intuitive navigation,
      so that I can move through the app without confusion.
    acceptanceCriteria:
      - "Bottom navigation or tab bar for main sections"
      - "Back gesture/button works consistently"
      - "Deep links supported (optional for MVP)"
      - "Current section clearly indicated"
    verifiableValue: "Navigate app → always know where you are"
    dependsOn: ["mobile-skeleton"]

  - ref: "mobile-cicd"
    title: "CI/CD for app stores"
    effort: 3
    area: "ci-cd"
    description: |
      As a development team,
      I want automated builds for app stores,
      so that we can release updates quickly and users get improvements faster.
    acceptanceCriteria:
      - "Automated build creates signed APK/IPA"
      - "Builds uploaded to TestFlight (iOS) / Internal Testing (Android)"
      - "Version numbers auto-incremented"
      - "Build artifacts archived"
    verifiableValue: "Push to main → new build available in TestFlight/Internal Testing"
    notes: "Production store submission may remain manual initially"

  - ref: "mobile-telemetry"
    title: "Crash reporting and analytics"
    effort: 1
    area: "observability"
    description: |
      As a development team,
      I want crash reports from user devices,
      so that we can fix issues users experience in the wild.
    acceptanceCriteria:
      - "Crashlytics/Sentry/AppCenter integrated"
      - "Crashes include device info, OS version, stack trace"
      - "Non-fatal errors also captured"
      - "Team alerted on crash spikes"
    verifiableValue: "Force crash on device → appears in dashboard within minutes"

  - ref: "mobile-push"
    title: "Push notifications setup"
    effort: 2
    area: "notifications"
    description: |
      As a user,
      I want to receive important notifications,
      so that I don't miss relevant updates when I'm not in the app.
    acceptanceCriteria:
      - "FCM (Android) and APNs (iOS) configured"
      - "User can grant/deny notification permission"
      - "Backend can send push to specific user"
      - "Tapping notification opens relevant screen"
    verifiableValue: "Trigger notification from backend → appears on device"
    notes: "Content of notifications is a separate feature story"

  - ref: "mobile-i18n"
    title: "Internationalization foundation"
    effort: 3
    area: "ux"
    description: |
      As a user,
      I want the app in my language,
      so that I can understand all content.
    acceptanceCriteria:
      - "i18n library integrated"
      - "All UI strings externalized"
      - "Language follows device settings or manual override"
      - "At least 2 languages supported"
      - "RTL support if needed for target languages"
    verifiableValue: "Change device language → app UI changes"
    optional: true

  - ref: "mobile-offline"
    title: "Basic offline support"
    effort: 3
    area: "ux"
    description: |
      As a user,
      I want to see cached content when offline,
      so that I can still use the app without internet.
    acceptanceCriteria:
      - "Previously loaded data available offline"
      - "Clear indication when offline"
      - "Actions queued and synced when back online (optional)"
      - "Graceful error messages for network failures"
    verifiableValue: "Enable airplane mode → can still see cached content"
    optional: true
    notes: "Scope depends on product requirements"
```

## Platform-Specific Notes

### iOS
- Minimum iOS version: define early (affects available APIs)
- App Store review process: plan for 1-3 days
- TestFlight for beta distribution

### Android
- Minimum SDK version: define early
- Google Play internal testing for beta
- Consider app bundles (AAB) vs APK

### Hybrid (React Native / Flutter)
- Single codebase but platform-specific testing still needed
- Native modules may be needed for some features
- Performance profiling on both platforms

## Typical Epic Structure

```
Epic: Technical Infrastructure (Mobile)
├── Walking skeleton - Mobile app end-to-end
├── [from _base] Infrastructure as Code setup (backend)
├── [from _base] Basic CI/CD pipeline (backend)
├── CI/CD for app stores
├── [from _base] Basic telemetry and error tracking (backend)
├── Crash reporting and analytics (mobile)
├── [from _base] User authentication
├── [from _base] Role-based authorization
├── Responsive layout for devices
├── Navigation design
├── Push notifications setup
├── Internationalization foundation (optional)
└── Basic offline support (optional)
```
