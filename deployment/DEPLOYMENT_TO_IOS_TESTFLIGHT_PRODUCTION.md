# TestFlight Deployment Guide (Production)

This guide provides step-by-step instructions for deploying the Bateriku Customer Mobile production app to TestFlight.

⚠️ **WARNING**: This document is for **PRODUCTION** deployment. Exercise extreme caution and ensure thorough testing before proceeding. Production builds should follow your organization's release approval process.

## Prerequisites

- **Apple Developer Account** with team membership
- **Team ID**: `JDZG97B6AT`
- **Bundle ID**: `com.bateriku.customer`
- **Xcode** installed and signed in with your Apple ID
- **iOS Distribution Certificate** (required for app-store exports)
- **App-specific password** for your Apple ID account
- **Flutter SDK** (>=3.24.0)
- **Ruby 3.x** (CocoaPods requires Ruby 3.x; system Ruby 2.6 is incompatible)
- **CocoaPods** installed and working

⚠️ **Important**: Ensure Ruby 3.x is installed. The system Ruby 2.6 will cause CocoaPods to fail.

## Step 1: Prepare Your Machine

### 1a. Set Up Ruby 3.x

CocoaPods requires Ruby 3.x. If you have Homebrew installed:

```bash
brew install ruby
```

Then set up your shell environment:

```bash
# Add this to your ~/.zshrc or ~/.bashrc
export PATH="/opt/homebrew/opt/ruby/bin:/Users/$USER/.local/share/gem/ruby/3.4.0/bin:$PATH"

# Reload your shell
source ~/.zshrc
```

### 1b. Verify Account Setup

Ensure you have the iOS certificates installed:

1. Open Xcode: `open /Applications/Xcode.app`
2. Go to **Xcode → Settings → Accounts**
3. Select your Apple ID
4. Click **Manage Certificates**
5. Verify you have both:
   - ✅ Apple Development certificate
   - ✅ Apple Distribution certificate (required for TestFlight)

If the Distribution certificate is missing, download it from the [Apple Developer Portal](https://developer.apple.com/account/resources/certificates/list).

## Step 2: Generate App-Specific Password

TestFlight requires an app-specific password (not your regular Apple ID password):

1. Go to [https://account.apple.com](https://account.apple.com)
2. Sign in with your Apple ID
3. Navigate to **Security** section
4. Under "App-specific passwords", click **Generate Password**
5. Select **"Xcode"** as the app type
6. Copy the generated password (format: `xxxx-xxxx-xxxx-xxxx`)
7. Store it securely - you'll need it in Step 7

⚠️ **Important**: Never commit this password to version control.

## Step 3: Pre-Release Checklist

Before building a production release, ensure:

- ✅ Version number is incremented in `pubspec.yaml`
- ✅ Build number is incremented in `ios/Runner.xcodeproj/project.pbxproj`
- ✅ All tests pass: `flutter test`
- ✅ Code review completed and approved
- ✅ Release notes prepared
- ✅ Release approval obtained from product/management team
- ✅ Feature flags configured for production environment

## Step 4: Build Flutter Release

Build the Flutter app for production flavor in release mode:

```bash
# Set up Ruby path first
export PATH="/opt/homebrew/opt/ruby/bin:/Users/$USER/.local/share/gem/ruby/3.4.0/bin:$PATH"

cd /Users/adrinzharif/Projects/customer-mobile
flutter clean
flutter pub get
flutter build ios --flavor prod -t lib/main_prod.dart --release
```

Expected output: `Built build/ios/iphoneos/Runner.app` (~57.7 MB)

✅ **Success Indicator**: The build completes without CocoaPods errors.

## Step 5: Create Xcode Archive

Generate an `.xcarchive` file for distribution:

```bash
# Set up Ruby path
export PATH="/opt/homebrew/opt/ruby/bin:/Users/$USER/.local/share/gem/ruby/3.4.0/bin:$PATH"

xcodebuild -workspace ios/Runner.xcworkspace \
  -scheme Runner \
  -configuration Release-prod \
  -derivedDataPath build/ios \
  -archivePath build/ios/Bateriku-PROD.xcarchive \
  -allowProvisioningUpdates \
  archive
```

⚠️ **Critical**: The `-allowProvisioningUpdates` flag is required to automatically sync provisioning profiles. Without it, the archive will fail with "No profiles found" error.

Expected output: `** ARCHIVE SUCCEEDED **`

## Step 6: Update ExportOptions.plist

Update the export options to use app-store method with your team ID:

```bash
cat > ios/ExportOptions.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>method</key>
	<string>app-store</string>
	<key>signingStyle</key>
	<string>automatic</string>
	<key>stripSwiftSymbols</key>
	<true/>
	<key>teamID</key>
	<string>JDZG97B6AT</string>
</dict>
</plist>
EOF
```

## Step 7: Export Archive to IPA

Convert the `.xcarchive` to an IPA file suitable for TestFlight distribution:

```bash
# Set up Ruby path
export PATH="/opt/homebrew/opt/ruby/bin:/Users/$USER/.local/share/gem/ruby/3.4.0/bin:$PATH"

xcodebuild -exportArchive \
  -archivePath build/ios/Bateriku-PROD.xcarchive \
  -exportPath build/ios/export \
  -exportOptionsPlist ios/ExportOptions.plist \
  -allowProvisioningUpdates
```

Expected output: `** EXPORT SUCCEEDED **`

File created: `build/ios/export/Bateriku.ipa` (approximately 34-35 MB)

## Step 8: Upload to TestFlight

Submit the IPA to TestFlight using the app-specific password:

```bash
xcrun altool --upload-app \
  --file "build/ios/export/Bateriku.ipa" \
  --type ios \
  --bundle-id com.bateriku.customer \
  --username adrin.zharif@bateriku.com \
  --password YOUR_APP_SPECIFIC_PASSWORD
```

Replace `YOUR_APP_SPECIFIC_PASSWORD` with the password from Step 2.

### Expected Success Response:

```
2026-XX-XX XX:XX:XX.XXX  INFO: [ContentDelivery.Uploader.XXXXX]
==========================================
UPLOAD SUCCEEDED with no errors
Delivery UUID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Transferred 35892218 bytes in 5.683 seconds (6.3MB/s, 50.526Mbps)
==========================================
No errors uploading archive at 'build/ios/export/Bateriku.ipa'.
```

## Step 9: Monitor TestFlight Processing

1. Go to [https://testflight.apple.com](https://testflight.apple.com)
2. Sign in with your Apple ID
3. Navigate to the **Bateriku** app
4. Watch for processing status (typically 5-30 minutes)
5. Once processed, review build details and prepare release notes
6. Invite internal testers first (QA team) for final validation
7. Submit to external testers after internal validation passes

## Step 10: Prepare for App Store Submission

Once TestFlight testing is complete and approved:

1. Go to [https://appstoreconnect.apple.com](https://appstoreconnect.apple.com)
2. Navigate to **Bateriku** → **TestFlight**
3. Review the build details and test feedback
4. Follow App Store Connect's submission flow to push to production App Store

## One-Line Deployment (Production)

⚠️ **WARNING**: Only run this command after all pre-release checks are complete and you have explicit approval.

```bash
export PATH="/opt/homebrew/opt/ruby/bin:/Users/$USER/.local/share/gem/ruby/3.4.0/bin:$PATH" && \
cd /Users/adrinzharif/Projects/customer-mobile && \
flutter clean && \
flutter pub get && \
flutter build ios --flavor prod -t lib/main_prod.dart --release && \
xcodebuild -workspace ios/Runner.xcworkspace -scheme Runner -configuration Release-prod -derivedDataPath build/ios -archivePath build/ios/Bateriku-PROD.xcarchive -allowProvisioningUpdates archive && \
xcodebuild -exportArchive -archivePath build/ios/Bateriku-PROD.xcarchive -exportPath build/ios/export -exportOptionsPlist ios/ExportOptions.plist -allowProvisioningUpdates && \
xcrun altool --upload-app --file "build/ios/export/Bateriku.ipa" --type ios --bundle-id com.bateriku.customer --username adrin.zharif@bateriku.com --password YOUR_APP_SPECIFIC_PASSWORD
```

⚠️ **IMPORTANT**: Replace `YOUR_APP_SPECIFIC_PASSWORD` with the actual password from Step 2.

## Troubleshooting

### Error: "CocoaPods is installed but broken"

**Cause**: System Ruby 2.6 is incompatible with modern CocoaPods. CocoaPods requires Ruby 3.x.

**Solution**:
1. Install Ruby 3.x: `brew install ruby`
2. Add Ruby 3.x to your PATH in `~/.zshrc` or `~/.bashrc`:
   ```bash
   export PATH="/opt/homebrew/opt/ruby/bin:/Users/$USER/.local/share/gem/ruby/3.4.0/bin:$PATH"
   ```
3. Reinstall CocoaPods: `gem install cocoapods`
4. Reload your shell: `source ~/.zshrc`
5. Retry the build

### Error: "No profiles for 'com.bateriku.customer' were found"

**Cause**: Xcode couldn't find or create provisioning profiles.

**Solution**:
1. Add the `-allowProvisioningUpdates` flag to your xcodebuild commands (Step 5 and Step 7)
2. Ensure your Apple ID is signed into Xcode
3. Go to **Xcode → Settings → Accounts** and verify your account
4. Click **Download All** to sync profiles manually

### Error: "Revoke certificate: Your account already has an Apple Development signing certificate"

**Cause**: The certificate's private key is not installed in your keychain.

**Solution**:
1. Download the certificate from [Apple Developer Portal](https://developer.apple.com/account/resources/certificates/list)
2. Double-click to install it in Keychain
3. Verify it appears in **Xcode → Settings → Accounts → Manage Certificates**
4. Retry the build with `-allowProvisioningUpdates` flag

### Error: "Please sign in with an app-specific password"

**Cause**: Using regular Apple ID password instead of app-specific password.

**Solution**:
1. Generate a new app-specific password at [https://account.apple.com](https://account.apple.com)
2. Use the generated password in the upload command

### Error: "Invalid bundle identifier"

**Cause**: Bundle ID doesn't match what's registered on Apple Developer Portal.

**Current bundle ID**: `com.bateriku.customer`

**Solution**: Verify the bundle ID matches the one registered in App Store Connect.

## Configuration Reference

| Setting | Value |
|---------|-------|
| **Bundle ID** | `com.bateriku.customer` |
| **Team ID** | `JDZG97B6AT` |
| **Configuration** | `Release-prod` |
| **Flavor** | `prod` |
| **Main Entry** | `lib/main_prod.dart` |
| **Export Method** | `app-store` |
| **Minimum iOS Version** | 13.0 |
| **IPA Location** | `build/ios/export/Bateriku.ipa` |
| **Apple ID** | `adrin.zharif@bateriku.com` |
| **Project Path** | `/Users/adrinzharif/Projects/customer-mobile` |

## Security Best Practices

- 🔒 **Never commit credentials** to version control
- 🔑 **Store app-specific password** securely (use environment variables or secure vaults)
- ✅ **Use environment variables** for automated deployments:

```bash
export APPLE_ID=adrin.zharif@bateriku.com
export APPLE_APP_PASSWORD=your_app_specific_password

xcrun altool --upload-app \
  --file "build/ios/export/Bateriku.ipa" \
  --type ios \
  --bundle-id com.bateriku.customer \
  --username $APPLE_ID \
  --password $APPLE_APP_PASSWORD
```

- 🔐 **For CI/CD pipelines**, use secure secret management (GitHub Secrets, GitLab CI/CD Variables, etc.)
- 📝 **Log all production deployments** for compliance and debugging
- ✔️ **Always use signed commits** for production releases

## Production Release Checklist

Before submitting to App Store from TestFlight:

- [ ] Internal testing completed successfully
- [ ] All critical bugs fixed and verified
- [ ] Performance metrics acceptable
- [ ] Crash rate acceptable
- [ ] Release notes finalized and reviewed
- [ ] Marketing team notified
- [ ] Customer support team briefed
- [ ] Rollback plan documented
- [ ] Deployment time scheduled (avoid peak hours)
- [ ] Stakeholder sign-off obtained

## Rollback Procedure

If critical issues are discovered after production deployment:

1. **Immediate**: Notify team leads and stakeholders
2. **Assessment**: Determine severity and impact
3. **Decision**: Decide on hotfix vs. rollback
4. **Action**: Either:
   - Build and deploy a hotfix version, OR
   - Pause the release and revert to previous stable version in App Store
5. **Communication**: Update support team and customer communication
6. **Post-mortem**: Document root cause and prevention measures

## Additional Resources

- [Apple TestFlight Documentation](https://developer.apple.com/testflight/)
- [App Store Connect Help](https://help.apple.com/app-store-connect/)
- [Xcode Build System](https://developer.apple.com/documentation/xcode/building-your-app)
- [Flutter iOS Deployment](https://docs.flutter.dev/deployment/ios)
- [App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)

## Support

For issues or questions about the production deployment process, refer to:
- Apple Developer Forum: [https://developer.apple.com/forums/](https://developer.apple.com/forums/)
- Flutter Community: [https://flutter.dev/community](https://flutter.dev/community)
- Your internal team: Check with product and DevOps leads for deployment approval
