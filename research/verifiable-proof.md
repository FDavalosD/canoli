# What Proof can be verified on iPhone

Research for issue #7. Question: what kinds of **Proof** can an iPhone app verify, and how hard is each to fake? Proof is what a Member provides to mark something done, and it has to take more effort than pressing a button (see `GLOSSARY.md`).

Sources are Apple developer documentation (read through its JSON endpoints, `developer.apple.com/tutorials/data/documentation/...`) and Anthropic's Claude platform docs, fetched 2026-11-23. Claims not backed by a source are marked **(inference)**.

## Short answer

| Proof type | How hard to fake | Effort for the Member | Running cost | Main limit |
|---|---|---|---|---|
| Live camera photo (in-app capture, no gallery) | Low to medium: blocks old gallery photos, but a staged scene or a photo of a screen still passes | Low (one tap plus framing) | Free | Shows that a photo was taken, not that the task was done |
| Before/after photo pair | Medium: needs two in-app captures separated in time | Medium (two visits, minutes apart) | Free | The Member still decides what to point the camera at |
| AI vision check on the photo(s) | Adds medium resistance: catches wrong or lazy photos, not careful staging | None beyond the photo | About $0.0003 per check (Haiku 5.5) to about $0.014 (Opus 5.5) | Accuracy on our tasks is unmeasured; can't detect AI-generated images; needs a server to hold the API key |
| HealthKit workout | Medium to high if we require a sensor-recorded workout; low if we accept manual entries | None if they already record runs | Free | Only fits exercise; Member has to grant read access, and the app can't tell a denial from no data |
| NFC tag scan | Medium: proves the phone was near the tag; a tag can be peeled off or the scan done without doing the task | Low (tap phone to tag) | Tags cost cents each **(inference)** | Needs iPhone XS+ for background reads; a tag only proves presence |
| Location check | Low on its own: proves where the phone was | None (automatic) | Free | Home chores all happen at home, so location adds almost nothing for them |
| Other Member confirms | High: a human checks, and the two Members can't easily collude against themselves | Low for the doer, real effort for the confirmer | Free | Depends on the other Member being available; slows dismissal |

**Recommendation (inference):** use the live in-app camera as the default Proof, with a before/after pair for chores where "after" is visibly different (cleaning, tidying, laying out clothes). Add an AI check on top so a blank wall or a random photo doesn't pass. Use HealthKit for workouts and NFC for "I was at the spot" tasks (laundry machine, front door). Keep "the other Member confirms" as the strongest level, used as an Escalation step or for disputed Proof. Location is only worth it for tasks away from home.

## 1. Live camera photo (no gallery)

**How it works.** The app captures the photo itself instead of opening the photo library. AVFoundation's `AVCapturePhotoOutput` is "a capture output for still image, Live Photos, and other photography workflows"; the app calls `capturePhoto(with:delegate:)` and gets the image back in its delegate ([AVCapturePhotoOutput](https://developer.apple.com/documentation/avfoundation/avcapturephotooutput)). A simpler option is `UIImagePickerController` with source type `.camera`, which "specifies the device's built-in camera as the source" ([UIImagePickerController.SourceType.camera](https://developer.apple.com/documentation/uikit/uiimagepickercontroller/sourcetype-swift.enum/camera)). Either way the image goes straight from the camera into the app.

The app must declare `NSCameraUsageDescription`, and the user must grant camera access. Until they do, "the system only vends black video frames" ([requestAccess(for:completionHandler:)](https://developer.apple.com/documentation/avfoundation/avcapturedevice/requestaccess(for:completionhandler:))).

SwiftUI's `PhotosPicker` is the opposite case: it picks from the library, and it is the thing we must not offer. It has no camera mode (see [PhotosPicker](https://developer.apple.com/documentation/photokit/photospicker)). The app enforces "no gallery" by never showing it.

**Metadata.** An `AVCapturePhoto` carries a metadata dictionary that "may include image orientation, Exif camera properties" ([AVCapturePhoto.metadata](https://developer.apple.com/documentation/avfoundation/avcapturephoto/metadata)). Because we capture in-app, the app also knows the capture time from its own clock. It doesn't need to trust EXIF **(inference)**.

**How hard to fake.** It blocks the easiest cheat, reusing an old photo from the library. It doesn't stop the Member from photographing a photo on another screen, or from staging the scene (laying out the running clothes for the photo, then putting them back). A modified or jailbroken phone could feed fake frames to the app. Apple says an app "can't rely on [its] logic to perform security checks on itself because a compromised app can falsify the results" ([Establishing your app's integrity](https://developer.apple.com/documentation/devicecheck/establishing-your-app-s-integrity)). App Attest can prove that requests come from a real copy of the app ([DCAppAttestService](https://developer.apple.com/documentation/devicecheck/dcappattestservice)). For a two-person household that is overkill **(inference)**.

**Effort.** Open the Proof screen, frame, tap. A few seconds. That is more than a button press, but not by much.

**Cost.** Free on the device.

**Limits.** It proves that a photo was taken now, not that the task was done.

## 2. Before/after photo pair

**How it works.** Same in-app capture, done twice: a "before" photo when the Member starts and an "after" photo when they finish, with a minimum gap the app enforces using its own timestamps **(inference: app design, not an Apple API)**.

**How hard to fake.** Harder than a single photo, because the Member has to come back after a delay, and the two photos have to show the same place in different states. Staging is still possible: mess the room up, take "before", tidy, take "after". For the household that is about as much work as the chore itself, which is the point **(inference)**.

**Effort.** Medium: two captures and a wait. It doesn't suit tasks with no visible "before" (for example "take your vitamins").

**Cost.** Free on the device. Doubles the image cost of an AI check (section 3).

**Limits.** Only useful for tasks with a visible change of state.

## 3. AI vision check (photo matches the task)

**How it works.** The app sends the photo(s) and the task text ("bathroom cleaned: sink, mirror, toilet") to a Claude model and asks for a structured yes/no with a reason. Vision is supported across the current Claude 5 family, including `claude-haiku-5-5`, `claude-sonnet-5-5` and `claude-opus-5-5` ([Vision](https://platform.claude.com/docs/en/build-with-claude/vision)).

**Cost.** Images are billed as input tokens. "An image … costs ⌈width / 28⌉ × ⌈height / 28⌉ visual tokens". Claude 4.7 and later models are on a high-resolution tier capped at a 2,576 px long edge and 4,784 visual tokens ([Vision: Resolution and token cost](https://platform.claude.com/docs/en/build-with-claude/vision#evaluate-image-size)). A 1000×1000 image costs 1,296 tokens; a 4K photo is downscaled to 4,784 tokens. So the app should resize photos to about 1000 px before sending.

Prices per million tokens, input / output ([Pricing](https://platform.claude.com/docs/en/about-claude/pricing)):

- Claude Haiku 5.5: $0.10 / $0.50 (prompts up to 100K tokens)
- Claude Sonnet 5.5: $2 / $10
- Claude Opus 5.5: $4 / $20
- The Batch API halves these, but it's asynchronous, so it doesn't fit a "dismiss now" flow.

Worked estimate for a before/after pair at 1000×1000 (2 × 1,296 image tokens plus about 300 prompt tokens, about 100 output tokens; thinking tokens, billed as output, come on top) **(calculation)**:

| Model | Per check | 300 checks / month |
|---|---|---|
| Haiku 5.5 | about $0.0003 | about $0.10 |
| Sonnet 5.5 | about $0.007 | about $2 |
| Opus 5.5 | about $0.014 | about $4 |

For one household this is cheap even on Opus. Model choice should depend on accuracy, not price.

**Accuracy.** Anthropic publishes no accuracy figure for "does this photo show a clean bathroom". The documented limitations are ([Vision: Limitations](https://platform.claude.com/docs/en/build-with-claude/vision#limitations)):

- It "might hallucinate or make mistakes when interpreting low-quality, rotated, or very small images under 200 pixels."
- Counting is approximate ("might not always be precisely accurate").
- AI-generated images: Claude "cannot determine whether an image is AI-generated". Don't rely on it to catch synthetic photos. In-app capture (section 1) is what blocks those, not the AI check.
- It won't identify people by name, so it can't confirm *which* Member is in a photo.
- Anthropic advises human review for high-stakes uses.

The way to settle accuracy is to build a small eval from our own photos (say 30 done and 30 not-done photos per chore type) and measure each model on it **(inference)**.

**How hard to fake.** It stops lazy cheats (a random photo, the wrong room, a half-done job). It doesn't stop careful staging, and a model judging "clean" will sometimes be wrong both ways. False rejections matter: a Member who did the chore and gets rejected will lose trust in the app, so a rejection should fall back to "the other Member confirms" rather than to a dead end **(inference)**.

**Effort.** Nothing beyond taking the photo(s). It adds a few seconds of network round trip.

**Limits.** Calling the Claude API needs an API key. Shipping that key inside the iPhone app would expose it, so the check needs a small server or serverless proxy that holds the key **(inference)**. It also sends household photos to a third party, which Fernando and Chloe should agree to.

## 4. Apple Health / HealthKit workouts

**How it works.** The app asks for read access to workouts and queries `HKWorkout` samples. An `HKWorkout` is "a workout sample that stores information about a single physical activity", with an activity type, start/end and duration, plus associated samples such as distance, energy and heart rate ([HKWorkout](https://developer.apple.com/documentation/healthkit/hkworkout)). For "go for your run" Proof, the app checks for a running workout of at least N minutes in the expected window.

Setup: add the HealthKit capability, set `NSHealthShareUsageDescription`, check `isHealthDataAvailable()`, then request per-type permission ([Setting up HealthKit](https://developer.apple.com/documentation/healthkit/setting-up-healthkit), [Authorizing access to health data](https://developer.apple.com/documentation/healthkit/authorizing-access-to-health-data)).

**How hard to fake.** Depends on what we accept:

- Every sample records which app or device saved it in `sourceRevision`, and devices that write directly also fill `device` ([HKObject.sourceRevision](https://developer.apple.com/documentation/healthkit/hkobject/sourcerevision), [HKObject.device](https://developer.apple.com/documentation/healthkit/hkobject/device)). We can accept only workouts recorded by an Apple Watch or by the Workout app.
- `HKMetadataKeyWasUserEntered` marks samples "entered by the user" ([HKMetadataKeyWasUserEntered](https://developer.apple.com/documentation/healthkit/hkmetadatakeywasuserentered)). We should reject those.
- A watch workout session "generate[s] high-frequency heart rate samples" ([HKWorkoutSession](https://developer.apple.com/documentation/healthkit/hkworkoutsession)), so we can also require heart-rate samples. iPhone alone has no heart-rate sensor (same page).
- Other apps with write permission can save workouts. Any app can set the user-entered key or not, so the source check matters more than that key **(inference)**.

With a watch-recorded workout that has heart rate and distance, faking means actually moving for the duration. That is close to doing the run.

**Effort.** None if the Member already records runs. Proof arrives on its own once the workout syncs.

**Cost.** Free.

**Limits.**
- Only fits exercise.
- Privacy design: "your app doesn't know whether someone granted or denied permission to read data", and a denial looks like having no data. People can also grant access to "only a limited window of recent data" ([Authorizing access to health data](https://developer.apple.com/documentation/healthkit/authorizing-access-to-health-data)). The app has to treat "no workout found" as "not proven", not as "you lied".
- The example "prepare your clothes for tomorrow's run" is not a workout. HealthKit can prove the run happened the next day, but the evening task still needs a photo.

## 5. NFC tags (Core NFC)

**How it works.** Stick an NFC tag at the place where a task happens (washing machine, bathroom door, the shelf where the running clothes go). Core NFC reads "NFC tags of types 1 through 5 that contain data in the NFC Data Exchange Format (NDEF)" and can write to tags ([Core NFC](https://developer.apple.com/documentation/corenfc)). `NFCTagReaderSession` also reads ISO 7816, ISO 15693, FeliCa and MIFARE tags ([NFCTagReaderSession](https://developer.apple.com/documentation/corenfc/nfctagreadersession)). These expose "the unique hardware identifier of the tag" ([NFCMiFareTag.identifier](https://developer.apple.com/documentation/corenfc/nfcmifaretag/identifier)), so the app can bind a task to one physical tag instead of to whatever URL a tag contains.

There are two ways to scan:

- **In-app reader session.** The app starts a session, and the Member holds the phone to the tag. Only one reader session can be active in the system at a time ([NFCReaderSession](https://developer.apple.com/documentation/corenfc/nfcreadersession-swift.class)).
- **Background tag reading.** On iPhone XS and later, the system reads a tag holding a universal link, shows a notification, and opens the app when the Member taps it. It doesn't work when the phone has never been unlocked, while the camera is in use, or in airplane mode, and it doesn't support custom URL schemes ([Adding support for background tag reading](https://developer.apple.com/documentation/corenfc/adding-support-for-background-tag-reading)).

Core NFC "isn't available for use in app extensions", and the app should check `readingAvailable` first (Core NFC page). Setup: `NFCReaderUsageDescription` plus the reader-session-formats entitlement (NFCTagReaderSession page).

**How hard to fake.** A scan proves the phone was within a few centimetres of the tag at that moment. It doesn't prove the task was done: the Member can walk to the machine, tap, and walk away. An NDEF tag's content can be copied to another tag. Checking the hardware identifier makes copying harder, though some cheap tags allow the UID to be changed **(inference)**. Someone could also peel the tag off and keep it by the sofa. Within one household that is unlikely, and it would be visible.

**Effort.** Low: walk there and tap. That is more effort than a button only because it forces the Member to be physically at the task.

**Cost.** Free in software. Tags cost a few cents to a dollar each **(inference; not from a primary source)**.

**Limits.** Proves presence, not completion. Pair it with a photo for chores; use it alone for "check-in" tasks (for example "lock the back door" with a tag on the door).

## 6. Location checks (Core Location)

**How it works.** Either take a one-time fix with `requestLocation()`, which delivers a single location and then stops location services ([requestLocation()](https://developer.apple.com/documentation/corelocation/cllocationmanager/requestlocation())), or monitor a circular region (geofence). The system relaunches the app when a condition is satisfied. There is a limit of 20 monitored conditions per app, and monitoring only starts after the first unlock after a reboot ([Monitoring the user's proximity to geographic regions](https://developer.apple.com/documentation/corelocation/monitoring-the-user-s-proximity-to-geographic-regions)).

**How hard to fake.** `CLLocation.sourceInformation` reports `isSimulatedBySoftware` (for example, a GPX file loaded through the Xcode debugger) and `isProducedByAccessory` (an external GPS or CarPlay). Apple suggests rejecting simulated locations "when knowing the true location of the device … is critical" ([CLLocation.sourceInformation](https://developer.apple.com/documentation/corelocation/cllocation/sourceinformation), [isSimulatedBySoftware](https://developer.apple.com/documentation/corelocation/cllocationsourceinformation/issimulatedbysoftware), [isProducedByAccessory](https://developer.apple.com/documentation/corelocation/cllocationsourceinformation/isproducedbyaccessory)). So casual spoofing is detectable. The real weakness is that location only proves where the *phone* was. The Member can leave it at the gym, or a family member can carry it **(inference)**.

**Effort.** None.

**Cost.** Free (some battery for monitoring).

**Limits.** Almost useless for home chores, because the Member is already at home whether or not the bathroom gets cleaned. Useful for away-from-home tasks ("drop the parcel at the post office", "go to the gym"), and as extra evidence on a HealthKit run.

## 7. The other Member confirms

**How it works.** The Proof (photo, note) goes to the other Member, who approves or rejects it. Fernando and Chloe each have their own iPhone. CloudKit sharing lets an owner share records with other iCloud users and manage what participants can do ([Shared records](https://developer.apple.com/documentation/cloudkit/shared-records)), so both phones can see the same Proof and its approval state without a custom backend. A notification tells the confirmer there is something to review **(inference: app design)**.

**How hard to fake.** Hardest of all: a person who lives in the same home looks at the Proof, and can walk to the bathroom to check. The weak spot is the confirmer rubber-stamping without looking. The app can require the confirmer to open the photo before approving **(inference)**.

**Effort.** Low for the doer, real for the confirmer. It also adds delay, since the reminder isn't dismissed until the other Member responds.

**Cost.** Free.

**Limits.** Depends on the other Member being around and willing. It fits as a fallback (AI rejected a photo, or the Member disputes it) and as an Escalation step, more than as the everyday Proof.

## Open questions for the map

- Do Fernando and Chloe accept sending household photos to Anthropic for AI checks? If not, Proof is photo plus the other Member's review, without AI.
- If AI checks are in, a small server or serverless function is needed to hold the API key. That affects the architecture ticket.
- AI accuracy on our own chores needs measuring with an eval before we decide which model to use.
- Does either Member wear an Apple Watch? That decides whether HealthKit Proof can require sensor data.
