You are my pair engineer. Build the foundation for an iOS 17+ SwiftUI app called “Social Planner” that plans social interactions I define (who + cadence + purpose), schedules them as Reminders or Calendar events, sends local notifications with actions, and supports one-tap snooze/reschedule.

**Non-negotiables**
- Local-first. Use Core Data for models: ContactRef, Goal, PlanItem, InteractionLog, Prefs (see fields below).
- Frameworks: Contacts, EventKit (pick Reminders for MVP), UserNotifications, BackgroundTasks, App Intents.
- Swift Concurrency (async/await), dependency injection via a simple `AppServices` container.
- Clean modular structure: `Core/` (Models, Store, Scheduling, Notifications, EventKit, Contacts, Background, Intents), `Features/` (Onboarding, People, Goals, Planner, Reschedule, Settings), `SharedUI/`, `Tests/`.

**Data Models (Core Data)**
- ContactRef { id: UUID, cnIdentifier: String, displayName: String, tags: [String], purpose: String? }
- Goal { id: UUID, contactId: UUID, cadenceDays: Int, importance: Int, minDurationMin: Int, windowsJSON: String, lastCompletedAt: Date? }
- PlanItem { id: UUID, contactId: UUID, startAt: Date, durationMin: Int, status: String, ekIdentifier: String?, origin: String, createdAt: Date }
- InteractionLog { id: UUID, contactId: UUID, when: Date, outcome: String, notes: String? }
- Prefs { id: UUID, capacityPerWeek: Int, quietHours: [Int], defaultNotificationTimes: [String], schedulingHorizonDays: Int }

**Features to scaffold**
1) Onboarding: request Contacts, Reminders, Notifications permissions with rationale screens.
2) People: pick contacts (`CNContactPickerViewController`), persist `cnIdentifier` + `displayName`, tag & set purpose.
3) Goals: set cadenceDays, importance, minDuration, availability windows JSON.
4) Planner: list upcoming PlanItems by day; detail view with actions (Done, Snooze, Reschedule).
5) RescheduleSheet: present next 3–5 candidate slots; confirm updates local + EventKit.
6) Notifications: register category `PLAN_ITEM_DUE` with actions `MARK_DONE`, `SNOOZE`, `RESCHEDULE`. Handle in `UNUserNotificationCenterDelegate`.
7) Background: register `BGAppRefreshTask` “com.yourorg.socialplanner.refresh”. On run → recompute plan, write PlanItems, sync notifications, request next refresh.
8) App Intents: `PlanMyWeekIntent`, `SnoozeTonightIntent`, `WhatsNextIntent`.

**Scheduling engine (MVP)**
- Priority score = 0.7 * urgency + 0.3 * importanceNormalized, where urgency is overdueDays / cadenceDays.
- Candidate slot generation within horizon; respect windows, quiet hours, capacity, and EventKit conflicts.
- Greedy placement (earliest valid slot). Mirror to Reminders; attach local notification.
- Snooze (+1h/+1d/+3d) & Reschedule (pick from generated candidates) must update both Core Data and EventKit.

**Deliverables to generate now**
- Xcode project files (SwiftUI lifecycle).
- Core Data model + lightweight persistence layer.
- Services: `ContactsService`, `EventKitService` (Reminders), `NotificationService`, `SchedulingService`, `BackgroundService`.
- SwiftUI screens for Onboarding, People, Goals, Planner, Reschedule, Settings (basic UI is fine).
- Unit test targets for SchedulingService (slot generation, conflict checks).

**Developer ergonomics**
- Add sample seed data in DEBUG builds.
- Add a “Run Scheduler Now” debug button in Planner screen.
- Log key actions with `os.Logger`.

Follow this spec exactly. Prefer small, composable types and explicit protocols for services. Avoid third-party dependencies. Generate clear TODOs where needed.
