# Existing apps worth borrowing from

Research for issue #5. Question: which existing apps cover parts of what we want (Proof, Escalation, chore division between the two Members, GTD with email Capture, habits), what does each enforce and how, what can we borrow, and does any combination make building our own unnecessary?

Sources are the apps' own sites, App Store listings, official support pages and Apple developer docs, read on 2026-11-23. Where only a secondary source was available, it says so. Lines marked **Inference** are my reading, not something a source states.

## Short answer

No app or combination covers the core of what we want. Three things are missing everywhere:

1. **Proof on household chores.** Effortful Proof exists only in alarm apps (Alarmy) and money-on-the-line habit apps (Forfeit). No chore app asks for anything beyond a tap. The closest is Sweepy, where a parent approves a child's task.
2. **Escalation that ends with the other Member being told.** Forfeit can text or email a buddy when you fail, but it's a solo habit app with strangers or AI reviewing the proof. No chore or GTD app escalates to a partner.
3. **A shared GTD system for two people.** Things has no collaboration at all. OmniFocus's features page describes no multi-user sharing. Apple Reminders has shared lists with assignment but no GTD structure (no review, no clarify step).

The best off-the-shelf stack would be **OmniFocus or Things (GTD + email inbox) + Tody Premium+ Duo (chore split) + Forfeit or Alarmy (Proof for a few habits)**. It would still leave both Members juggling four apps, with no Proof on chores and no partner Escalation. Those are the headline features, so building is still justified. The stack is cheap enough to trial for a couple of weeks first, to find out which enforcement mechanics actually work on Fernando and Chloe before any code is written.

## App by app

| App | What it enforces | How | Borrow |
|---|---|---|---|
| Alarmy | Getting up | Alarm keeps ringing until a mission is done (photo of a set object, QR/barcode, math, squats, steps, shaking); premium "wake up check" re-rings if you fall back asleep | Photo-of-a-registered-object Proof; chaining missions; a "still awake?" re-check after dismissal |
| Forfeit | Doing a habit by a deadline | Money at stake; proof by photo (AI-checked), timelapse (human-reviewed), friend-verify, Health, GPS, timers; failure triggers your accountability method, e.g. texting a buddy | Multiple Proof types per task; friend-verify (the other Member as verifier); appeals; "hard mode" with no appeals |
| Beeminder | Staying on a committed rate | Pledge charged when you cross the "Bright Red Line"; changes to a goal only take effect a week later | The one-week delay on weakening a commitment (blocks quitting in a moment of weakness); autodata instead of self-report |
| Habitica | Doing dailies | Game: missed dailies cost health; in a party boss quest one player's misses damage *every* participant | Shared consequence: one Member's miss visibly costs both. Good for two, but game cosmetics are optional |
| Streaks | Daily habits | Streak count; reminders while a task is due; Health-linked tasks complete themselves; tasks can be shared with family via iCloud | Health auto-completion as Proof for workouts and steps; iCloud-only sharing (no own server) |
| Tody | Household cleaning | Each task has an interval and a green-to-red dueness bar; assignment and auto-rotation; FairShare and a monthly leaderboard; household sync on Premium+ | Dueness model (intervals, not due dates); auto-rotation; FairShare split. No Proof |
| Sweepy | Household cleaning | Smart schedule sized to available time; points, streaks, family leaderboard; kids' tasks count only after parent validation | Validation step (one Member approves the other's completion); effort-sized daily list |
| OurHome | Kids' chores | Points per chore, redeemed for parent-approved rewards; shared calendar and lists | Little; aimed at parents and children. Only secondary sources found |
| Things 3 | GTD structure | Clean Inbox/Today/Anytime/Someday; Mail to Things gives a private address whose emails land in the Inbox | Mail-to-inbox Capture pattern; the visual polish. No collaboration |
| OmniFocus 4 | GTD structure | Inbox, defer/due, Review that prompts periodic project review, perspectives, Forecast; Mail Drop emails land in the Inbox with attachments | Review cadence as a forced weekly review; defer dates; Mail Drop's per-person addresses |
| Apple Reminders | Shared lists | iCloud shared lists; assign a reminder to anyone on the list with @name; notifications on adding and completing items | Reference for shared-list UX and @-assignment; possible sync substrate via EventKit (not researched here) |
| Couples to-do apps (Honeydo Tasks, HoneyDo Together, CouplesDo, Halves, Merge, Tend Task) | Shared to-dos | Assign tasks to each other, recurring tasks, shopping lists | Nothing on enforcement; HoneyDo Together syncs through the users' own iCloud |
| Fair Play (Eve Rodsky) | Division of household labour | A physical deck of about 100 task cards; the official site lists book, cards and curriculum but no official app | The card deck as a starter catalogue for chore division |

## Details and sources

### Alarmy

- Missions include math, memory, shaking, copy-typing, squats, QR/barcode scan, walking and photo. The photo mission requires photographing a designated object or place. ([alar.my](https://alar.my/))
- The alarm stays on until the mission is done. An "anti-re-sleep" premium feature re-rings until you're confirmed awake. ([alar.my](https://alar.my/))
- The iOS listing requires iOS 17+. The developer notes that iOS blocks the alarm sound if the app has been terminated in the background. Reviewers note you can still escape by deleting the alarm. ([App Store](https://apps.apple.com/us/app/alarmy-alarm-clock-sleep/id1163786766))
- Free, with premium via in-app purchase (listing shows $4.99–$69.99 tiers; 7-day trial). ([App Store](https://apps.apple.com/us/app/alarmy-alarm-clock-sleep/id1163786766))

### Forfeit: Habit Contracts

- Proof methods: photo (AI checks it against your description), timelapse (human-reviewed), self-verify, friend-verify, Apple Health thresholds, GPS check-ins, RescueTime, Pomodoro timers. ([App Store](https://apps.apple.com/us/app/id1633125787))
- Rejected or skipped submissions can be appealed to a human; "hard mode" disables appeals. ([App Store](https://apps.apple.com/us/app/id1633125787))
- On failure, your chosen accountability method fires, such as texting or emailing a buddy. User reviews say failed forfeits charge your card. ([App Store](https://apps.apple.com/us/app/id1633125787))
- Free download; subscriptions from $3.99/week up to $99.99/year. 4.8 stars, frequent updates. ([App Store](https://apps.apple.com/us/app/id1633125787))
- **Inference:** Forfeit shows that photo Proof needs a verifier: AI, a human or a friend. For us, the obvious verifier is the other Member, which ties Proof to Escalation.

### Beeminder

- Each goal is a commitment contract: you're charged when you go off track (cross the "Bright Red Line"), and you set how high the pledge can climb. ([beeminder.com/overview](https://www.beeminder.com/overview))
- Data comes from reminders you reply to, or from integrations such as Strava and RescueTime. ([beeminder.com/overview](https://www.beeminder.com/overview))
- You can quit or change a goal at any time, but the change takes effect a week later. ([beeminder.com/overview](https://www.beeminder.com/overview))

### Habitica

- Three task types: habits, dailies and to-dos. Completing them earns gold, XP and items. Parties add accountability. Free; subscription $4.99/month or $47.99/year. ([App Store](https://apps.apple.com/us/app/habitica-gamified-taskmanager/id994882113))
- In a boss quest, a player's missed dailies make the boss attack all quest participants, including the player who missed. Source: the community-maintained Habitica wiki (pages such as [Death](https://habitica.fandom.com/wiki/Death) and [FAQ](https://habitica.fandom.com/wiki/FAQ)), surfaced via search because the wiki blocked direct fetching. Treat the exact mechanics as unverified.

### Streaks

- Tasks are completed by tapping (press and hold), or automatically from Health data such as steps; also via Siri, Shortcuts and Apple Watch. ([streaks.app](https://streaks.app/))
- Reminds you while a task is still due. Tasks can be shared with friends and family, who see your progress; sharing goes through iCloud rather than Streaks' servers. ([streaks.app](https://streaks.app/))
- Up to 24 tasks per day. One-time purchase, no subscription. ([streaks.app](https://streaks.app/))

### Tody

- Each task has its own interval. A colour bar runs from green (can wait) to red (overdue), and Today shows only what's due. ([todyapp.com](https://www.todyapp.com/))
- Assign chores once or rotate them automatically each cycle (free). A monthly race shows each member's share of completed tasks; Dusty the dust-bunny mascot is the rival. ([todyapp.com](https://www.todyapp.com/))
- Household sync and FairShare need Premium+, from $25/year for a Duo. No photo or other verification is described. ([todyapp.com](https://www.todyapp.com/))

### Sweepy

- Smart Schedule builds a cleaning list that fits the time you have; tasks are worth up to 3 points and filterable by difficulty and dirtiness. ([sweepy.com](https://sweepy.com/))
- Invite family or roommates. Children's tasks are "marked as completed only after a parent's validation." Leaderboard, streaks, history. ([sweepy.com](https://sweepy.com/))
- Automatic schedule generation is premium. No proof beyond parental validation. ([sweepy.com](https://sweepy.com/))

### OurHome

- Not verified against a primary source: the official site reset the connection and no App Store listing turned up. Secondary roundups describe parents assigning chores, kids earning points redeemable for parent-approved rewards, plus a shared calendar and grocery lists, free. ([Greenlight roundup](https://greenlight.com/best-chore-apps-for-families), [Little Day Out](https://www.littledayout.com/?p=225799))

### Things 3

- Mail to Things: enable it to get a private `@things.email` address. The subject becomes the to-do title and the body its note. Attachments are ignored; bodies over 4,000 characters are truncated; up to 100 emails per day. ([Cultured Code support](https://culturedcode.com/things/support/articles/2908262/))
- No collaboration: "Things does not support real-time sharing of to-dos or lists." Assigning to a contact is private to you. Family Sharing only shares the purchase. ([Getting Things](https://culturedcode.com/things/support/articles/2803552/), [Things Cloud](https://culturedcode.com/things/support/articles/2803586/), [Contacts](https://culturedcode.com/things/support/articles/2803571/))
- Mac, iPhone/Watch, iPad, Vision Pro. ([culturedcode.com/things](https://culturedcode.com/things/))

### OmniFocus 4

- Inbox capture; defer and due dates with notifications; Review that periodically reminds you to check projects; custom perspectives and Focus (Pro); Forecast; Nearby. Mac, iPhone/iPad, Vision Pro, Watch, Web. The features page mentions no multi-user sharing. ([omnigroup.com features](https://www.omnigroup.com/omnifocus/features))
- Mail Drop: create one or more addresses on your Omni Sync Server account, for example one per person or integration. The subject becomes the Inbox item, the body its note, and attachments are kept. ([Omni support](https://support.omnigroup.com/omnifocus-mail-drop/))

### Apple Reminders (shared lists)

- Share a list via iCloud. Participants can add, edit and complete items. Assign a reminder to anyone on the list with `@name` or the Person button. Notify-when options are "Adding Items" and "Completing Items". Everyone must use iCloud Reminders. ([Apple iPhone User Guide](https://support.apple.com/guide/iphone/share-and-collaborate-iph2a8f9121e/ios))

### Couples to-do apps

- Honeydo Tasks (assign tasks to each other), HoneyDo Together (syncs through your own iCloud), CouplesDo (recurring tasks, to-dos, shopping), Halves, Merge, Tend Task. None advertises Proof or Escalation. ([Honeydo Tasks](https://apps.apple.com/us/app/honeydo-tasks/id6475327895), [HoneyDo Together](https://apps.apple.com/us/app/id6764382729), [CouplesDo](https://apps.apple.com/us/app/couplesdo/id6758944472), [Halves](https://apps.apple.com/app/id6759830825), [Merge](https://apps.apple.com/us/app/id1227082205), [Tend Task](https://apps.apple.com/app/id1456102211))
- Fair Play: the official site lists the book, cards, documentary and curriculum, but no app. ([fairplaylife.com](https://www.fairplaylife.com/)) The cards are a physical deck of about 100 household tasks (per secondary coverage, e.g. [NPR](https://www.npr.org/2022/09/16/1123560719/splitting-chores-partner-roommate)).

## Platform facts that limit "nags that can't be ignored"

These matter for any iPhone app we build:

- **AlarmKit** (iOS 26+) lets third-party apps schedule alarms and timers. An alarm "overrides both a device's focus and silent mode, if necessary." It needs `NSAlarmKitUsageDescription` and user authorization. ([AlarmKit](https://developer.apple.com/documentation/alarmkit), [Scheduling an alarm with AlarmKit](https://developer.apple.com/documentation/alarmkit/scheduling-an-alarm-with-alarmkit))
- The alert has a stop button and an optional secondary button: either "Repeat" (re-alerts after a set interval) or "Open" (launches the app). App Intents can run on these. "AlarmManager automatically handles stop or countdown functionalities." ([Scheduling an alarm with AlarmKit](https://developer.apple.com/documentation/alarmkit/scheduling-an-alarm-with-alarmkit))
- **Inference:** the system owns the stop button, so an app can't put Proof in front of stopping an alarm. What it can do is re-schedule the alarm from the stop intent until Proof arrives, and start Escalation if it doesn't. Alarmy's mission-to-dismiss works inside its own app, and its developer admits iOS stops the alarm if the app is killed.
- **Critical alerts** ignore the mute switch and Do Not Disturb, but "require a special entitlement issued by Apple." ([UNAuthorizationOptions.criticalAlert](https://developer.apple.com/documentation/usernotifications/unauthorizationoptions/criticalalert)) **Inference:** a household chore app is unlikely to get it. AlarmKit is the realistic route.

## Open questions this didn't settle

- Whether OmniFocus has any real multi-user sharing beyond multiple Mail Drop addresses (pricing and sharing pages didn't render).
- Beeminder's reminder escalation ("zeno polling") and pledge schedule; the help pages fetched didn't cover them.
- Whether Apple Reminders/EventKit could be the shared data layer instead of our own sync.
