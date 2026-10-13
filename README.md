Continue the Android app (Flutter UI + native Kotlin worker). This update has 6 parts: (1) hourly max limit, (2) "Reality check" life projection with years saved, (3) small in-app pop-ups at the top-left, (4) alarm screen at the limit, (5) precise timing, (6) monthly recap. Keep all existing features. If any of this already exists (limits, alarm mode, Time's-up screen, overlay permission flow), modify it instead of duplicating, and replace the daily-limit setting with the hourly max (if a daily limit exists, migrate to the default hourly max). Do NOT add accessibility services, app blocking, USE_FULL_SCREEN_INTENT, exact-alarm permissions, ads, billing, or new backend endpoints. All data stays on the device. Do not ask me questions; where a detail isn't specified, use the default given here and note it in the README.

DEFINITIONS (use these everywhere)
- Tracked apps: the apps the user picked.
- Hourly max G: minutes the user allows themselves on tracked apps combined in any rolling 60 minutes. Default 10, allowed 5 to 30 in steps of 5.
- Optional per-app hourly max A(app): a stricter per-app limit, same range.
- Window usage: foreground minutes of tracked apps in the last 60 minutes, computed from UsageStatsManager.queryEvents: pair MOVE_TO_FOREGROUND / MOVE_TO_BACKGROUND per package, clip to the window, and include any app still in the foreground up to now. Never use daily aggregate stats for this.
- Limit reached: window usage >= G, or window usage of an app >= its A(app).
- "Tracked app in foreground now": the last event for a tracked package is MOVE_TO_FOREGROUND with no later background event, AND PowerManager.isInteractive, AND KeyguardManager.isKeyguardLocked is false.
- Waking hours W: 24 minus the sleep duration from the user's sleep/wake times; default 16.
- Average daily hours D: the average daily foreground hours of tracked apps over the longest reliable window up to 30 days. Prefer the app's own stored daily totals once at least 7 valid days exist; otherwise use the system's weekly/daily usage buckets for up to the last 28 days (best effort). Require at least 3 days of data, otherwise show "Collecting your data" and no numbers. Store how many days it's based on, plus the per-app averages.
- Remaining years R = max(lifeExpectancy - age, 5). lifeExpectancy default 78. age = currentYear - birthYear.

1. ONBOARDING CHANGES
- New slide "What year were you born?" (year wheel, 1940 to currentYear-13), placed after the role slide. If age < 16: hide the life projection everywhere and use "{n} days a year" wording instead (n = D * 365 / 24); add a README note to set the Play target audience to 18+.
- After the usage-access slide succeeds, insert in this order: (a) "Reality check" (Part 2), (b) "Set your hourly max" (Part 2), (c) "How should we alert you?": Gentle (normal notifications only) or Active (small pop-ups + alarm screen).
- Active requires "Display over other apps". Show a prominent disclosure first ("When you pass your limit, Life can show a small pop-up or a screen over the app you're using. It only does this for apps you chose. You can switch it off any time."), then deep-link Settings.ACTION_MANAGE_OVERLAY_PERMISSION with Uri "package:<name>", detect with Settings.canDrawOverlays on resume plus 1s polling, auto-return to the app, and include a "Can't find it?" help sheet with the restricted-settings note for installs outside Play. If skipped or denied, use Gentle and show a fixable status chip on Home.
- Then the existing notifications, battery, and all-set slides.

2. REALITY CHECK (projection) AND YEARS SAVED
Implement these as pure functions with unit tests (Dart for the UI; Kotlin reads stored results and formats with NumberFormat; share the same test vectors in both):
- yearsAtPace = D / 24 * R
- cap_total = G * W / 60 (hours per day)
- cap_app = (A(app) or G) * W / 60
- limitedDaily = min( sum over tracked apps of min(avg_app, cap_app), cap_total )
- yearsWithLimit = limitedDaily / 24 * R
- yearsSaved = max(yearsAtPace - yearsWithLimit, 0)
- shareOfWaking = D / W as a percent
Test vector: age 25, lifeExpectancy 78 (R=53), D=5.0, W=16, G=10, one tracked app with avg 5.0 -> yearsAtPace 11.04, cap_total 2.667 h, yearsWithLimit 5.89, yearsSaved 5.15. Also test: D below the cap (saved = 0), a per-app override stricter than G, age < 16 hides the projection, and fewer than 3 days of data.
UI:
- "Reality check" slide and a collapsible card on Home: a big count-up number, "If you keep this pace, you'll spend about {yearsAtPace} years of your remaining life on {top 3 apps}", then "That's {shareOfWaking}% of your waking hours." Round to 1 decimal; if under 1 year show months. Show the top 3 apps as bars with hours per day and years each.
- A "How we calculate this" bottom sheet: the formula, D with the number of days it's based on, the lifespan assumption, and the line "A rough estimate, not a prediction."
- Styling: a dark warm gradient with red/orange accents from the logo palette, smooth animation, good contrast, respects reduce-motion.
- "Set your hourly max" slide: quick-pick chips for 5 and 10, plus a -5 / +5 stepper from 5 to 30 (default 10). Two live, animated numbers that update as the user changes it: "With a {G}-minute hourly max: {yearsWithLimit} years" and "You'd get back {yearsSaved} years." If yearsSaved is 0, show "You're already within this limit. Keep it up." with no scary text.
- Settings > Limits: each tracked app has its own stepper (default "Use general max") with a live "years saved" line per app and in total.
Guardrails: show the projection only in the onboarding slide, the Home card, the monthly recap, and the second or later alarm of the day (one short line). Never put it in notification text more than once a day. Never insulting wording.

3. SMALL POP-UPS TOP-LEFT (Active mode)
- Implement with WindowManager, TYPE_APPLICATION_OVERLAY, gravity TOP|START, a small card (about 280dp x 72dp), rounded corners, the app icon, one short line, and a thin progress bar. Offset below the status bar and display cutout using window insets. Use FLAG_NOT_FOCUSABLE and FLAG_NOT_TOUCH_MODAL so it never steals touches from the app underneath; only the card itself is touchable. Slide-in and fade-out animation, auto-dismiss after 6 seconds, swipe to dismiss, tap opens the Time's-up screen. Light haptic, no sound.
- Host it inside the CoroutineWorker: add the view on the main thread (withContext(Dispatchers.Main)), delay 6 seconds (this keeps the process alive), and remove it in a finally block before returning. Only if testing shows overlays vanishing early, fall back to a short-lived foreground service, and document this in the README.
- Popup types: (a) at 80% of the limit: neutral "{app}: {used} of {G} min this hour"; (b) at the limit: no popup (the alarm handles it); (c) overrun: after +5 minutes of tracked-app use since the last alert, a red-tinted "Over by {n} min. Next up: {task}" (use the goal if there's no plan).
- Show a popup only if a tracked app is in the foreground now, it's outside quiet hours, "Pause alerts today" is off, and fewer than 8 popups happened today. Otherwise fall back to the normal notification (heads-up channel) or send nothing. In Gentle mode or without overlay permission, always use the normal notification.

4. ALARM SCREEN AT THE LIMIT (Active mode)
- When the limit is reached, start a native Kotlin AlarmActivity with FLAG_ACTIVITY_NEW_TASK from the worker (allowed via the overlay permission) so it appears over the tracked app immediately. Native, not Flutter, so there's no engine cold start. Strings come from Android resources (all locales, plurals), colors from the app palette.
- Play one short alarm sound (3-5 seconds, once) on AudioAttributes.USAGE_ALARM at the user's own alarm volume (never force the volume up), plus vibration. Stop it when the screen closes. Setting "Alarm sound": on / vibrate only / off.
- Content: "{name}, you've used {used} min in the last hour. Your max is {G}." Then the "What you planned" card: the next undone task, its first step, and the estimate (or the goal if there's no plan). If it's the second or later alarm today, add one line: "At this pace: {yearsAtPace} years."
- Buttons: "Okay, I'll stop" (go home with ACTION_MAIN / CATEGORY_HOME, then finish), "Mark task done" (if a task is shown), and a small "Change my limit" link that opens the app. The Back button behaves like "Okay, I'll stop." No snooze and no "5 more minutes."
- Escalation after an alarm, by cumulative tracked usage since the last alert: +5 min -> overrun popup, +10 min -> alarm again. Max 3 alarms per day, at least 15 minutes apart.
- Skip the takeover and sound (normal notification only) when: a call or communication audio mode is active (AudioManager.mode), quiet hours, "Pause alerts today" is on, the tracked app isn't in the foreground now, or the daily cap is reached.
- Always also post the high-importance notification with a PendingIntent to the same screen as a backup. Log every attempt (launched, or skipped and why) in the debug log.

5. TIMING WITHOUT EXACT ALARMS
- On each periodic worker run, if a tracked app is in the foreground now, compute the minutes until the limit (G minus window usage, and the same for per-app limits, taking the smallest) and until the 80% mark. If the next event is within 20 minutes, enqueue a unique one-time WorkManager job at now + that time. When it runs, recompute from real events; if the limit isn't reached yet (old usage aged out of the window), enqueue again. Cancel if the app left the foreground. With Always-on mode enabled, evaluate every 30 seconds instead.
- Persist state per local date with the existing mark-first-then-notify logic; use deterministic notification IDs; make sure only one worker run executes at a time (unique work plus a lock).

6. MONTHLY RECAP (every 30 days after install)
- A notification or popup plus a recap screen: total hours on tracked apps in the last 30 days, the change vs the previous 30 days (if available), the top 3 apps, tasks completed, and the projection ("At this pace: {yearsAtPace} years" and "With your limit: {yearsWithLimit}"). If usage went down, lead with the improvement and hide the scary line. Add a share button that creates an image card with the app name, logo, and the numbers (no personal data). Needs at least 14 days of the app's own data, otherwise skip.

7. SETTINGS
- Mode (Gentle / Active), alarm sound, "Pause alerts today" (clears at local midnight), hourly max and per-app maxes, name and birth year editing, a "Reality check" toggle (default on), and a link to the system overlay settings.

8. REMOTE CONFIG (validate everything, keep the last good values, defaults in the app): hourly_max_default (10), hourly_max_min (5), hourly_max_step (5), hourly_max_cap (30), life_expectancy_years (78), waking_hours_default (16), popup_enabled (true), overrun_popup_minutes (5), overrun_alarm_minutes (10), max_alarms_per_day (3), max_popups_per_day (8), reality_check_enabled (true).

9. DEBUG
- Inputs for simulated window usage, simulated daily average, birth year, and a "tracked app in foreground" toggle. Buttons to fire: the 80% popup, an overrun popup, the alarm screen, and the monthly recap. Show the computed formulas with all their inputs, and a decision log (why something was or wasn't shown).

10. LOCALIZATION AND README
- All new strings via the existing localization setup (Flutter ARB plus Android resources, plurals, RTL, locale number formatting). Update the README with the formulas, defaults, flows, the overlay permission and Play review notes, and the under-16 behavior.

BUILD ORDER (stop after each step so I can test)
(1) Pure functions plus unit tests (window usage, projection, escalation logic) and the debug simulator; (2) onboarding slides (birth year, Reality check, hourly max with live savings) and Settings > Limits; (3) the foreground check and predicted-time job; (4) the pop-ups; (5) the alarm screen and overlay permission flow; (6) the monthly recap, Remote Config flags, localization, README.
