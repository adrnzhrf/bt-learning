# Firebase App Distribution APK Deployment

## Overview

This guide covers deploying Android APK builds to Firebase App Distribution for the Bateriku Customer Mobile app.

## Prerequisites

- **Firebase CLI** installed globally (`npm install -g firebase-tools`)
- Firebase project with App Distribution enabled
- Authenticated Firebase CLI (`firebase login`)
- Signing keys configured for Android builds

## Setup

### 1. Configure Firebase Project

Your project is already configured with the following Firebase projects:

**Staging Environment:**
- Project ID: `bateriku-customer-v2-stg`
- App ID: `1:155384328120:android:9c1d4e7650ad6e4ad6e94d`
- Application ID: `com.bateriku.customer.stg`

**Production Environment:**
- Project ID: `bateriku-customer-v2-prod` (when available)
- Application ID: `com.bateriku.app`

Configuration is stored in `firebase.json` and `android/app/src/stg/google-services.json`.

### 2. Set Up Firebase CLI Authentication

```bash
firebase login
# Follow the browser prompts to authenticate with your Google account
```

Verify authentication:
```bash
firebase projects:list
```

## Deployment Process

### Build Staging APK

```bash
flutter clean
flutter pub get
flutter build apk --flavor stg -t lib/main_stg.dart --release
```

**Output Location:** `build/app/outputs/flutter-apk/app-stg-release.apk`

### Deploy to Firebase App Distribution

```bash
firebase appdistribution:distribute build/app/outputs/flutter-apk/app-stg-release.apk \
  --app=1:155384328120:android:9c1d4e7650ad6e4ad6e94d \
  --release-notes="Your release notes here"
```

### Optional: Specify Testers

To automatically add testers to the release, use:

```bash
firebase appdistribution:distribute build/app/outputs/flutter-apk/app-stg-release.apk \
  --app=1:155384328120:android:9c1d4e7650ad6e4ad6e94d \
  --release-notes="Your release notes here" \
  --testers="tester1@example.com,tester2@example.com"
```

Or add testers to a group in Firebase Console and reference the group:

```bash
firebase appdistribution:distribute build/app/outputs/flutter-apk/app-stg-release.apk \
  --app=1:155384328120:android:9c1d4e7650ad6e4ad6e94d \
  --release-notes="Your release notes here" \
  --testers-group="internal"
```

## Accessing Releases

After deployment, you can access the release through:

1. **Firebase Console:**
   - Project: https://console.firebase.google.com/project/bateriku-customer-v2-stg
   - Navigate to: App Distribution → Android → Releases

2. **Tester Link:**
   - Share with testers at: https://appdistribution.firebase.google.com/testerapps

3. **Direct Download:**
   - Download links are provided after successful deployment (valid for 1 hour)

## Example Deployment Result

```
✔ uploaded new release 0.7.3 (18) successfully!
✔ View this release in the Firebase console: https://console.firebase.google.com/project/bateriku-customer-v2-stg/appdistribution/app/android:com.bateriku.customer.stg/releases/[RELEASE_ID]
✔ Share this release with testers who have access: https://appdistribution.firebase.google.com/testerapps/1:155384328120:android:9c1d4e7650ad6e4ad6e94d/releases/[RELEASE_ID]
```

## Troubleshooting

### Firebase CLI Not Found
```bash
npm install -g firebase-tools
```

### Authentication Issues
```bash
firebase logout
firebase login
```

### APK Build Failures
- Ensure `android/key.properties` is configured (see CI_CD_SETUP.md)
- Check that all dependencies are installed: `flutter pub get`
- Verify Flutter is up to date: `flutter upgrade`

### App Distribution Errors
- Verify the app ID matches the configuration in `firebase.json`
- Ensure the Firebase project has App Distribution enabled
- Check that you have appropriate permissions in the Firebase project

## Automated Deployment (GitHub Actions)

For automated deployments on version tags, configure GitHub Actions with the following secrets:

- `FIREBASE_CLI_TOKEN`: Service account token for Firebase CLI
- Required Android signing credentials (see CI_CD_SETUP.md)

## Related Documentation

- [Firebase App Distribution Docs](https://firebase.google.com/docs/app-distribution)
- [Flutter Release Build Guide](https://flutter.dev/docs/deployment/android#release-signing)
- [CI/CD Setup Guide](./CI_CD_SETUP.md)
- [iOS TestFlight Deployment](./TESTFLIGHT_DEPLOYMENT.md)
