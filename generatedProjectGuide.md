Social Planner iOS App — Technical Spec (v0.1)
Goal: A local-first iOS app that creates and maintains a schedule for social interactions you define (who + purpose + cadence), reminds you on time, and makes rescheduling effortless. No reading or sending messages—just planning, nudging, and simple logs.

1) Target Platforms & Baseline
iOS 17+ (works great on 18+, but we’ll target 17 for stability).
Local-first architecture; optional cloud sync later.

2) Core Apple Frameworks
Contacts (Contacts, CNContactPickerViewController) – select people, store identifier references.

EventKit – create/update Calendar events or Reminders (choose one as MVP; Reminders are lighter).

UserNotifications – local notifications with action buttons (Done / Snooze / Reschedule).

BackgroundTasks (BGTaskScheduler) – daily/periodic “planner refresh”.

App Intents – Shortcuts/Siri hooks (e.g., “Plan my week”, “Snooze tonight”).

Core Data – local storage (mirror selected contacts, goals, plan items, logs, prefs).

Optional later: CoreLocation + MapKit (place suggestions), CloudKit/Supabase (sync), Widgets.

3) Permissions (JIT prompts + copy)
Contacts: “We’ll only read the names you pick to plan social check-ins. We never read messages.”

Reminders/Calendar: “We’ll add schedule entries so they show up across your devices.”

Notifications: “We’ll remind you when it’s time and let you snooze or reschedule with one tap.”

4) Data Model (Core Data entities)
pgsql
Copy
Edit
ContactRef {
  id: UUID
  cnIdentifier: String      // CNContact.identifier
  displayName: String
  tags: [String]            // e.g., ["mentor", "friend"]
  purpose: String?          // e.g., "career check-in"
}

Goal {
  id: UUID
  contactId: UUID
  cadenceDays: Int          // e.g., 7, 30, 90
  importance: Int           // 1..5 weight
  minDurationMin: Int       // session length (e.g., 30)
  windowsJSON: String       // serialized availability rules (see below)
  lastCompletedAt: Date?    // last successful interaction
}

PlanItem {
  id: UUID
  contactId: UUID
  startAt: Date
  durationMin: Int
  status: String            // "pending" | "done" | "snoozed" | "rescheduled" | "canceled"
  ekIdentifier: String?     // EventKit id (event or reminder)
  origin: String            // "auto" | "manual"
  createdAt: Date
}

InteractionLog {
  id: UUID
  contactId: UUID
  when: Date
  outcome: String           // "met" | "call" | "text" | "other"
  notes: String?
}

Prefs {
  id: UUID (singleton)
  capacityPerWeek: Int      // max planned sessions/week
  quietHours: [Int]         // [startHour, endHour] local
  defaultNotificationTimes: [String] // e.g., ["09:00","18:00"] for daily rollups
  schedulingHorizonDays: Int // how far ahead to plan (e.g., 14)
}
Availability/Windows format (in windowsJSON):

json
Copy
Edit
{
  "weekdays": ["Mon","Tue","Wed","Thu"],
  "timeBlocks": [{"start":"18:00","end":"21:00"}],
  "excludedDates": ["2025-08-12"]
}
5) Scheduling Engine (MVP → V1)
5.1 Inputs
Goals (cadence, importance, min duration, windows).

Prefs (weekly capacity, quiet hours).

Current plan items & EventKit conflicts.

5.2 Priority Score (per contact)
ini
Copy
Edit
overdueDays = max(0, today - (lastCompletedAt + cadenceDays))
urgency     = overdueDays / cadenceDays         // normalized 0..∞
score       = (0.7 * urgency) + (0.3 * importanceNormalized)
5.3 Candidate Slot Generation
Generate slots within next schedulingHorizonDays.

Constrain by user windows & quiet hours.

Check conflicts against:

existing PlanItems (pending),

EventKit busy times (if using Calendar),

capacityPerWeek budget.

5.4 Placement Heuristic (MVP)
Sort contacts by score desc.

For each, pick the earliest valid slot that:

matches minDurationMin,

respects capacity and has no conflicts,

balances day-of-week distribution (light penalty if too many in one day).

Create PlanItem and mirror to EventKit (event or reminder).

Attach a local notification scheduled at startAt - leadTime.

5.5 Reschedule / Snooze Logic
Snooze: push by quick options (+1h, +1 day, +3 days) → recheck conflicts → update PlanItem & EventKit.

Reschedule: show next N candidate slots; on select, update and re-mirror.

5.6 V1 Optimizations
Weekly “packing” pass (simple greedy/knapsack) to respect capacityPerWeek.

Learn preferred times: track completion rate by weekday/time; apply bias in slot selection.

6) Notification UX
Categories & Actions

PLAN_ITEM_DUE: actions = MARK_DONE, SNOOZE, RESCHEDULE.

Deep link to a RescheduleSheet with 3–5 upcoming candidate slots.

Rolling scheduling: after firing, schedule the next reminder to avoid iOS “far future” batching.

7) Background Tasks
Register com.yourorg.socialplanner.refresh (BGAppRefreshTask) at launch.

On run:

Load Core Data, recompute priorities.

Prune past due/canceled items.

Fill horizon with new placements up to capacity.

Schedule notifications for new/updated items.

Submit next BGAppRefreshTaskRequest.

8) App Intents (Shortcuts/Siri)
PlanMyWeekIntent: run scheduler now, return summary.

SnoozeTonightIntent: snooze all today’s pending to tomorrow morning.

What’sNextIntent: show next 3 PlanItems.

9) UI Surfaces (SwiftUI)
Onboarding: permissions, select contacts, create first goals.

People: list of chosen contacts, edit goal (cadence, windows).

Planner: upcoming plan items by day; tap item → details (mark done, snooze, reschedule).

RescheduleSheet: candidate slot picker.

Settings: capacity/week, quiet hours, default lead time, data export/delete.

10) Project Structure
cpp
Copy
Edit
SocialPlanner/
  App/
    SocialPlannerApp.swift
    DIContainer.swift
  Features/
    Onboarding/
    People/
    Goals/
    Planner/
    Reschedule/
    Settings/
  Core/
    Models/
    Store/ (Core Data stack)
    Scheduling/
    Notifications/
    EventKit/
    Contacts/
    Background/
    Intents/
  SharedUI/
    Components/
    Theme/
  Tests/
    SchedulingTests/
    NotificationTests/
11) Key Implementation Notes
Contacts: store only identifier + displayName. Re-resolve on demand to avoid stale data.

EventKit: decide MVP target:

Option A (Recommended): Reminders (low friction; completion UX aligns with reminders).

Option B: Calendar events (clear visibility; may need a dedicated calendar).

Consistency: write-through to EventKit on create/update; listen to EKEventStoreChanged to detect external edits and replan if conflicts arise.

Error paths: if EventKit write fails (permissions/daylight-time mismatch), keep local PlanItem and notify user to grant permission or adjust slot.

Testing: unit tests for slot generation, conflict detection, and reschedule; UI tests for notification actions (use UNUserNotificationCenter test hooks).

12) Milestone Acceptance Criteria
M1 (Local MVP)

Create goals for 3 contacts, generate next 2 weeks of PlanItems, schedule notifications.

Snooze & reschedule from a notification action updates local + EventKit.

Background task repopulates horizon daily.

M2 (Reschedule UX + Learning)

Candidate slot picker suggests at least 3 valid options respecting capacity.

Completion rates per weekday/time tracked and influence future placements.

M3 (Shortcuts + Polish)

“Plan my week” Shortcut runs scheduler and returns a summary.

Widgets show “Today’s socials” with quick ‘Snooze next’.