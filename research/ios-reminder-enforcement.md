# How far an iOS app can go in forcing a reminder

Research for [#4](https://github.com/FDavalosD/canoli/issues/4). Researched 2026-11-23 against Apple developer documentation, WWDC sessions and the App Review Guidelines. Target: current iOS. The APIs below are documented as iOS 26.0+ (AlarmKit) or older; the WWDC26 iOS guide lists no changes to any of them.

Scenario: Fernando gets "prepare your clothes for tomorrow's run" and it must not go away until he has provided **Proof** (a photo).

## Short answer

No public iOS API can make a reminder impossible to dismiss. The system always leaves the Member a way out: stop the alert, swipe it away, turn off the permission in Settings, or delete the app. What an app *can* do is make ignoring the reminder **expensive and repetitive**, and record when it was ignored so **Escalation** can tell the other Member.

The strongest combination that does not need any special approval from Apple:

1. An **AlarmKit alarm** at the due time. It sounds through silent mode and Focus, and its Stop button runs our own App Intent. That intent opens the app on the Proof screen and, if no Proof has been recorded, schedules the next alarm a few minutes later. This is a nag loop the Member can only leave by giving Proof or revoking alarm permission.
2. A **Live Activity** on the Lock Screen and Dynamic Island that shows the open item until Proof is given.
3. **Time Sensitive** repeating local notifications as a backstop. They break through Focus if the Member allows it, and repeat every 60 s or more until the app removes them.
4. **Camera-only Proof** capture through `UIImagePickerController` with `.camera` source or AVFoundation, so no photo-library picker.
5. Optional hard mode: **Screen Time shielding** (FamilyControls `.individual` + ManagedSettings) that covers every other app except a short allowlist until Proof is given. This is the only thing that really gets in the way of using the phone. It needs the Family Controls entitlement, and any distribution build (TestFlight, Ad Hoc, App Store) needs Apple to approve that entitlement first.

**Critical alerts are not a realistic option.** Apple has to approve the entitlement, and it is meant for health and safety. AlarmKit already gives scheduled reminders the same break-through-silent-mode behaviour without that approval.

## Mechanism by mechanism

| Mechanism | What it can do | Requirements | Hard limits |
|---|---|---|---|
| AlarmKit | Full alarm alert at a fixed time or after a countdown; overrides Focus and silent mode; Stop/secondary buttons can run App Intents; custom sound; also on paired Watch | iOS 26+; `NSAlarmKitUsageDescription`; user authorization; widget extension if countdown is used | User can stop or snooze it; user can revoke authorization in Settings; Apple says it is "not a replacement for" critical or time-sensitive notifications; undocumented cap on number of alarms |
| Critical alerts | Notification plays sound even when muted, locked or in Do Not Disturb; custom volume | `com.apple.developer.usernotifications.critical-alerts` entitlement, issued by Apple through a request form; plus `.criticalAlert` user authorization | Apple frames it for urgent safety use (severe weather, local safety alerts); a chore app is very unlikely to qualify; the notification is still dismissable |
| Time Sensitive notifications | Delivered immediately, lights screen, breaks through Notification Summary and Focus | Time Sensitive Notifications capability (self-service); normal notification authorization | User can turn off time-sensitive interruptions per app, or all notifications; Focus pass-through is user-controlled |
| Repeating local notifications | Re-deliver on a schedule until the app removes the request | `UNTimeIntervalNotificationTrigger(repeats: true)` needs an interval of at least 60 s | 64 pending requests per app (system limit); each delivery is a swipe-away banner |
| Live Activities | Persistent Lock Screen / Dynamic Island / StandBy presence with live state; alert-style updates | Widget extension; `NSSupportsLiveActivities`; starts from foreground, a `LiveActivityIntent`, push, or a scheduled start | Max 8 h active, 12 h on Lock Screen; user can remove it at any time; user can disable Live Activities per app; 4 KB data limit |
| Screen Time shielding (FamilyControls + ManagedSettings + DeviceActivity) | Cover all apps and websites (with exceptions) or chosen apps with a custom shield; schedule when it applies via a DeviceActivity monitor extension | Family Controls capability/entitlement; `.individual` authorization with Face ID / Touch ID; distribution entitlement approval from Apple before App Store (and, per the capability table, any non-development build) | For `.individual`, the system lets the user delete the app and sign out of iCloud; user can revoke; system "doesn't guarantee" the settings govern the device; 50 app tokens per explicit shield list; too many or tightly scheduled activities throw |
| Focus interaction | No API to change the Member's Focus | none | Only AlarmKit (always), critical alerts (always) and time-sensitive (if allowed) get through |
| Camera-only capture | Proof screen that only offers live camera capture, no library | `NSCameraUsageDescription`; camera permission; Guideline 2.5.14 consent/indicator | A photo is not proof of reality: a photo of a photo or of yesterday's pile still passes; any judgement needs a human (the other Member) or on-device analysis |

## Detail and sources

### AlarmKit

- AlarmKit is iOS/iPadOS 26.0+. It supports "one-time and repeating alarms, with the option for countdown durations and snooze functionality", and provides methods to "schedule, pause, resume, and cancel alarms" ([AlarmKit](https://developer.apple.com/documentation/alarmkit)).
- An alarm "overrides both a device's focus and silent mode, if necessary" ([Scheduling an alarm with AlarmKit](https://developer.apple.com/documentation/alarmkit/scheduling-an-alarm-with-alarmkit)). WWDC25 says the same: when it fires "the alert breaks through the silent mode and the current focus" ([WWDC25 230, Wake up to the AlarmKit API](https://developer.apple.com/videos/play/wwdc2025/230/)).
- Authorization: the app calls `AlarmManager.requestAuthorization()`, or AlarmKit asks automatically on the first alarm. Without it, alarms are not scheduled. If `NSAlarmKitUsageDescription` is missing or empty, "apps can't schedule alarms with AlarmKit" ([sample](https://developer.apple.com/documentation/alarmkit/scheduling-an-alarm-with-alarmkit), [NSAlarmKitUsageDescription](https://developer.apple.com/documentation/bundleresources/information-property-list/nsalarmkitusagedescription)). People "can change their authorization status in the Settings app" at any time (WWDC25 230).
- Schedules are a fixed time with `.never` or `.weekly([weekdays])` repeats, and/or a countdown (`preAlert`, `postAlert`) ([sample](https://developer.apple.com/documentation/alarmkit/scheduling-an-alarm-with-alarmkit)).
- The alert shows the title and the app name, with Stop and an optional secondary button (Repeat/snooze through `.countdown`, or a custom action through `.custom` that opens the app) (WWDC25 230; sample).
- **The key enforcement hook:** "You can provide a custom app intent for the stop button or for the secondary button", and an intent can set `openAppWhenRun` (WWDC25 230). `AlarmConfiguration` takes `stopIntent` and `secondaryIntent` ([sample](https://developer.apple.com/documentation/alarmkit/scheduling-an-alarm-with-alarmkit)). The secondary intent "is only available after first unlock" ([AlarmManager.AlarmConfiguration](https://developer.apple.com/documentation/alarmkit/alarmmanager/alarmconfiguration)). So Stop can open the Proof screen and re-arm the next alarm whenever Proof is missing. Stopping still silences the current alert; the app cannot refuse the Stop.
- The app gets alarm state through `alarmUpdates` even if it was not running when the state changed ([sample](https://developer.apple.com/documentation/alarmkit/scheduling-an-alarm-with-alarmkit)).
- If countdowns are used, a widget extension is required, "Otherwise, the system may unexpectedly dismiss alarms and fail to alert" (same page).
- Intended use: "a great fit for countdowns ... or recurring alerts with a schedule, like a wake-up alarm. They are not a replacement for other prominent notifications, like critical alerts or time-sensitive notifications" (WWDC25 230). A chore deadline sits between those. App Review could object if alarms are used for every small reminder, so keep alarms for the forced items only.
- No documented limit on the number of alarms. A forum thread asking for one has no Apple answer ([forum 787520](https://developer.apple.com/forums/thread/787520)). **Unverified; test on device.**

### Critical alerts

- `.critical`: "presents the notification immediately, lights up the screen, and bypasses the mute switch to play a sound. ... requires an approved entitlement. The system always presents this notification, even when Do Not Disturb is active" ([UNNotificationInterruptionLevel.critical](https://developer.apple.com/documentation/usernotifications/unnotificationinterruptionlevel/critical)).
- Entitlement `com.apple.developer.usernotifications.critical-alerts`. To get it you [fill out the request form](https://developer.apple.com/contact/request/notifications-critical-alerts-entitlement/) ([Critical Alerts entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.usernotifications.critical-alerts)). "Critical alerts require a special entitlement issued by Apple", and the app must also request `.criticalAlert` authorization ([UNAuthorizationOptions.criticalAlert](https://developer.apple.com/documentation/usernotifications/unauthorizationoptions/criticalalert)).
- Critical alerts are not in the self-service [supported capabilities](https://developer.apple.com/help/account/reference/supported-capabilities-ios) table, so even development builds need the approval.
- Apple's intended use: "Because of this disruptive behavior, critical interruption will continue to require an approved entitlement. Some use cases for critical notifications are severe weather and local safety alerts" ([WWDC21 10091](https://developer.apple.com/videos/play/wwdc2021/10091/)). Household chores do not fit, so assume the request would be refused.

### Time Sensitive notifications

- `.timeSensitive`: "can break through system controls such as Notification Summary and Focus. The user can turn off the ability for time sensitive notification interruptions" ([timeSensitive](https://developer.apple.com/documentation/usernotifications/unnotificationinterruptionlevel/timesensitive)).
- Needs the Time Sensitive Notifications capability, which is self-service in the [capabilities table](https://developer.apple.com/help/account/reference/supported-capabilities-ios) (WWDC21 10091). Apple asks developers to use them only "when it is relevant to actively interrupt, requiring immediate attention" and warns "Do not overuse their interruptive nature" (WWDC21 10091).
- Focus: the Member's Focus settings decide whether time-sensitive notifications get through ("You can also turn on Time Sensitive Notifications, which allows all apps to send time-sensitive notifications immediately", [iPhone User Guide](https://support.apple.com/guide/iphone/allow-or-silence-notifications-for-a-focus-iph21d43af5b/ios)).

### Repeating notifications

- For a repeating `UNTimeIntervalNotificationTrigger`, the time interval "parameter must be 60 seconds or greater", and with `repeats: true` "you must explicitly remove the notification request to stop the delivery" ([init(timeInterval:repeats:)](https://developer.apple.com/documentation/usernotifications/untimeintervalnotificationtrigger/init(timeinterval:repeats:))). So a reminder can repeat every minute until the app removes it after Proof.
- Apple's cancel guidance matches our flow: "if the user completes a reminder, you'd cancel any active requests associated with that reminder" ([Scheduling a notification locally](https://developer.apple.com/documentation/usernotifications/scheduling-a-notification-locally-from-your-app)).
- `UNNotificationCategoryOptions.customDismissAction` sends dismissals to the app's delegate ([customDismissAction](https://developer.apple.com/documentation/usernotifications/unnotificationcategoryoptions/customdismissaction)). That lets the app count dismissals for Escalation, but it cannot block them.
- "There is a limit of 64 for how many simultaneous notification requests can be active/pending at one time per app. This is a system limit and there is no way around it" (Apple engineer, [forum 811171](https://developer.apple.com/forums/thread/811171)). A repeating request counts as one.

### Live Activities

- A Live Activity "can be active for up to eight hours", then stays on the Lock Screen up to four more hours, "a maximum of 12 hours". "A person can remove your Live Activity from their Lock Screen at any time." "People can choose to deactivate Live Activities for an app in the Settings app." Data is limited to 4 KB. Needs `NSSupportsLiveActivities` and a widget extension. Can be scheduled for a start date, which requires an `AlertConfiguration` ([Displaying live data with Live Activities](https://developer.apple.com/documentation/activitykit/displaying-live-data-with-live-activities)).
- "In general, your app needs to be in the foreground to start a Live Activity", but a `LiveActivityIntent` can start one without opening the app ([LiveActivityIntent](https://developer.apple.com/documentation/appintents/liveactivityintent)).
- Role here: a visible "still open" item, not enforcement. The 8 h limit covers an evening reminder through to bedtime.

### Screen Time: FamilyControls, ManagedSettings, DeviceActivity

- FamilyControls authorizes "parental controls" for a `.child` (a parent approves) or an `.individual` (the device owner approves with Face ID / Touch ID) ([Family Controls](https://developer.apple.com/documentation/familycontrols), [requestAuthorization(for:)](https://developer.apple.com/documentation/familycontrols/authorizationcenter/requestauthorization(for:))). Fernando and Chloe are adults, so `.individual` is the only option. Family Sharing child accounts don't apply to them.
- Big limit for `.individual`: once authorized, "the system removes any restrictions that prevent the user from bypassing parental controls so the user can delete an authorized app or sign out of iCloud as needed" ([requestAuthorization(for:)](https://developer.apple.com/documentation/familycontrols/authorizationcenter/requestauthorization(for:))). The anti-tamper protection (can't delete the app, can't sign out of iCloud) applies only to children authorized by a guardian ([Family Controls](https://developer.apple.com/documentation/familycontrols)).
- Entitlement: adding the capability sets `com.apple.developer.family-controls`. "Before submitting your app to the App Store, you must [request permission](https://developer.apple.com/contact/request/family-controls-distribution) to use the entitlement" ([entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.family-controls)). The capabilities table lists it as "Family Controls (development)" with "Development only" ([supported capabilities](https://developer.apple.com/help/account/reference/supported-capabilities-ios)). Without approval, only development-signed builds can use it, which rules out TestFlight and Ad Hoc. *My inference from the table, not stated outright.*
- Shielding: `ShieldSettings.ActivityCategoryPolicy.all(except:)` shields "all apps and websites, except content that you specify" ([ActivityCategoryPolicy](https://developer.apple.com/documentation/managedsettings/shieldsettings/activitycategorypolicy)). An explicit app list holds "up to 50 application tokens at once" ([applications](https://developer.apple.com/documentation/managedsettings/shieldsettings/applications-swift.property)). The shield's look comes from a sandboxed `ShieldConfigurationDataSource` extension that cannot make network requests ([ShieldConfigurationDataSource](https://developer.apple.com/documentation/managedsettingsui/shieldconfigurationdatasource)). Taps on shield buttons go to a `ShieldActionDelegate` extension, which gets tokens, not app names ([ShieldActionDelegate](https://developer.apple.com/documentation/managedsettings/shieldactiondelegate)).
- "The system doesn't guarantee that the settings you specify govern the device's behavior" ([ManagedSettingsStore](https://developer.apple.com/documentation/managedsettings/managedsettingsstore)).
- Timing: a `DeviceActivitySchedule` plus a monitor extension can apply the shield at the due time while the app is closed ([DeviceActivity](https://developer.apple.com/documentation/deviceactivity), [DeviceActivitySchedule](https://developer.apple.com/documentation/deviceactivity/deviceactivityschedule)). "Attempting to monitor too many activities or activities that are too tightly scheduled can cause this method to throw" ([startMonitoring](https://developer.apple.com/documentation/deviceactivity/deviceactivitycenter/startmonitoring(_:during:events:))).
- App Review: Guideline 4.10 forbids monetizing "Screen Time APIs" ([App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)). That doesn't matter for a free household app.

### Focus

No API lets an app change or override the Member's Focus. Only three things get through it: AlarmKit (always, per the sources above), critical alerts (always), and time-sensitive notifications (if the Member allows them).

### Camera-only Proof capture

- `UIImagePickerController` with `sourceType = .camera` "provides a user interface for taking a new picture"; `.photoLibrary` is a separate source we simply don't offer. For full control (no retake-from-library, timestamps, overlays), use AVFoundation ([UIImagePickerController](https://developer.apple.com/documentation/uikit/uiimagepickercontroller), [.camera](https://developer.apple.com/documentation/uikit/uiimagepickercontroller/sourcetype-swift.enum/camera)).
- `NSCameraUsageDescription` "is required if your app uses APIs that access the device's camera" ([NSCameraUsageDescription](https://developer.apple.com/documentation/bundleresources/information-property-list/nscamerausagedescription)). Guideline 2.5.14 requires consent and a clear indication when recording ([App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)).
- Limit: the app controls *where* the photo comes from (live camera, now), not *what* it shows. Whether the Proof is real is a human call, which fits sending it to the other Member as part of Escalation.

## Distribution for two people

| Route | Cost / limits | Works for this app? |
|---|---|---|
| Free Apple Account (Personal Team in Xcode) | Up to 3 devices and 3 apps per device; App IDs and devices "expire after 7 days" and you must "rebuild and reinstall" ([compare memberships](https://developer.apple.com/support/compare-memberships/)). No TestFlight or App Store Connect | Workable only for quick tests. Weekly re-install from a Mac for both phones. Whether Family Controls / Time Sensitive capabilities are available to a Personal Team isn't stated in the text I could read: **unverified** |
| Apple Developer Program, development-signed builds installed from Xcode | Paid annual membership (price not stated on the pages read). Up to 100 iPhones per membership year ([devices overview](https://developer.apple.com/help/account/devices/devices-overview)) | **Best fit.** Family Controls (development) works without Apple's distribution approval. Needs a Mac nearby for installs and occasional re-signing |
| Ad Hoc (paid program) | Registered devices only. Apple itself points apps meant for "family and friends" here or to Xcode installs ([App Review Guidelines intro](https://developer.apple.com/app-store/review/guidelines/)) | Yes for everything except Family Controls, which needs distribution approval |
| TestFlight internal testing (paid program) | Up to 100 internal testers who are team members; builds expire after 90 days ([TestFlight](https://developer.apple.com/testflight/), [overview](https://developer.apple.com/help/app-store-connect/test-a-beta-version/testflight-overview)). Only external testing triggers Beta App Review | Easiest for OTA updates. Chloe must be added to the team. Family Controls needs distribution approval |
| App Store | Full review | Not needed. Guideline 2.5.1 ("use APIs and frameworks for their intended purposes") and AlarmKit's intended use would be review risks |

EU alternative marketplaces / web distribution are out of scope. They still need a paid program membership and notarization, and add nothing for two people.

## Hard limits (what no app can do)

- Make an alert undismissable. AlarmKit Stop, notification swipe and Live Activity removal are always available.
- Stop the Member from revoking permissions (notifications, alarms, Live Activities, Screen Time) in Settings, or deleting the app, as an adult `.individual`.
- Override Focus beyond the three exceptions above. The app can't change the Member's Focus.
- Get critical alerts for chores. Apple approves them for safety use only.
- Keep an unlimited queue: 64 pending notifications per app.
- Confirm what a photo shows. Proof quality is a human judgement.

What the app *can* rely on: it knows when a reminder was stopped or dismissed without Proof, so every bypass can feed Escalation to the other Member.

## Open questions to test on device

- How many AlarmKit alarms an app can hold, and how fast it can re-arm (the docs don't say).
- Whether a Stop-button App Intent with `openAppWhenRun` reliably opens the app from the Lock Screen (after authentication) on current iOS.
- Whether a Personal Team (free account) can sign the Family Controls and Time Sensitive capabilities.
