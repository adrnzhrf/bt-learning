# Firebase App Distribution APK Production Deployment

## Overview

This guide covers deploying Android APK builds to Firebase App Distribution for the Bateriku Customer Mobile app in the **production environment**. Production deployments require additional verification steps and approval processes.

## Prerequisites

- **Firebase CLI** installed globally (`npm install -g firebase-tools`)
- Firebase project with App Distribution enabled
- Authenticated Firebase CLI (`firebase login`)
- Signing keys configured for Android production builds
- **Authorization:** Access to the production Firebase project (bateriku-customer-v2-prod)
- **Approval:** Release approval from the product/release team

## Production Environment Configuration

**Production Environment:**
- Project ID: `bateriku-customer-v2-prod`
- Application ID: `com.bateriku.app`
- Configuration stored in: `firebase.json` and `android/app/src/prod/google-services.json`

> **Important:** Ensure you are authenticated with the correct Firebase account that has permissions for the production project.

### Verify Firebase Authentication

```bash
firebase login
# Follow the browser prompts with your production account
```

Verify you have access to the production project:
```bash
firebase projects:list
# Should show: bateriku-customer-v2-prod
```

## Pre-Deployment Checklist

Before deploying to production, complete the following:

- [ ] Version number updated in `pubspec.yaml` (follow semantic versioning)
- [ ] Release notes prepared and documented
- [ ] All feature tests passed in staging environment
- [ ] Release approval obtained from product/release team
- [ ] Changelog updated (if applicable)
- [ ] Firebase project credentials verified for production
- [ ] Android signing keys available and verified

## Build Production APK

### 1. Clean and Prepare

```bash
flutter clean
flutter pub get
```

### 2. Build Release APK

```bash
flutter build apk --flavor prod -t lib/main_prod.dart --release
```

**Output Location:** `build/app/outputs/flutter-apk/app-prod-release.apk`

### 3. Verify Build

```bash
# Check APK file exists and size is reasonable
ls -lh build/app/outputs/flutter-apk/app-prod-release.apk
```

## Deploy to Firebase App Distribution

### Standard Production Deployment

```bash
firebase appdistribution:distribute build/app/outputs/flutter-apk/app-prod-release.apk \
  --app=bateriku-customer-v2-prod:android:com.bateriku.app \
  --release-notes="Production Release v[VERSION]: [RELEASE_NOTES]"
```

Replace `[VERSION]` with the actual version (e.g., `1.0.0`) and `[RELEASE_NOTES]` with your release notes.

### Example: Production Deployment with Release Notes

```bash
firebase appdistribution:distribute build/app/outputs/flutter-apk/app-prod-release.apk \
  --app=bateriku-customer-v2-prod:android:com.bateriku.app \
  --release-notes="Production Release v1.0.0: 
- Feature A: Description
- Feature B: Description
- Bug fixes: Description"
```

### Deployment with Testers/Groups

For phased rollout or internal tester distribution:

```bash
firebase appdistribution:distribute build/app/outputs/flutter-apk/app-prod-release.apk \
  --app=bateriku-customer-v2-prod:android:com.bateriku.app \
  --release-notes="Production Release v1.0.0: [RELEASE_NOTES]" \
  --testers-group="production-testers"
```

## Accessing Production Releases

After successful deployment, access the release through:

### 1. Firebase Console
- Project: https://console.firebase.google.com/project/bateriku-customer-v2-prod
- Navigate to: App Distribution → Android → Releases

### 2. Release Verification
- Confirm the version number matches what was built
- Verify release notes are displayed correctly
- Check that all testers have access (if applicable)

### 3. Tester Distribution Link
- Share with approved testers at: https://appdistribution.firebase.google.com/testerapps

## Production Release Notes Template

```
Version: [X.Y.Z]
Release Date: [YYYY-MM-DD]

### New Features
- Feature 1: Description
- Feature 2: Description

### Improvements
- Improvement 1: Description
- Improvement 2: Description

### Bug Fixes
- Bug fix 1: Description
- Bug fix 2: Description

### Known Issues
- Known issue 1 (if any)

### Upgrade Notes
- Breaking changes (if any)
- Required actions from users (if any)
```

## Post-Deployment Steps

### 1. Notification
- Notify the release team and stakeholders
- Share the deployment link with approved users/testers
- Post announcements to relevant channels

### 2. Monitoring
- Monitor crash reports in Firebase Crashlytics
- Track user feedback and early adoption issues
- Monitor performance metrics in Firebase Performance Monitoring

### 3. Documentation
- Update release notes in the app store listing
- Document any deployment issues for future reference
- Archive release information for compliance

## Rollback Procedure

If critical issues are discovered after production deployment:

1. **Immediate Action:** Notify the release team immediately
2. **Halt Distribution:** Stop sharing the release link
3. **Issue Assessment:** Determine if rollback is necessary
4. **Previous Version:** Deploy the previous stable version
   ```bash
   # Build and deploy previous version tagged in your repository
   git checkout [PREVIOUS_TAG]
   flutter clean && flutter pub get
   flutter build apk --flavor prod -t lib/main_prod.dart --release
   firebase appdistribution:distribute build/app/outputs/flutter-apk/app-prod-release.apk \
     --app=bateriku-customer-v2-prod:android:com.bateriku.app \
     --release-notes="Hotfix Rollback: Reverting to stable version"
   ```
5. **Root Cause Analysis:** Document what went wrong for prevention

## Troubleshooting

### Firebase CLI Not Found
```bash
npm install -g firebase-tools
```

### Authentication Issues
```bash
firebase logout
firebase login
# Ensure you're logged in with the production account
```

### Wrong Project Error
```bash
# Verify you're using the correct project
firebase projects:list
# If needed, use --project flag
firebase appdistribution:distribute build/app/outputs/flutter-apk/app-prod-release.apk \
  --app=bateriku-customer-v2-prod:android:com.bateriku.app \
  --project=bateriku-customer-v2-prod \
  --release-notes="Release notes"
```

### APK Build Failures
- Ensure production signing keys in `android/key.properties` are correct
- Check all dependencies are installed: `flutter pub get`
- Verify Flutter is up to date: `flutter upgrade`
- Confirm flavor is set to `prod` in the build command

### App Distribution Errors
- Verify the app ID matches the production configuration
- Ensure production Firebase project has App Distribution enabled
- Check you have appropriate permissions in the Firebase project
- Confirm the APK file is signed correctly for production

## Automated Production Deployment (GitHub Actions)

For automated deployments on version tags or manual triggers, configure GitHub Actions with:

**Required Secrets:**
- `FIREBASE_PROJECT_ID_PROD`: Typically `bateriku-customer-v2-prod`
- `FIREBASE_CLI_TOKEN`: Service account token for Firebase CLI (production)
- `ANDROID_SIGNING_KEY_PROD`: Android signing key for production
- `ANDROID_KEY_PASSWORD_PROD`: Android key password
- `ANDROID_STORE_PASSWORD_PROD`: Android store password

**Trigger:** Tag releases as `prod-v[X.Y.Z]`

## Production Deployment Approval Policy

- **Code Review:** Minimum 2 approvals required
- **Testing:** Verified in staging environment
- **Release Notes:** Documented and reviewed
- **Team Approval:** Release manager or product lead approval
- **Documentation:** Updated and archived

## Related Documentation

- [Firebase App Distribution Docs](https://firebase.google.com/docs/app-distribution)
- [Flutter Release Build Guide](https://flutter.dev/docs/deployment/android#release-signing)
- [Staging Deployment Guide](./Firebase_App_Distribution_APK_Deployment.md)
- [CI/CD Setup Guide](./CI_CD_SETUP.md)
- [iOS TestFlight Production Deployment](./DEPLOYMENT_TO_IOS_TESTFLIGHT_PRODUCTION.md)

## Support & Escalation

For deployment issues or support:

1. Check the troubleshooting section above
2. Review Firebase Console logs and error messages
3. Consult the CI/CD setup documentation
4. Escalate to the DevOps/infrastructure team if needed
