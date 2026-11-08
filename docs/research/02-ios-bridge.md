# Self-hosted assistant ↔ iPhone (no jailbreak): feasibility (as of 2026-11-08)

Research agent output. VERIFIED = source URL read; ASSUMED = inference.

**Version correction.** Current public iOS is **27.2** (released 2026-10-26); iOS 27.3 beta 1 on 2026-11-03; iOS 26.6.2 is last 26.x. VERIFIED: https://developer.apple.com/news/releases/ . iOS 27 materially changes #1 and #3 because Shortcuts gained a **notification trigger**.

## 1. Read notifications → server

**Shortcuts "notification received" trigger — EXISTS in iOS 27.** Automation: "When I receive a notification from [app]", optional title/subtitle/message filters, **Notification magic variable** with body text, time-sensitive flag, date. VERIFIED: https://www.macstories.net/stories/ios-and-ipados-27-review/13/ ; https://derekseaman.com/2026/08/ios-27-flips-the-script-home-assistant-can-now-control-your-apple-devices.html ("triggers can run in the background even if your device is locked") ; https://matthewcassinelli.com/wwdc26-sessions-whats-new-in-shortcuts/ .
Caveats: one automation names one source app (multiple triggers per shortcut allowed in iOS 27); no retroactive history; the shortcut POSTs via `Get Contents of URL` (ASSUMED-standard). Silent notifications do fire it (Deflector relies on this: https://www.macstories.net/ios/deflector-turns-ios-27-notification-triggers-into-a-shortcut-launcher/ VERIFIED). Behaviour under Focus / Notification Summary: UNKNOWN, test.

**Focus / Screen Time / Apple Watch:** no third-party read access to notification stream (ASSUMED by absence).
**Pushover/ntfy:** push TO phone only (VERIFIED by absence in docs).

**Mac-side (iPhone Mirroring) — viable backup path.** macOS Sequoia 15+ on Apple silicon, iOS 18+, same Apple Account w/ 2FA. "The Mac receives iPhone notifications … whether or not you're actively mirroring … as long as the iPhone is turned on." **Unavailable in the EU.** VERIFIED https://support.apple.com/en-us/120421 . Mirrored notifications land in macOS Notification Center (VERIFIED https://www.macstories.net/stories/macos-sequoia-the-macstories-review/2/ ).
DB: `~/Library/Group Containers/group.com.apple.usernoted/db2/db` (SQLite, binary-plist records, plaintext content); Group Containers are TCC-protected (VERIFIED https://9to5mac.com/2024/09/01/security-bite-apple-addresses-privacy-concerns-around-notification-center-database-in-macos-sequoia/ ). Needs Full Disk Access for the reader process: ASSUMED. Whether mirrored iPhone notifications persist in db2: ASSUMED yes, verify locally.

**Recommendation:** iOS 27 notification-trigger shortcut per app → POST to server (primary); Mac usernoted poller (secondary, Mac on, non-EU).

## 2. Telegram

**Bot API:** bots can't start conversations; user must message first. Bot sees all messages in its own private chats; group privacy mode limits visibility. Limits ~1 msg/s per chat, 20/min per group, ~30/s overall; files 20 MB down / 50 MB up. VERIFIED https://core.telegram.org/bots , https://core.telegram.org/bots/faq . Bot cannot see your personal chats.

**MTProto user client (Telethon):** acts as you. Telethon **v1.45.0, 2026-09-10**, v1 "maintenance mode" (VERIFIED https://pypi.org/project/Telethon/ ); GitHub archived 2026-02-21, moved to https://codeberg.org/Lonami/Telethon . README warns of bans. Telegram API ToS forbids "actions on behalf of the user without the user's knowledge and consent", requires own `api_id`, §1.5 restricts aggregating data for AI (VERIFIED https://core.telegram.org/api/terms ). Login: phone code + 2FA once; session file persists.

**Recommendation:** "assistant is a bot I chat with" → Bot API, zero ban risk. "assistant reads my Telegram and replies as me" → MTProto only; own api_id, real number, low volume; non-zero ToS risk.

## 3. Launch apps remotely

| Path | Works? | Evidence |
|---|---|---|
| Server push (any app) → iOS 27 notification-trigger automation → `Open App` | **Yes, hands-free, even when locked** | VERIFIED trigger (Seaman, Deflector); `Open App` foregrounding from background automation: ASSUMED |
| HA companion `data.shortcut` payload | Requires **tap**; opens Shortcuts app | VERIFIED https://companion.home-assistant.io/docs/integrations/siri-shortcuts/ |
| HA notification commands | No iOS launch command (`command_launch_app` Android-only) | VERIFIED https://companion.home-assistant.io/docs/notifications/notification-commands |
| Pushcut Automation Server | Needs **dedicated device with Pushcut in foreground** | VERIFIED (dev statement) https://redlib.belloworld.it/r/shortcuts/comments/eol77w/automate_shortcuts_without_user_interaction/fedhocb/?context=3 ; https://www.pushcut.io/ |
| Siri via speaker audio ("Hey Siri, open X") | Crude | ASSUMED |

**Recommendation:** push with fixed message body → iOS 27 notification automation → `Open App`. Include a Message body (title-only pushes don't match).

## 4. Close apps remotely

**No official API force-quits another app.** No `Close App` action; HA has no iOS close command. Hard limit.
Nearest: **`Lock Screen` action** (iOS 16.4+, VERIFIED https://www.macrumors.com/how-to/automatically-lock-iphone-app-opened ; Face ID may immediately re-unlock, https://discussions.apple.com/thread/255492553 ); **`Set Focus`** (block apps via allow-lists; iOS 27 adds Sleep Focus control); Screen Time App Limits **not scriptable** from Shortcuts; Guided Access manual only (VERIFIED https://support.apple.com/en-us/ht202612 ).

## 5. Expose phone state

**HA companion iOS sensors** (VERIFIED https://companion.home-assistant.io/docs/core/sensors ): `battery_level`, `battery_state` (+Low Power Mode), `ssid`, `bssid`, `connection_type`, `binary_sensor.focus` (boolean only), `geocoded_location`, `last_update_trigger`, `storage`, `activity`, `steps/distance/floors`, `pressure`, `sim_1/2`, `location_permission`, `app_version`, Apple Health (opt-in), `watch_battery_*`. **Not on iOS:** screen on/off, `frontmost_app` (macOS-only), `next_alarm`, DND name, volume, Bluetooth, last notification. Background updates opportunistic/stale (community reports).

**Shortcuts (timed/trigger automations → POST):** `Get Device Details` (name/model/OS/screen/volume/brightness/appearance, VERIFIED https://matthewcassinelli.com/actions/get-device-details/ ); `Get Battery Level` (ASSUMED); `Get Current Focus` (iOS 16.1+); `Find Alarm` (iOS 17+, misses Sleep Schedule alarm); `Find Calendar Events`/`Find Reminders` (ASSUMED standard). **Current foreground app and screen on/off: not readable** by any mechanism.

## 6. Push to phone from server

- **ntfy:** self-hosted server must set `upstream-base-url: "https://ntfy.sh"`; relay forwards only a poll_request (message id + SHA256 of topic URL), phone then fetches from your server. Without it "delivery can take hours". **iOS instant push cannot be fully self-hosted.** VERIFIED https://docs.ntfy.sh/config/ , https://docs.ntfy.sh/known-issues/ . Free.
- **Pushover:** $4.99 one-time per platform, 10,000 msgs/month. VERIFIED https://pushover.net/pricing .
- **HA companion:** remote push via project's push service (FCM→APNs), **500/device/day**; no Nabu Casa needed; **Local Push** over HA WebSocket on home Wi-Fi bypasses APNs, no limit. VERIFIED https://companion.home-assistant.io/docs/notifications/notification-details , https://companion.home-assistant.io/docs/notifications/notification-local .
- **Telegram bot:** simplest; can't initiate until you /start it.

**Recommendation:** Telegram bot for conversation; HA companion for automation pushes if HA is run; ntfy otherwise (accept ntfy.sh relay).

## 7. iOS 26/27 changes that matter

- iOS 26: `Use Model` Apple Intelligence action (VERIFIED https://9to5mac.com/2025/06/16/shortcuts-app-has-chatbot-like-apple-intelligence-powers-in-ios-26/ ).
- iOS 27: notification/screenshot/keyboard triggers; multiple triggers per shortcut; persistent Get/Set global values; `Use Model` with web retrieval (VERIFIED MacStories + Cassinelli). No new background restrictions found (ASSUMED).

## 8. Pricing

Pushcut: Free (3 notifications, 1 action); Pro $1.99/mo, $39.99 lifetime; Automation Server Extended $2.49/mo. Nabu Casa: $6.50/mo, not needed for push. Pushover: $4.99 one-time. ntfy, Telegram Bot API, Telethon: free.

## Feasibility table

| Requirement | Best path | Hard limit | Impossible |
|---|---|---|---|
| Read notifications → server | iOS 27 notification automation → POST; backup: Mac usernoted db2 (FDA, non-EU) | per-app automations; no history | third-party app reading all notifications on-device |
| Telegram | Bot API (chat with assistant); Telethon (act as you) | bot can't DM first; userbot ToS §1.4 risk | bot reading your personal chats |
| Launch apps | push with fixed body → notification automation → `Open App` | per-phrase setup; HA `shortcut` needs tap | Pushcut server on same phone |
| Close apps | `Lock Screen` / `Set Focus` after trigger | app stays alive; lock may re-unlock via Face ID | force-quit; scripting Screen Time/Guided Access |
| Phone state | HA companion sensors + timed Shortcuts | opportunistic updates; Focus name only via Shortcuts | current app, screen on/off |
| Push to phone | Telegram bot / HA local push / ntfy | HA 500/day via FCM; ntfy via ntfy.sh | fully self-hosted instant iOS push |

## Sources

Apple releases https://developer.apple.com/news/releases/ ; iPhone Mirroring https://support.apple.com/en-us/120421 ; MacStories iOS 27 https://www.macstories.net/stories/ios-and-ipados-27-review/13/ ; Deflector https://www.macstories.net/ios/deflector-turns-ios-27-notification-triggers-into-a-shortcut-launcher/ ; Cassinelli https://matthewcassinelli.com/wwdc26-sessions-whats-new-in-shortcuts/ , https://matthewcassinelli.com/actions/get-device-details/ ; Seaman https://www.derekseaman.com/2026/06/home-assistant-notifications-that-run-apple-shortcuts-yes-really.html , https://derekseaman.com/2026/08/ios-27-flips-the-script-home-assistant-can-now-control-your-apple-devices.html ; 9to5Mac usernoted https://9to5mac.com/2024/09/01/security-bite-apple-addresses-privacy-concerns-around-notification-center-database-in-macos-sequoia/ ; HA companion sensors/shortcuts/commands/details/local push (see inline); Nabu Casa https://www.nabucasa.com/pricing/ ; Telegram https://core.telegram.org/bots , https://core.telegram.org/bots/faq , https://core.telegram.org/api/terms ; Telethon https://pypi.org/project/Telethon/ , https://github.com/LonamiWebs/Telethon ; ntfy https://docs.ntfy.sh/config/ , https://docs.ntfy.sh/known-issues/ ; Pushover https://pushover.net/pricing ; Pushcut https://www.pushcut.io/ ; Lock Screen https://www.macrumors.com/how-to/automatically-lock-iphone-app-opened ; Guided Access https://support.apple.com/en-us/ht202612 .
