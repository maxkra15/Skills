# Capabilities & Entitlements Reference

> Any capability requires an `.entitlements` file. No exceptions.

## How to Add

**Step 1: Create the entitlements file** at `{AppName}/{AppName}.entitlements`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>aps-environment</key>
    <string>development</string>
</dict>
</plist>
```

**Step 2: Update project.pbxproj** — Add `CODE_SIGN_ENTITLEMENTS` to BOTH Debug and Release build configurations:
```
CODE_SIGN_ENTITLEMENTS = {AppName}/{AppName}.entitlements;
```

## Supported Capabilities

**Authentication & Identity:**
- Push Notifications: `aps-environment` = `development` or `production`
- Sign in with Apple: `com.apple.developer.applesignin` = `["Default"]`
- App Groups: `com.apple.security.application-groups` = `["group.com.example"]`
- Associated Domains: `com.apple.developer.associated-domains` = `["applinks:example.com"]`
- AutoFill Credential: `com.apple.developer.authentication-services.autofill-credential-provider` = `true`
- App Attest: `com.apple.developer.devicecheck.appattest-environment` = `production`

**Cloud & Data:**
- iCloud: `com.apple.developer.icloud-container-identifiers` = `["iCloud.com.example"]`
- Data Protection: `com.apple.developer.default-data-protection` = `NSFileProtectionComplete`
- Shared with You: `com.apple.developer.shared-with-you` = `true`
- Messages Collaboration: `com.apple.developer.shared-with-you.collaboration` = `true`

**Apple Frameworks:**
- Apple Pay: `com.apple.developer.in-app-payments` = `["merchant.com.example"]`
- HealthKit: `com.apple.developer.healthkit` = `true`
- HealthKit Recalibrate: `com.apple.developer.healthkit.recalibrate-estimates` = `true`
- HomeKit: `com.apple.developer.homekit` = `true`
- Siri: `com.apple.developer.siri` = `true`
- Wallet: `com.apple.developer.pass-type-identifiers` = `["pass.com.example"]`
- ClassKit: `com.apple.developer.ClassKit-environment` = `production`
- WeatherKit: `com.apple.developer.weatherkit` = `true`
- MusicKit: `com.apple.developer.musickit` = `true`
- ShazamKit: `com.apple.developer.shazamkit` = `true`
- Maps: `com.apple.developer.maps` = `true`
- Family Controls: `com.apple.developer.family-controls` = `true`
- Journaling Suggestions: `com.apple.developer.journaling-suggestions` = `true`

**Hardware & Sensors:**
- NFC Tag Reading: `com.apple.developer.nfc.readersession.formats` = `["NDEF"]`
- Tap to Pay: `com.apple.developer.proximity-reader.payment.acceptance` = `true`
- Wireless Accessory: `com.apple.external-accessory.wireless-configuration` = `true`
- Media Device Discovery: `com.apple.developer.media-device-discovery-extension` = `true`
- Matter Setup: `com.apple.developer.matter.allow-setup-payload` = `true`

**Networking:**
- Access WiFi Info: `com.apple.developer.networking.wifi-info` = `true`
- Hotspot Configuration: `com.apple.developer.networking.HotspotConfiguration` = `true`
- Multipath: `com.apple.developer.networking.multipath` = `true`
- Network Extensions: `com.apple.developer.networking.networkextension` = `["packet-tunnel-provider"]`
- Personal VPN: `com.apple.developer.networking.vpn.api` = `["allow-vpn"]`
- Custom Network Protocol: `com.apple.developer.networking.custom-protocol` = `["protocol"]`

**Notifications:**
- Communication Notifications: `com.apple.developer.usernotifications.communication` = `true`
- Time Sensitive Notifications: `com.apple.developer.usernotifications.time-sensitive` = `true`
- Push to Talk: `com.apple.developer.push-to-talk` = `true`

**System & Performance:**
- Extended Virtual Addressing: `com.apple.developer.kernel.extended-virtual-addressing` = `true`
- Increased Memory Limit: `com.apple.developer.kernel.increased-memory-limit` = `true`
- System Extension: `com.apple.developer.system-extension.install` = `true`
- FileProvider Testing: `com.apple.developer.fileprovider.testing-mode` = `true`
- On Demand Install: `com.apple.developer.on-demand-install-capable` = `true`

**Other:**
- Group Activities (SharePlay): `com.apple.developer.group-session` = `true`
- Fonts: `com.apple.developer.user-fonts` = `["installable-fonts"]`
- Low Latency HLS: `com.apple.developer.coremedia.hls.low-latency` = `true`
- MDM Managed Domains: `com.apple.developer.associated-domains.mdm-managed` = `true`
- Sensitive Content Analysis: `com.apple.developer.sensitivecontentanalysis.client` = `true`

## Permissions (Info.plist Usage Descriptions)

These are **separate from entitlements**. Add as `INFOPLIST_KEY_*` in `project.pbxproj` build settings.

| Permission | Key | Example Value |
|---|---|---|
| Camera | `INFOPLIST_KEY_NSCameraUsageDescription` | "Take photos for your profile" |
| Photo Library | `INFOPLIST_KEY_NSPhotoLibraryUsageDescription` | "Save and select photos" |
| Location (When In Use) | `INFOPLIST_KEY_NSLocationWhenInUseUsageDescription` | "Show nearby places" |
| Location (Always) | `INFOPLIST_KEY_NSLocationAlwaysAndWhenInUseUsageDescription` | "Track your workouts" |
| Microphone | `INFOPLIST_KEY_NSMicrophoneUsageDescription` | "Record voice messages" |
| Contacts | `INFOPLIST_KEY_NSContactsUsageDescription` | "Find friends in your contacts" |
| Calendars | `INFOPLIST_KEY_NSCalendarsUsageDescription` | "Add events to your calendar" |
| Face ID | `INFOPLIST_KEY_NSFaceIDUsageDescription` | "Unlock with Face ID" |
| Bluetooth | `INFOPLIST_KEY_NSBluetoothAlwaysUsageDescription` | "Connect to nearby devices" |
| Motion | `INFOPLIST_KEY_NSMotionUsageDescription` | "Track your steps and activity" |
| Speech Recognition | `INFOPLIST_KEY_NSSpeechRecognitionUsageDescription` | "Transcribe your voice notes" |
| Local Network | `INFOPLIST_KEY_NSLocalNetworkUsageDescription` | "Discover devices on your network" |
| Health Share | `INFOPLIST_KEY_NSHealthShareUsageDescription` | "Read your health data" |
| Health Update | `INFOPLIST_KEY_NSHealthUpdateUsageDescription` | "Save workout data" |

## Example: WeatherKit

1. Create `MyApp/MyApp.entitlements`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.developer.weatherkit</key>
    <true/>
</dict>
</plist>
```
2. Add to BOTH Debug and Release in `project.pbxproj`:
```
CODE_SIGN_ENTITLEMENTS = MyApp/MyApp.entitlements;
```
